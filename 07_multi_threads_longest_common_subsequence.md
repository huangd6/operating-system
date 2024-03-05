
This experiment requires us to parallelize the single-threaded Longest Common Subsequence (LCS) Dynamic Programming (DP), so that it can be accelerated by the parallel computing of multiprocessor.


# 1. Single-threaded Longest Common Subsequence (LCS) dynamic programming (DP)

We can find many examples of single-threaded Longest Common Subsequence (LCS) dynamic programming on the Internet. Here is one example.

`a`="ACADB" and `b`="CBDA" are the two given sequences. Find the longest common subsequence using the following steps:

+ Create a table of dimension n+1*m+1 where n and m are the lengths of `a` and `b` respectively. The first row and the first column are filled with zeros.
+ If the character correspoding to the current row and current column are matching, then fill the current cell by adding one to the diagonal element. Point an arrow to the diagonal cell.
+ Else take the maximum value from the previous column and previous row element for filling the current cell. Point an arrow to the cell with maximum value. If they are equal, point to any of them.
+ Repeat until the table is filled.
+ The value in the last row and the last column is the length of the longest common subsequence.

![DP computational graph](./multi_threads_longest%20common_subsequence/lcs1.png)

![DP computational graph](./multi_threads_longest%20common_subsequence/lcs.png)


The Longest Common Subsequence (LCS) dynamic programming algorithm can be viewed as the computation of each node on a directed acyclic graph. The value of node DP(i, j) represents the length of the longest common subsequence between the first i elements of sequence `a` and the first j elements of sequence `b`.

To get the value of DP(i, j), we need the values of DP(i-1, j), DP(i, j-1), and DP(i-1, j-1). Only if these three values are calculated correctly can we calculate DP(i,j) correctly.


Here is a single-threaded Dynamic Programming with a double loop:

```c
for (int i = 0; i < N; i++) {
  for (int j = 0; j < M; j++) {
    int skip_a = DP(i - 1, j);
    int skip_b = DP(i, j - 1);
    int take_both = DP(i - 1, j - 1) + (A[i] == B[j]);
    dp[i][j] = MAX3(skip_a, skip_b, take_both);
  }
}
```

# 2. Explore parallel computing opportunities

In the single-threaded Dynamic Programming above, the inner loop cannot be directly parallelized: when i is fixed, computing DP(i,j) requires the value of DP(i,j−1). 

So, in what order should it be calculated? The lecturer provides us an idea. Rotate the table 45 degrees. We find that we can calculate the values ​​on all nodes within n+m-1 steps (each red line represents a step), and the nodes in each step can be calculated concurrently, since their values do not depend on other nodes in the same step.

![Rotate DP table](./multi_threads_longest%20common_subsequence/lcs-rot.png)

In this picture, once the nodes "above" the red line are calculated correctly, the nodes on the red line can be calculated concurrently. When the sequences are long enough, each thread can be assigned enough work.

# 3. Implement parallel computing

If we consider each node in the rotated table as a calculation task and the edges as the dependencies of the node calculations, we get a `dependency graph` of the calculation. In parallel computing, `dependency graphs` help identify tasks that can be executed concurrently and those that must wait for others to complete.

The depth of our `dependency graph` is the number of steps of serial calculation (rounds of iterations). In each round, we can perform parallel calculations on nodes at the same depth.

We use the APIs `thread.h/thread-sync.h` provided in the framework code to complete parallel computing.

thread.h

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <stdatomic.h>
#include <assert.h>
#include <unistd.h>
#include <pthread.h>

#define NTHREAD 64
enum { T_FREE = 0, T_LIVE, T_DEAD, };
struct thread {
  int id, status;
  pthread_t thread;
  void (*entry)(int);
};

struct thread tpool[NTHREAD], *tptr = tpool;

void *wrapper(void *arg) {
  struct thread *thread = (struct thread *)arg;
  thread->entry(thread->id);
  return NULL;
}

void create(void *fn) {
  assert(tptr - tpool < NTHREAD);
  *tptr = (struct thread) {
    .id = tptr - tpool + 1,
    .status = T_LIVE,
    .entry = fn,
  };
  pthread_create(&(tptr->thread), NULL, wrapper, tptr);
  ++tptr;
}

