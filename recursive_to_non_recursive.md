We have shown how to convert a recursive hanoi code into a non-recursive code.
This ducoment tries to illustrate how to deal with a more complex case.


# 1. The recursive fg code

```c
int f(int n) {
  if (n <= 1) return 1;
  return f(n - 1) + g(n - 2);
}

int g(int n) {
  if (n <= 1) return 1;
  return f(n + 1) + g(n - 1);
}
```
we can see from the code above that f calls g and g calls f, so this makes the problem more challenging. Also there is a return value for the function which is different to the hanoi problem.

# 2. The basic idea

We can solve this problem based on the same idea as that for hanoi problem, i.e. state machine.
We treat each function/system call as a state, then we push the state into the stack to schedule the function calls.

## 2.1 How to define the state machine

Each state machine contains the following elements: 
+ input parameters, here it is n
+ function name, since we have two functions f and g, we need to know the relevant function
+ local variables that store the return values of f(n-1),g(n-2) etc
+ the return value of a function call
+ program counter: pc pointer that points to the current execution position or line number of the current function/program

## 2.2 Code modification

To better understand the transitions between states, I slightly modify the code. These modifications do not affect the code execution and results.  

```c
int f(int n) {
  int a, b, re;
  if (n <= 1) return 1;
  a=f(n-1);
  b=g(n-2);
  r=a+b;
  return r;
}

int g(int n) {
  int a, b, re;
  if (n <= 1) return 1;
  a=f(n+1);
  b=g(n-1);
  r=a+b;
  return r;
}

```
## 2.3 State definition

Here we can see that we use local variables a, b and r to represent the return values.

The states are stored in the following Frame struct.


```c
typedef struct
{
    int pc, n, a, b, r;
    char func_name;
    /* data */
}Frame;
```
## 2.4 Return values

It is important to know that stk[top-1] is the function call that call another function call corresponds to stk[top], 
i.e. stk[top-1] is the parent call of stk[top]. Therefore, we can assign the return value of stk[top] to a or b of stk[top-1].

We notice that if the function call is f, its value is always assigned to a of its parent call; if the function call is g, its return value is always assigned to b of its parent call (f or g).

# 3. Solution

We define the following three macros to manipulate the stack.
```c
#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
#define ret()     ({ top--; })
#define goto(loc) ({ f->pc = (loc) - 1; })
```

We can treat the f(n) function as a switch of 5 states.
1. if (n <= 1) return 1; Modify the return value 'r' 
2. call(pc=0, n-1, a=0, b=0, r=0, func_name='f'); Push a state into the stack
3. call(pc=0, n-2, a=0, b=0, r=0, func_name='g'); Push a state into the stack
4. r=a+b; Modify the return value 'r'
5. ret(); Pop a state and modify the value of the state variable 'a' on the top of the stack, refer to the "Return values" section for the design

We can treat the g(n) function as a switch of 5 states too.
1. if (n <= 1) return 1; Modify the return value 'r'  
2. call(pc=0, n+1, a=0, b=0, r=0, func_name='f'); Push a state into the stack
3. call(pc=0, n-1, a=0, b=0, r=0, func_name='g'); Push a state into the stack
4. r=a+b; Modify the return value 'r'
5. ret(); Pop a state and modify the value of the state  variable 'b' on the top of the stack, refer to the "Return values" section for the design

Based on the above ideas, I completed the corresponding non-recursive code, as follows:
```c
#include <stdio.h>
#include <assert.h>

/*#include "nr.c"*/
#include "fg_nr.c"

int main() {

	fg_nr('f',5);

}
```



```c
typedef struct
{
    int pc, n, a, b, r;
    char func_name;
    /* data */
}Frame;

#define call(...) ({ *(++top) = (Frame) { .pc = 0, __VA_ARGS__ }; })
#define ret()     ({ top--; })
#define goto(loc) ({ f->pc = (loc) - 1; })

void fg_nr(char func_name, int n){
    Frame stk[64], *top=stk-1;
    call(n,0,0,0,func_name);
    for(Frame *f; (f=top)>=stk; f->pc++){
        n=f->n; func_name=f->func_name;
        if (func_name=='f'){
            switch (f->pc)
            {
            case 0: if(n<=1) {f->r=1; goto(4);} break;
            case 1: call(n-1,0,0,0,'f'); break;
            case 2: call(n-2,0,0,0,'g'); break;
            case 3: f->r=f->a+f->b; printf("%c(%d) returns value f->r=%d\n", func_name,n,f->r); break;
            case 4: ret(); if(top>=stk) {top->a=f->r;} break;
            default:
                break;
            }
        }
        if (func_name=='g')
        {
            switch (f->pc)
            {
            case 0: if(n<=1) {f->r=1; goto(4);} break;
            case 1: call(n+1,0,0,0,'f'); break;
            case 2: call(n-1,0,0,0,'g'); break;
            case 3: f->r=f->a+f->b; printf("%c(%d) returns value f->r=%d\n", func_name,n,f->r); break;
            case 4: ret(); if(top>=stk) {top->b=f->r;} break;
            default:
                break;
            }
        }
        
    }
    printf("%c(%d) returns value=%d\n", func_name, n, stk->r);

}
```

This code runs successfully and prints out the return value 'r' of each function. Note the final return value is stored in the Frame at stk[0].

# 4. Conclusion

By using the concept of a state machine, we can convert recursive code into non-recursive code. This process mimics the compilation process.