

This document tries to illustrate how the non-recursive Hanoi code works by tracing the stacks. The idea of this code is to mimic the process of functional calls in operation system. 

## What is state machine
Before we start, it will be nice to gain a basic understanding of state machines. For a function, we could see it as a state machine. The states are the input parameters, local variables, that is relevant to this function. If we know all the values of these variables, we know the state of this function and the function should return exactly the same value. This means if we can store the variables of a function, and do some other functions calls, then resume the execution of this function by loading the stored variables (states).

## Recursive Hanoi code
You can search Hanoi problem from wikipedia, here I am not going to introduce it. Here is the classic recursive code for Hanoi problem

```c
1 void hanoi(int n, char from, char to, char via) {
2  if (n == 1) {
3    printf("%c -> %c\n", from, to);
4  } else {
5    hanoi(n - 1, from, via, to);
6    hanoi(1,     from, to,  via);
7    hanoi(n - 1, via,  to,  from);
8  }
9}
```

## Can we turn the recursive code into non-recursive code?

According to the lecturer of operation system, there exist a correspondence between a normal c code and another if-goto grammar. This correspondence is based on some complex programming theory. 

Here we use the following non-recursive code given by the lecturer to illustrate the idea. For the recursive Hanoi code, its states are the input parameters n, from, to, via and its current position for function/system call. Here we use PC (program counter) to indicate the position of the function/system call. For example, call(n=1, from=C, to=B, via=A, pc=0) means the input  n=1, from=C, to=B, via=A and program execute to line 3. call(n=1, from=C, to=B, via=A, pc=1) means the input n=1, from=C, to=B, via=A, and program execute to line 5. Both calls are the same function call, but execute to different lines.




```c
1. typedef struct {
2.   int pc, n;
3.   char from, to, via;
4. } Frame;
5.
6. #define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
7. #define ret()     ({ top--; })
8. #define goto(loc) ({ f->pc = (loc) - 1; })
9.
10. void hanoi(int n, char from, char to, char via) {
11.   Frame stk[64], *top = stk - 1;
12.   call(n, from, to, via);
13.   for (Frame *f; (f = top) >= stk; f->pc++) {
14.     n = f->n; from = f->from; to = f->to; via = f->via;  # execute to this line
15.     switch (f->pc) {
16.       case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17.       case 1: call(n - 1, from, via, to);   break;
18.       case 2: call(    1, from, to,  via);  break;
19.       case 3: call(n - 1, via,  to,  from); break;
20.       case 4: ret();                        break;
21.       default: assert(0);
22.     }
23.   }
24. }
```

 
The key points of the code
1. the struct Frame is used to store all the states of a functional call, especially pc means the program counter of a function call
2. the macro call(…) pushes the states into the stack, note the top variable is always pointing to the top of the stack, if top<stk, the stack is empty
3. the macro ret() pop out one function call from the stack
4. the macro goto() moves the pc of the current state. 
5. in the for loop, f-pc is first incremented, then top is assigned to f, after that f is compared with stk

Here we track the change of the stack for the following function call

```c
hanoi(2, 'A', 'B', 'C');
```

step 0.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```


| pointer | content |
|----------|----------|
|     |     |
|     |     |
| stk, f, top    |  call(n=2, from=A, to=B, via=C, pc=0)  |


Step 0.2, we execute to line 16. since pc is 0 and n !=1, nothing happened to the stack. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|     |     |
|Stk, f, top     |  call(n=2, from=A, to=B, via=C, pc=0)   |

Step 1.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|     |     |
|Stk, f, top    | call(n=2, from=A, to=B, via=C, pc=1)    |






Step 1.2, we execute to line 17. 

```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```
| pointer | content |
|----------|----------|
|     |     |
|top     |  call(n=1, from=A, to=C, via=B, pc=0)  |
|Stk, f    | call(n=2, from=A, to=B, via=C, pc=1)    |


Step 2.1, we execute to line 14. 

```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```


| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=A, to=C, via=B, pc=0)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=2)     |


Note that the pc of f increment by 1, and then f is assign to top.

Step 2.2, we execute to line 16. 

```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```


| pointer | content |
|----------|----------|
|     |     |
|Top, f     | call(n=1, from=A, to=C, via=B, pc=3)    |
|Stk     |call(n=2, from=A, to=B, via=C, pc=2)     |

PRINT A -> C, since n==1, execute goto()

Step 3.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```


| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=A, to=C, via=B, pc=4)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=2)     |

Step 3.2, we execute to line 20.
```c 
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
| f     |call(n=1, from=A, to=C, via=B, pc=4)     |
|Stk, Top     |call(n=2, from=A, to=B, via=C, pc=2)     |

Step 4.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|     |call(n=1, from=A, to=C, via=B, pc=5)     |
|Stk, Top， f     |call(n=2, from=A, to=B, via=C, pc=2)     |

Step 4.2, we execute to line 18. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|top      |call(n=1, from=A, to=B, via=C, pc=0)     |
|Stk, f     |call(n=2, from=A, to=B, via=C, pc=2)    |


Note we push a new function call call(n=1, from=A, to=B, via=C, pc=0) to the stack, and top moves up.


Step 5.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=A, to=B, via=C, pc=0)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=3)     |


Step 5.2, we execute to line 16. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=A, to=B, via=C, pc=3)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=3)     |

PRINT  A -> B,  since n==1, execute goto()


Step 6.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=A, to=B, via=C, pc=4)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=3)     |

Step 6.2, we execute to line 20. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|f     |call(n=1, from=A, to=B, via=C, pc=4)     |
|Stk, top     |call(n=2, from=A, to=B, via=C, pc=3)     |


Execute ret(), pop one frame out of stack.


Step 7.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```
| pointer | content |
|----------|----------|
|     |     |
|     |call(n=1, from=A, to=B, via=C, pc=5)     |
|Stk, top, f     |call(n=2, from=A, to=B, via=C, pc=3)     |

Step 7.2, we execute to line 19. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|top     |call(n=1, from=C, to=B, via=A, pc=0)     |
|Stk, f     |call(n=2, from=A, to=B, via=C, pc=3)     |

Push the third function call to the stack.

Step 8.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=C, to=B, via=A, pc=0)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=4)     |


Step 8.2, we execute to line 16. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=C, to=B, via=A, pc=3)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=4)     |

PRINT  C -> B,  execute goto()


Step 9.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|Top, f     |call(n=1, from=C, to=B, via=A, pc=4)     |
|Stk     |call(n=2, from=A, to=B, via=C, pc=4)     |


Step 9.2, we execute to line 20. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|f     |call(n=1, from=C, to=B, via=A, pc=4)     |
|Stk, top     |call(n=2, from=A, to=B, via=C, pc=4)     |

Step 10.1, we execute to line 14. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |     |
|     |call(n=1, from=C, to=B, via=A, pc=5)     |
|Stk, top, f     |call(n=2, from=A, to=B, via=C, pc=4)     |

Step 10.2, we execute to line 20. 
```c
 1 typedef struct {
 2  int pc, n;
 3  char from, to, via;
 4} Frame;
 5
 6#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
 7#define ret()     ({ top--; })
 8#define goto(loc) ({ f->pc = (loc) - 1; })
 9
10void hanoi(int n, char from, char to, char via) {
11  Frame stk[64], *top = stk - 1;
12  call(n, from, to, via);
13  for (Frame *f; (f = top) >= stk; f->pc++) {
14    n = f->n; from = f->from; to = f->to; via = f->via;
15    switch (f->pc) {
16      case 0: if (n == 1) { printf("%c -> %c\n", from, to); goto(4); } break;
17      case 1: call(n - 1, from, via, to);   break;
18      case 2: call(    1, from, to,  via);  break;
19      case 3: call(n - 1, via,  to,  from); break;
20      case 4: ret();                        break;
21      default: assert(0);
22    }
23  }
24}
```

| pointer | content |
|----------|----------|
|     |call(n=1, from=C, to=B, via=A, pc=5)     |
|Stk,  f     |call(n=2, from=A, to=B, via=C, pc=4)     |
|top     |     |

Since top < stk, the for loop ends.