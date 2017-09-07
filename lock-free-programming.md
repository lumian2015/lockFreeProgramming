# What is Lock-free programming

Lock-free programming is a technique that allow concurrent updates of shared data structures without the need to perform costly synchronization between threads. This method ensures that no threads block for arbitrarily long times, and it guarantee progress for some threads when there are multiple threads.

Lock-free algorithms are carefully designed data-structures and functions to allow for multiple threads to attempt to make progress independently of one-another. This means that you do not try to acquire a lock before performing your critical region. Instead, you independently update a local copy of a portion of the data-structure and then apply it atomically to the shared structure with a CAS\(Compare-And-Swap\).

Lock-free programming has the following advantages:

* Can be used in places where locks must be avoided, such as interrupt handlers
* Avoid the troubles with blocking, such as deadlock and priority inversion
* Improve preformance on a multi-core processor

Now, let's begin to design the lock-free data-structure.