void join() {
  for (int i = 0; i < NTHREAD; i++) {
    struct thread *t = &tpool[i];
    if (t->status == T_LIVE) {
      pthread_join(t->thread, NULL);
      t->status = T_DEAD;
    }
  }
}

__attribute__((destructor)) void cleanup() {
  join();
}
```
thread-sync.h

```c
#include <semaphore.h>
#include<pthread.h> 
#include <stdint.h>

// Spinlock
typedef int spinlock_t;
#define SPIN_INIT() 0

static inline int atomic_xchg(volatile int *addr, int newval) {
  int result;
  asm volatile ("lock xchg %0, %1":
    "+m"(*addr), "=a"(result) : "1"(newval) : "memory");
  return result;
}

void spin_lock(spinlock_t *lk) {
  while (1) {
    intptr_t value = atomic_xchg(lk, 1);
    if (value == 0) {
      break;
    }
  }
}
void spin_unlock(spinlock_t *lk) {
  atomic_xchg(lk, 0);
}

// Mutex
typedef pthread_mutex_t mutex_t;
#define MUTEX_INIT() PTHREAD_MUTEX_INITIALIZER
void mutex_lock(mutex_t *lk)   { pthread_mutex_lock(lk); }
void mutex_unlock(mutex_t *lk) { pthread_mutex_unlock(lk); }

// Conditional Variable
typedef pthread_cond_t cond_t;
#define COND_INIT() PTHREAD_COND_INITIALIZER
#define cond_wait pthread_cond_wait
#define cond_broadcast pthread_cond_broadcast
#define cond_signal pthread_cond_signal

// Semaphore
#define P sem_wait
#define V sem_post
#define SEM_INIT(sem, val) sem_init(sem, 0, val)

```
Our code 

```c
// M2：multi_threads longest common subsequence

#include<stdio.h>
#include<string.h>
#include "thread-sync.h"
#include "thread.h"

mutex_t lk = MUTEX_INIT();
cond_t cv = COND_INIT();

#define MaxN 10000
int n, m, T;
char A[MaxN+1], B[MaxN+1]; // Two sequences: A and B
int dp[MaxN][MaxN]; // store results of computational graph

/* define a struct stack to store the coordinates of nodes in the computational graph that need to caculated in current round, and a pointer to current node.*/
struct stack{
    int row_arr[MaxN];
    int col_arr[MaxN];
    int pointer;
}task_stack;

#define CAN_CACULATE_NODE task_stack.pointer>=0

// return the largest number from three numbers
#define MAX(x, y) (((x) > (y)) ? (x) : (y))
#define MAX3(x, y, z) MAX(MAX(x, y), z)

// Thread gets tasks from task_stack and calculates node value
void Tworker(){  
    while (1)
    {
        mutex_lock(&lk);
        if(CAN_CACULATE_NODE) {
            int row_current=task_stack.row_arr[task_stack.pointer];
            int col_current=task_stack.col_arr[task_stack.pointer];
            task_stack.pointer--;
            mutex_unlock(&lk);
            // caculate a node value
            dp[row_current][col_current]=MAX3(dp[row_current-1][col_current],dp[row_current][col_current-1],dp[row_current-1][col_current-1]+(A[row_current-1]==B[col_current-1]));
        }else{
            mutex_unlock(&lk);
            break;
        }     
    }     
}

