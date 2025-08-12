# Linux System Programming

- Linux Internals
- OS Concepts Using Linux

# Process and Process Management

## Function Pointer
- Variable
- Function

**Syntax**

```c++
returntype (* variable name)(arguments);
```

**Example**
```c++
int (* abc)(int , int);
```
- abc is a pointer to a `function`, which takes **two integer** as arguments and **return int**.

### Program 

```c
#include <stdio.h>

int sum(int , int);
int sub(int , int);
int mul(int , int);
int dive(int , int);

int main()
{
    int i=10, j=20, k;

    int (*p)(int, int);

    p = sum;

    /*Calling Indirectly Using Function Pointer*/
    k = (*p)(i,j); /*k = p(i,j);*/
    printf("k = %d\n", k);

    /*Calling Directly */

    k = sum(i,j);
    printf("k = %d\n", k);
}
int sum(int m, int n)
{
    return m+n;
}
```
**Output**
- Calling Indirectly
```sh
k = 30
```
- Calling Directly
```sh
k = 30
```

## typedef with Function Pointer

**Syntax**

```c
typedef int (*FPTR) (int , int);
```

```c
FPTR p, q;
```

- p : ```int (*p) (int, int) ```
- q : ```int (*q) (int, int) ```


**Similar Example**

```c
char *abc (int, char);
```
- its typedef will be.
```c
typedef char * (*ptr) (int, char);
```

## Call Back Function

- Any function which is taking another function address as argument, such function are called as **Call Back Function**.
