# ABA problem

The use of CAS has one problem to deal with.  It is called the ABA problem.  The problem arises from the **C **of **C**AS, where the **c**omparison is value based.  That is, as long as the value involved in the comparison is the same, the swap can proceed.  However, there are still occasions that fool the CAS solution we presented.  Let's see an example where three threads concurrently access the lock-free stack we presented:![](/assets/aba_example9.jpg)\(Ps: In our case, it is not necessary for the newly allocated node's content is the same as the original node, they just need to use the same memory block\)

# ABA Problem Solutions

The root solution to ABA problem is that we should defer reclamation while other threads are holding the node or we should have a way to identify the difference between the old node and the new node.

There are several solutions to the ABA problem, for the details, you can see this link. [https://en.wikipedia.org/wiki/ABA\_problem](https://en.wikipedia.org/wiki/ABA_problem "ABA problem")

Here, I would introduce a common solution for you.

A common workaround is to add extra "tag" or "stamp" word to the quantity being considered. For example, an algorithm using compare and swap on a pointer might use a "tag" to indicate how many times the pointer has been successfully modified. Because of this, the next compare-and-swap will fail, even if the addresses are the same, because the tag word will not match.  This is sometimes called ABAʹ since the second A is made slightly different from the first.

Here , we solve the ABA problem by using the above solution. We reconstruct the stack by adding a "tag":

```
typedef struct _lfstack_t
{
    int tag;
    Node *head;
}
```

Every time when we use the pointer, we will increase the "tag" by 1.

```
void lfstack_push(_Atomic lfstack_t *lfstack, int value)
{
    lfstack_t next, orig = atomic_load(lfstack);
    Node *node = malloc(sizeof(Node));
    node->data = value;
    do{
        node->next = orig.head;
        next.head = node;
        next.tag = orig.tag+1; //increase the "tag"
    }while(!atomic_compare_exchange_weak(lfstack,&orig,next));
}

int lfstack_pop(_Atomic lfstack_t *lfstack)
{
    lfstack_t next, orig = atomic_load(lfstack);
    do{
       if(orig.head == NULL)
       {
            return -1;
       }
       next.head = orig.head->next;
       next.tag = orig.tag+1; //increase the "tag"
    }while(!atomic_compare_exchange_weak(lfstack,&orig,next));
    printf("poping value %d\n",orig.head->data);
    free(orig.head);
    return 0;
}
```

Don't forget to change the initialization of stack.

```
_Atomic lfstack_t top = {0,NULL};
```

Now, when we run the program again, it will not print any error information. Here is a C library called **liblfds**, [https://www.liblfds.org/](https://www.liblfds.org/), a lock-free data structure library written in C. If you don't like to write the lock-free data structure by yourself, you could use this C library.

Let's compare the lock-free stack with the stack implement by using mutex. Here is the code of the lock-based stack.

lock\_based\_stack.c

```
#include<stdio.h>
#include<pthread.h>
#include<stdlib.h>

typedef struct Node{
    int data;
    struct Node *next;
}Node;

Node *head = NULL;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
void *push(void *input)
{
    for(int i=0; i<100000; i++)
    {
        Node *node = malloc(sizeof(Node));
        node->data = i;
        pthread_mutex_lock(&mutex);
        node->next = head;
        head = node;
        //printf("pushing value %d\n",node->data);
        pthread_mutex_unlock(&mutex);
    }
    pthread_exit(NULL);
}

void *pop(void *input)
{
    for(int i=0; i<100000;)
    {
        pthread_mutex_lock(&mutex);
        if(head ==NULL)
        {
            //printf("the stack is empty\n");
            pthread_mutex_unlock(&mutex);
        }
        else{
            Node *pop_node = head;
            head = head->next;
            //printf("poping value %d\n",pop_node->data);
            free(pop_node);
            i++;
            pthread_mutex_unlock(&mutex);
        }
    }
    pthread_exit(NULL);
}

int main()
{
    pthread_t tid[200];
    for(int i=0; i<100; i++)
        pthread_create(&tid[i],NULL,push,NULL);
    for(int i=100; i<200; i++)
        pthread_create(&tid[i],NULL,pop,NULL);
    for(int i=0; i<200; i++)
        pthread_join(tid[i],NULL);
    return 0;
}
```

I compare the lock-free stack and lock-based stack by measuring their running time. In order to compare their efficiency, I comment the "printf" statement in both of the programs. I use the built-in command `time` in linux to measure their running time.

Here are the average results by running each of them 10 times.

lock-free stack

```
real 0m2.133s
user 0m6.140s
sys  0m0.946s
```

lock-based stack

```
real 0m2.730s
user 0m3.594s
sys  0m4.907s
```

From the result, we could see that lock-free stack is more efficient than lock-based stack while doing the same thing.  Actually, if you pay attention to the implementation above, a lock-free stack is logically using a spin lock as well.  Moreover, recall that pthread library's mutex is also implemented using CAS.  So, why one is faster than the other?

The answer is, the one that uses CAS to implement is a **direct low-level implementation **that uses CAS instructions, whereas the one that uses pThread is implemented **using the pThread's library**.  Think about the use of "ls -l \| less" vs the use of a GUI file manager to browse a directory.  The former is efficient but primitives, more suitable for advanced users.  The latter is easier and has more overhead \(e.g., rendering the GUI\).  Similarly, the pThread-based stack is easier to implement \(using mutex\), higher-level, but incurs syscall overhead within the mutex implementation \(e.g., wait\(\) and context switch\).  In contrast, the CAS one we presented above is lower-level \(almost direct use of CAS instructions\), direct \(so that involves no sys call\), but it is much more cumbersome to implement \(e.g., take care of the ABA problem yourself\).

Therefore, lock-free implementation in essence is for advanced developers, especially, system developers, who aim for highest efficiency.  Remember, the lock-free implementation above essentially implements a spin lock **implicitly **and it is "lock-free" just because it is \(high-level\) **mutex lock **free only.

