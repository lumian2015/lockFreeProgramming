# Atomic types

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
struct Node{
    int data;
    struct Node *next;
};
_Atomic struct Node s; //s is also an atomic type
```

In addition to the ability to define user-defined atomic types, the \_Atomic keyword makes it possible to use objects of all atomic types in nearly all the same expressions and contexts as their direct \(non-atomic\) counterparts.

```
_Atomic int a = 3; // initialize like a normal int type
a++; //perform the post-incrementing in an atomic way
atomic_fetch_add(&a, 1); //do the same job like a++, I will introduce later
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

    printf("the value of acnt is %d\n",acnt);
    printf("the value of cnt is %d\n", cnt);
}
```

In order to compile C11 program, we need at least version of GCC4.9, the good news is that gcc version in the virtual machine used in CSCI3150 is GCC5.4.0, so we don't need to update the gcc version. We should specify the parameter to tell the gcc compiler that we would like to compile the program by using C11 standard. So, the gcc command is follow:

```
gcc -std=c11 -pthread -o test test.c
```

One of the possible outputs is:

```
the value of acnt is 100000(correct)
the value of cnt is 89824(error)
```

# Operations on atomic types

C11 provides some function for us to change the content of atomic types like user-defined struct in an atomic way. Let's see some decription of them. \(In the following function description, the parameter A means atomic type, while the C means the non-atomic type corresponding to A, for example, if A is \_Atomic int , then, C is int\)

1.

```
void atomic_store(volatile A *object, C desired);
```

Description: Atomically replaces the value of the atomic variable pointed to by object with desired. The operation is atomic write operation. The **volatile** is a type qualifier, like const, and is a property of the type. It indicates that a value may change between different accesses \(eg: shared variable in multi-threading program\), even if it does not appear to be modified. With the volatile keyword, the compiler knows a variable needs to be reread from memory at each use, and it prevents the compiler from performing optimization on code.

eg:

```
#include<stdio.h>
#include<stdatomic.h>

struct Node{
    int data;
    struct Node *next;
};

int main()
{
    _Atomic struct Node obj = {3,NULL};
    struct Node desired = {5,NULL};
    atomic_store(&obj,desired); // the first parameter must be atomic type, but it is not necessary to be "volatile"
                                // it can be changed from non-volatile to volatile type automatically. 
    printf("Now the data in obj is %d\n",obj.data); 
}

// the output is: Now the data in obj is 5.
```

2.

```
C atomic_load(volatile A *object);
```

Description: Atomically returns the value pointed to by object.

3 compare\_exchange functions \(CAS function in C language format\)

```
_Bool atomic_compare_exchange_strong(volatile A *object, C *expected, C desired);
_Bool atmoic_compare_exchange_weak(volatile A *object, C *expected, C desired);
```

Description: Atomically,  compares the value pointed to by object for equality with that in expected, and if true, replaces the value  pointed to by object with desired, and if false, updates the value in expected with the value pointed to by object.

Returns: the result of the comparison,true or false.

Actually, the effect of atomic\_compare\_exchange\_strong is:

```
if(memcmp(object,expected,sizeof(*object))==0)
    memcpy(object,&desired, sizeof(*object));
else
    memcpy(expected, object, sizeof(*object));
```

A weak compare-and-exchange operation may fail spuriously. That is, even when the contents of memory referred to by expected and object are equal, it may return zero and store back to expected the same memory contents that were originally there. This spurious failure enables implementation of compare-and-exchange on a broader class of machines, e.g. load-locked store-conditional machines.

A consequence of spurious failure is that nearly all uses of weak compare-and-exchange will be in a loop.

```
exp = atomic_load(&cur);
do{
    des = function(exp);
}while(!atomic_compare_exchange_weak(&cur,&exp,des));
```

When a compare-and-exchange is in a loop, the weak version will yield better performance on some platforms. When a weak compare-and-exchange would require a loop and a strong one would not, the strong one is preferable.

4 The atomic\_fetch and modify generic functions

```
key                operator             computation
add                   +                   addition
sub                   -                  subtraction
or                    |              bitwise inclusive or
xor                   ^              bitwise exclusive or
and                   &                  bitwise and
```

```
C atomic_fetch_key(volatile A *object, M operand)
```

Description: Atomically replaces the value pointed to by object with the result of the computation applied to the value pointed to by object and the given operand.

Returns:  Atomically, the value pointed to by object before the effects.

```
_Atomic int obj = 3;
int returned = atomic_fetch_add(&obj,4); //the function return the value pointed to by object before addition
printf("the value of obj is %d\n", obj); //the value of obj is 7
printf("the value of returned is %d\n",returned); //the value of returned is 3
```

For more information, please refer to [http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf "ISO/IEC 9899:2011 P273-286")

