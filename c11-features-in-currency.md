# C11 Overview

C11 is an informal name for ISO/IEC 9899:2011, the current standard for the C programming language. It includes multi-threading support , which enables us higher-level programming types to deal with concurrency now. Then, let's see how to use the C11 features to design the lock-free data-structures.

# C11 Atomic types

C11 defines a new \_Atomic type specifier. The atomic declaration syntax is like this:

\_Atomic \(type\_name\) or

\_Atomic type\_name

The type\_name can not be an array type, a function type, an atomic type, or a qualified type. For example, we could define an atomic integer like this:

```
_Atomic(int) a; //or
_Atomic int b; //both of them are the atomic integer
```

Please be sure to include the header "\#include&lt;stdatomic.h&gt;" when you are using the atomic type in your program.

Also, we could define a use-defined struct atomic types, which enable the struct to be manipulated atomically.

```
struct Node
{
    int data;
    struct Node *next;
};
_Atomic struct Node s; //s is also an atomic type
```

In addition to the ability to define user-defined atomic types, the \_Atomic keyword makes it possible to use objects of all atomic types in nearly all the same expressions and contexts as their direct \(non-atomic\) counterparts.

```
_Atomic int a = 3; // initialize like a normal int type
a = a + 2; //you can perform the atomic addition like this
```

Here is a simple example of using C11 atomic type:

test.c

```
#include<stdio.h>
#include<stdatomic.h>
#include<pthread.h>

_Atomic int acnt;
int cnt;
void *adding(void *input)
{
    for(int i=0; i<10000; i++)
    {
        acnt++;
        cnt++;
    }
    pthread_exit(NULL);
}
int main()
{
    pthread_t tid[10];
    for(int i=0; i<10; i++)
        pthread_create(&tid[i],NULL,adding,NULL);
    for(int i=0; i<10; i++)
        pthread_join(tid[i],NULL);

    printf("the value of acnt is %d\n", acnt);
    printf("the value of cnt is %d\n", cnt);
    return 0;
}
```

In order to compile C11 program, we need GCC4.9 or later. We should specify the parameter to tell the gcc compiler that we would like to compile the program by using C11 standard. So, the gcc command is as follows:

```
gcc -std=c11 -pthread -o test test.c
```

One of the possible outputs is:

```
the value of acnt is 100000 (correct)
the value of cnt is 89824 (wrong)
```

# Operations on atomic types

C11 provides some functions for us to change the content of atomic types like user-defined struct in an atomic way. Here I will introduce the CAS function, which we will use in lock-free data-struct designing.

```
_Bool atmoic_compare_exchange_weak(volatile A *object, C *expected, C desired);
```

Description: Atomically,  compares the value pointed to by object for equality with that in expected, and if true, replaces the value  pointed to by object with desired, and if false, updates the value in expected with the value pointed to by object. \(Here, **A **means atomic type, while the **C **means the non-atomic type corresponding to **A**, for example, **if A is \_Atomic int , then, C is int**\)

The **volatile** is a type qualifier, like const, and is a property of the type. It indicates that a value may change between different accesses \(eg: shared variable in multi-threading program\), even if it does not appear to be modified. With the volatile keyword, the compiler knows a variable needs to be reread from memory at each use, and it prevents the compiler from performing optimization on code. For more information, please refer to this link:

[http://www.geeksforgeeks.org/understanding-volatile-qualifier-c-set-1-introduction/](http://www.geeksforgeeks.org/understanding-volatile-qualifier-c-set-1-introduction/)

Returns: the result of the comparison, true or false.

Here, we will use the function to implement a simple lock.

```
#include<stdio.h>
#include<stdatomic.h>
#include<pthread.h>

int lock = 0;
int lock_count = 0;
int unlock_count = 0;

void *counting(void *input)
{
    int expected = 0;
    for(int i = 0; i< 100000; i++)
    {
        unlock_count++;
        while(!atomic_compare_exchange_weak(&lock,&expected,1)) //if the lock is 0(unlock), then set it to 1(lock).
            expected = 0; //if the CAS fails, the desired will be set to 1, so we need to change it to 0 again.
        lock_count++; 
        lock = 0;
    }
    pthread_exit(NULL);
}

int main()
{
    pthread_t tid[10];
    for(int i=0; i<10; i++)
    {
        pthread_create(&tid[i],NULL,counting,NULL);
    }
    for(int i=0; i<10; i++)
        pthread_join(tid[i],NULL);

    printf("the value of lock_count is %d\n",lock_count);
    printf("the value of unlock_count is %d\n",unlock_count);
}
```

one of the possible outputs is:

```
the value of lock_count is 1000000 (correct)
the value of unlock_count is 998578 (wrong)
```

For more information, please refer to [http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf "ISO/IEC 9899:2011 P273-286")

