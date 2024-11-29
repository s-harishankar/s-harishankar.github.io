---
title: "Knock knock! Race Condition! Who's there?"
date: 2024-11-26 00:00:00
categories: [Concurrency]
tags: [python,concurrency]
---

A few months ago, on a quiet Tuesday evening, the dreaded concurrency bug bit me. The ask was simple—"speed up" some 
python code that collected data from several network-connected devices in a long-running for loop. Seemed straightforward. **Easy**, I thought. 
**Why not throw in some threads to make this thing fly?**

The result? left I was two problems with.

And so began my crash course in "task juggling"—a lesson in humility I won't soon forget!

## "Thread"ing the needle

As hours ticked on, I quickly realized that concurrency in Python isn't just about slapping a few threads into your code and 
hoping for the best. Nope. This left me with not one, but two thorny problems I had never quite anticipated. Buckle up, 
this one's gonna be a bit of a rabbithole!

So, what are threads? Threads in Python are a way of running multiple operations concurrently within the same program. They are
particularly useful for tasks that are I/O-bound. But why? Let's break it down.

## What is an I/O bound task? 

When we say a task is I/O-bound, we're referring to operations that spend most of their time waiting for external resources or
input/output (I/O) operations to complete. These resources could include things like disk access, database 
queries, user input or as it was in my case—network requests. 

In simple terms, an I/O-bound task is limited by the speed at which it can read from or write to external systems, rather than 
being limited by CPU processing power. Imagine a program downloading several files from the internet. The task's speed depends 
more on network speed and server response time than on CPU performance. The CPU often sits idle, waiting for data to return, 
making it a classic I/O-bound operation.

While I/O-bound tasks are waiting on external systems, CPU-bound tasks are limited by the processing power of the CPU itself. 
Tasks that perform complex mathematical operations or run algorithms that require a lot of CPU cycles are good examples 
of CPU-bound tasks.

## Why are threads "particularly" useful for I/O bound tasks?

Before trying to answer this question, we first need to understand the Global Interpreter Lock (GIL). GIL, in 
simple words, is a lock that prevents multiple native threads from running Python bytecode simultaneously. It is
defined in the implementation notes as "just a boolean variable whose access is protected by a mutex". So 
even if your program uses multiple threads, GIL ensures that only one thread can execute at
a time, even on multi-core systems.

While this wouldn't be much of an issue to an I/O bound task which by its very nature, would spend most
of its time waiting on I/O operations rather than hammering the CPU, GIL would be a problem for CPU bound
tasks—tasks that require a lot of computation and can benefit from parallel execution across multiple
CPU cores. Below is a trivial CPU-bound function.
```python
def countdown(n):
    while n > 0:
        n -= 1
```
Running this sequentially without threads takes about 0.07s.
```commandline
timeit.timeit(lambda: countdown(1000000),number=1)
0.07554565300000604
```
If Python threads were to parallel process in its true-sense, then the below snippet of code should be able
to perform twice the amount of work in a similar amount of time by leveraging the multiple cores on my laptop.
```python
import threading

def countdown(n):
    while n > 0:
        n -= 1

def time_threaded_execution():
    thread1 = threading.Thread(target=countdown, args=(1000000,))
    thread2 = threading.Thread(target=countdown, args=(1000000,))
    
    thread1.start()
    thread2.start()
    
    thread1.join()
    thread2.join()
```
But alas, it takes nearly 1.5x as much time for twice as much work.
```commandline
timeit.timeit(lambda:time_threaded_execution(),number=1)
0.10705760900054884
```
An I/O bound task on the other hand, can simply "drop" the GIL while it is waiting on an I/O
operation to complete, allowing other threads to run.

## Why does the GIL exist though?

The GIL exists primarily for simplified memory management in Python which uses reference counting 
for memory management. What this means is an object created in Python, would have a reference counter
variable that keeps track of the number of references pointing to that object. When this count reaches
zero, the memory occupied by that object is released.

In a multi-threaded program, multiple threads share the same memory space, which is actually a strength providing 
performance benefits. This means that multiple threads would be able to 
freely modify variables that are shared between threads—this includes the reference counter. This necessitates the protection
of the reference counter variable from race conditions where two threads increase or decrease its value simultaneously. While this can be achieved
by adding locks to _all_ datastructures that are shared across threads, it comes with a significant overhead and also opens up 
possibilities for deadlocks.

The Global Interpreter Lock on the other hand, is a single lock on the interpreter itself which adds a rule that execution of any
Python bytecode requires acquiring the interpreter lock. While this prevents deadlocks and doesn't degrade performance, it effectively makes
any CPU-bound program single-threaded.

## Circling back to my story

Right! So it all lines up, doesn't it? We've got a large number of I/O bound tasks; threads are a good fit for I/O bound tasks; 
Inference: Let's rock some threads, what could go wrong?

Enter - Race Conditions.
```python
import time
import random

counter = 0

def wait_on_network_or_disk():
  time.sleep(random.random())

def worker():
    # What is my purpose?
    # You increment counter
    global counter

    wait_on_network_or_disk()
    counter += 1 # some glob var that is updated in each iteration
    wait_on_network_or_disk()

    print(f"The count is {counter}")
    print('-'*10)

print("Starting Up")
for i in range(10):
    worker()
print("Finishing up")
```
Shown above is a watered-down outline of what the legacy script was doing. It would perform a well-defined unit of I/O bound "work", several times in a sequence. This 
would produce a simple and straight-forward result - something very sane, easy to trace & easy to debug. 
```commandline
Starting Up
The count is 1
----------
The count is 2
----------
The count is 3
----------
The count is 4
----------
The count is 5
----------
The count is 6
----------
The count is 7
----------
The count is 8
----------
The count is 9
----------
The count is 10
----------
Finishing up
```
The only problem with this approach is that each unit of work executes sequentially, one after the other. And with each of these tasks
being I/O bound, the overall wait times would have a snowball effect. So if we were to perform each of these units of work, parallely instead
of sequentially using the threading module, that should have a net positive effect right? Wrong!
```python
import time
import random
import threading

counter = 0

def wait_on_network_or_disk():
  time.sleep(random.random())

def worker():
    # What is my purpose?
    # You increment counter
    global counter

    wait_on_network_or_disk()
    counter += 1
    wait_on_network_or_disk()

    print(f"The count is {counter}")
    print('-'*10)

print("Starting Up")
for i in range(10):
    threading.Thread(target=worker).start()
print("Finishing up")
```
The Python threading module allows you to run multiple operations concurrently in a single process by means of threads. A thread is a seperate
flow of execution in a program. Each thread runs independently, but shares the same memory space—allowing for fast communication 
between threads. This concurrent access to shared data can also lead to race conditions—like the one shown below.
```commandline
Starting Up
Finishing up
The count is 3
----------
The count is 6
----------
The count is 7
----------
The count is 10
----------
The count is 10
----------
The count is 10
----------
The count is 10
----------
The count is 10
----------
The count is 10
----------
The count is 10
----------
```
So like I said at the very beginning of this post, in my attempt to solve one problem, "left I was two problems with"

**(A deafening explosion of energy erupts as the screen fades to black)**

_Is GIL truly the enemy, or is there more to this than what meets the eye? What happens next? Find out what follows, on the next episode
of Dragon-Ball-Z!_
