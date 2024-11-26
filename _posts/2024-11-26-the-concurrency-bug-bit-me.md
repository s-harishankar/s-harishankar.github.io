---
title: "Knock knock! Race Condition! Who's there?"
date: 2024-11-26 00:00:00
Categories: [concurrency]
tags: [python,concurrency]
---

A few months ago, on a quiet Tuesday evening, the dreaded concurrency bug bit me. The ask was simple—"speed up" some 
python code that collected data from several network-connected devices in a long-running for loop—seemed straightforward. **Easy**, I thought. 
**Why not throw in some threads to make this thing fly?**

The result? left I was two problems with.

And so began my crash course in "task juggling"—a lesson in humility I won't soon forget!

## "Thread"ing the needle

As hours ticked on, I quickly realized that concurrency in Python isn't just about slapping a few threads into your code and 
hoping for the best. Nope. This left me with not one, but two thorny problems I had never quite anticipated. Buckle up, 
this one's gonna be a bit of a rabbithole!

So, what are threads? Threads in Python are a way of running multiple operations concurrently within the same program. They are
particularly useful for tasks that are I/O-bound. But why? Let's break it down.

##### What is an I/O bound task, you may ask? 

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

##### Alright! Now why are threads "particularly" useful for I/O bound tasks? And why not for CPU bound ones?

To answer this, we first need to understand Global Interpreter Lock (GIL).