int main(int argc, char *argv[]) {
    assert(argc==4);
    strcpy(A, argv[1]);
    strcpy(B, argv[2]);
    T = atoi(argv[3]); // T is the number of threads
    n = strlen(A);
    m = strlen(B);

    // initialize the values of computational graph nodes 
    for(int i=0; i<n+1; i++){
        dp[i][0]=0;
    }
    for(int j=0; j<m+1; j++){
        dp[0][j]=0;
    }
    // if T=1, use single-threaded Dynamic Programming (DP) with a double loop to caculate the result.
    if (T==1){
        for (int i = 1; i < n+1; i++) {
            for (int j = 1; j < m+1; j++) {
                int skip_a = dp[i - 1][j];
                int skip_b = dp[i][j - 1];
                int take_both = dp[i - 1][j - 1] + (A[i] == B[j]);
                dp[i][j] = MAX3(skip_a, skip_b, take_both);
            }
        }
        printf("%d",dp[n][m]);
        return 0;
    }

    // else, multi_threaded
    int max_len = (n > m) ? n : m;

    for (int r = 1; r < m + n; r++) {
        //store the coordinates of nodes (on the red line) that need to be caculated in current round in task_stack
        task_stack.pointer=-1;
        if (r < max_len) {
            int a = r;
            int b = 1;
            for (int i = 0; i < r; i++) {
                task_stack.pointer++;
                task_stack.row_arr[task_stack.pointer]=a;
                task_stack.col_arr[task_stack.pointer]=b;
                a--;
                b++;              
            }
        } else {
            int a = n;
            int b = r - n + 1;
            for (int i = 0; i < n + m - r; i++) {
                task_stack.pointer++;
                task_stack.row_arr[task_stack.pointer]=a;
                task_stack.col_arr[task_stack.pointer]=b;
                a--;
                b++;              
            }
        }                
        // create threads to get tasks from task_stack
        for(int i=0; i<T; i++){
            create(Tworker);
        }
        // wait for all threads in this round to complete execution
        join();
    }
    printf("%d",dp[n][m]);
    return 0;
}

```
Observe the running time of the following three sets of sequences when the number of threads T are 1 (single_threaded), 2, 3, 4, and 5 respectively.

```c
// Test the following three sets of data:

// time ./a.out ACADB CBDA 1

// time ./a.out ACDKSHFWISKLFMCSLCBFJBGHKKANALAFJOUQWRNFNMEOFLENCLF ASDLFGLKESKLHFLSKDFJSLFJSLNFJALFFLASMFNGOJHA 1

/*
time ./a.out RJZQDZPHIDTWUKCSFLZHZXLSMHVQMQEZYRMLRUBKJAKRXEYGQANHXZUDEJQRFUEZSJSNMMZKYWSEKEGKUIVBVBTYLOXJIVQOIFRDOQLZJCMVWWPPCCNHBQMLKYNCPGPOQGJNFOZNSBDHMBJYUQNDODGHHIIXKRFVTMQNWBBVQBEQAGDYIWDYEXOZORJXZYEDKPRKXMXCBPRZLOFBSSKXCWPOCPZJIRJFZGADNZKVHYQYDQXUZKTSHQRUCOWAOFNJJUETQGRRONUURMIFXZNKREHRKBPSSKXZIYKTTTYNSQTFCRXWLDQRPPCYLOVRLJYQRODKLKWVRFAOMWBUVZWHVYBKUOWALATMPSZLFXEBXKNBFRCPXXBNNVJLYPSNDVVKGSCHQMKCUAFDOJXLUXECQXXSBZEWB HYFTIQKQSSGQVKGKZQYQELLYAQQHYGWVOXEGEWWPQFPPKCKRUEWXFOLUZKCMNLMQCUAQSCCGWTTMXKEIPKQIFXKMQBLMVZYKJGBJUJLJGYGWQGJKVMMLJIMNCRXWPNCLPWFLBBDBNQSTHNUCIZJFGJZGVLSTGZOZWLYVJCYWJISGQOCNWSAGAYZKRNKRWLTHCIVUKTJZRSOKRXWHLOFFAYOEVGVXNLUJFWRPUYFQJTFPUJAXIJSWVYFRMWWXBUXKPOPHRBEABEZXJUYTXWGOZTXHKFXFRVWCCAZVKQMDPHWZWCCFSKZBJHRLUXXKSLZZDHOAQTVLDKWTGYGRNRNVTPMXXGEEXETZQJDZVEBCUFLDMSCGLBCGHDXQRIQURZKEZSFBXXZFSYTBQWUQUNFZUPFGPBBVFTFO 1
*/
```
We find that if the problem size is relatively small, the overhead of parallelization, such as thread creation and mutual exclusion syscall, might outweigh the benefits of parallel computation.


![output](./multi_threads_longest%20common_subsequence/output-1.png)

![output](./multi_threads_longest%20common_subsequence/output-2.png)

To solve this problem, we may need to optimize parallel algorithms to reduce overhead and utilize resources efficiently. We will explore this issue in later lectures.













