Races and Synchronization
======================================

```
# git clone https://github.com/fxlin/p2-concurrency
cd p2-concurrency/exp1/
```

To build the given code, see [here](cmake.md).

Objectives
-------------

We will tinker with a minimalist, multithreaded app and see the role of synchronization. 

### Student experience

*   primary: recognize critical sections and protect them with a variety of mechanisms.
*   primary: demonstrate race conditions and how to fix them 
*   primary: experience with performance instrumentation, measurement and analysis

This experiment focuses **less on programming**, and more **on performance measurement and analysis**.

We will focus on concurrency correctness in this experiment, and will consider scalability in the subsequent experiment. 

## Prerequisites

To do this experiment, you may need to study a few things:

* a more complete tutorial on [pthreads](https://hpc-tutorials.llnl.gov/posix/). Our benchmark app is written in the `pthread` API.

* [clock\_gettime(2)](http://man7.org/linux/man-pages/man2/clock_gettime.2.html) ... high resolution timers (for accurate performance data collection).

* [GCC atomic builtins](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html) ... functions to directly execute atomic read-modify-write operations.

## Benchmark (counter.c): a shared integer counter

Our benchmark program uses multiple threads to update a shared integer counter in parallel. The source project is built via CMake. See our short introduction on CMake [here](./cmake.md).

The program defines a global variable: 

```
long the_counter = 0;
```

The main function spawns *N* worker threads and then waits for them to join: 

```c
pthread_t threads[numThreads];

for (int i = 0; i < numThreads; i++) {
    if (pthread_create(&threads[i], NULL,
                       (void*) &thread_func, &iterations) < 0) {
        perror("thread_create:");
        exit(1);
    }
}

for (int i = 0; i < numThreads; i++) {
    if (pthread_join(threads[i], NULL) < 0) {
        perror("thread_join");
        exit(1);
    }
}
```

Each thread runs the following function to flip the shared counter repeatedly: 

```
void thread_func(int *iterations) {
	for (int i = 0; i < *iterations; i++)
		add(&the_counter, 1);

	for (int i = 0; i < *iterations; i++)
		add(&the_counter, -1);
}
```

The function `add` updates the global counter value by adding a value to it: 


```
void add(long long *pointer, long long value) {
    long long sum = *pointer + value;
    *pointer = sum;
} 
```

Note that `value` is signed, meaning that `add()` can either increment or decrement the counter. 

### Observation

After all the worker threads are joined, what would be the final value for `the_counter`? Should it be zero, since the counter is incremented & decremented the same number of times? 

The problem is race condition: multiple worker threads update the counter without mutual exclusion. How bad could it be? 

### Concurrent update w/o locking

We may see `the_counter==0` after workers are joined -- with a few threads and fewer iterations
```
$./counter-nolock --iterations=100 --threads=10
test=add-none threadNum=10 iterations=100 numOperation=2000 runTime(ns)=634688 avgTime(ns)=317 count=0
```
However, as the numbers of threads/iterations increase moderately, we will soon observe a non-zero `the_counter` -- there's an error in our program! 

```
$./counter-nolock --iterations=10000 --threads=10
test=add-none threadNum=10 iterations=10000 numOperation=200000 runTime(ns)=5280826 avgTime(ns)=26 count=-5275
```

> Why does a smaller number of iterations rarely fail?  Why does it take more iterations before errors are seen?  

### Concurrent update with locking

Of course, one way to fix the race condition is to wrap `add()` with locking: 

```
// create a global mutex
pthread_mutex_t mutex;
pthread_mutex_init(&mutex, NULL)

// in each worker thread
pthread_mutex_lock(&mutex);
add(&the_counter, val);
pthread_mutex_unlock(&mutex);
```

With synchronized updates, we will always see the right count value: 

```
$ ./counter --iterations=100000 --threads=10 --sync=m
test=add-m threadNum=10 iterations=100000 numOperation=2000000 runTime(ns)=147502381 avgTime(ns)=73 count=0
```

## Conclusion

We have seen the necessity of synchronization and understood the basic structure of the benchmark. Play with the given benchmark and reproduce the results above. After that, proceed to the assignments.

