---
layout: post
title:  "User-Level Threads and Backend Services"
date:   2026-05-22 21:00:00 -0400
categories: systems
---

# Introduction
A user-level thread library implements threading without relying on the
kernel to schedule each logical thread. The project I built exposed thread
creation, `yield`, `join`, mutexes, and condition variables.

The interesting part was not the public API. The interesting part was
tracking where each thread was allowed to be. A thread could be running,
ready to run, blocked on a mutex, blocked on a condition variable, waiting
for another thread to finish, or finished.

That is useful to think about when writing backend services. Backend systems
also move work between states. A request may be running, waiting on I/O,
queued for retry, leased by a worker, or finished. If those states are not
clear, concurrency bugs become much harder to reason about.

This assumes some familiarity with C++, threads, and C# `async`/`await`.

## Thread State
The thread library needed a few core data structures:

* thread execution contexts
* a ready queue
* queues for blocked threads
* mutex ownership
* condition variable wait queues

The public API could be simple:

```cpp
thread t(worker, arg);
t.join();

mutex m;
cv condition;

m.lock();
while (!ready) {
    condition.wait(m);
}
m.unlock();
```

Internally, the library had to preserve a few invariants. A thread should
not be in two queues at once. A locked mutex should have one owner. A blocked
thread should have a specific reason to wake up. A completed thread should
wake any threads joining it.

The scheduler loop is conceptually simple:

```cpp
while (!ready_queue.empty()) {
    thread_state* next = ready_queue.pop();
    run(next);
}
```

The more important question is what happens to the thread that just stopped
running. Did it call `yield`? Did it block? Did it finish? That state change
is the hard part.

## Blocking
When a thread calls `join`, the current thread should stop running until the
target thread finishes.

Conceptually:

```cpp
if (!target.done()) {
    target.join_waiters.push(current);
    schedule_next_thread();
}
```

This is not very different from backend work waiting on a remote dependency,
queue lease, callback, or timeout. In both cases, the work is not gone. It is
blocked, and the system should know what can wake it.

A vague state like `pending` is often not enough. It is better to know if the
work is queued, leased, waiting on an event, retrying, cancelled, or done.

## Condition Variables
Condition variables are easy to misuse if they are treated as simple
notifications. The condition itself is the important part.

The usual pattern is:

```cpp
m.lock();
while (!predicate()) {
    condition.wait(m);
}
use_shared_state();
m.unlock();
```

The `while` loop is necessary because a wakeup does not prove that the
predicate is true. A different thread may have consumed the state first, or
the wakeup may not correspond to the condition this thread needs.

This is similar to event-driven backend code. A webhook, queue message, or
database notification should usually cause the service to inspect durable
state. The notification itself should not usually be treated as the source
of truth.

## Async/Await
C# `async`/`await` is a useful comparison. It is not implemented the same way
as a C++ user-level thread library, but an `async` method is also split into
states.

For example:

```csharp
public async Task<Order> GetOrderAsync(string id)
{
    var order = await orderRepository.GetAsync(id);
    return await priceService.ApplyPricesAsync(order);
}
```

The compiler rewrites this method into a state machine. When
`orderRepository.GetAsync(id)` returns an incomplete task, the method does
not need to keep the current thread occupied. The remaining work is stored as
a continuation and resumes when the task completes.

In the thread library, I explicitly moved a thread from a ready queue to a
blocked queue. In .NET, the compiler, `Task`, and scheduler handle most of
that movement.

This is why `.Result` and `.Wait()` should be used carefully in backend code.
They take an operation that could yield (allow the running thread to be continued later vs waiting) and turn it into a blocked thread.
Cancellation tokens and timeouts are also important because the continuation
may run much later than the original call.

## Critical Sections
The scheduler had internal state that could not be modified halfway and then
exposed to arbitrary user code. A context switch at the wrong time would
leave the library in an invalid state.

The internal pattern was:

1. enter a protected scheduler section
2. update the queues or ownership state
3. restore the scheduler invariants
4. switch to another thread

Backend services have similar sections. Claiming a job, updating an
idempotency record, releasing a lease, and transitioning workflow state
should be small and obvious. The rest of the code can happen after the
state transition is complete.

## Testing
The hardest tests were ordering tests. For example:

* a thread yields while another thread is joining it
* a mutex unlock wakes a waiter
* a condition variable signal happens when no thread is waiting
* multiple threads wait on the same condition

Backend services need the same style of testing around state transitions:

* duplicate requests
* retries after partial failure
* events delivered out of order
* cancellation racing completion
* two workers claiming the same work

These tests are useful because they expose unclear ownership. If the test
cannot explain who owns the work, the implementation probably cannot either.

## Conclusion
Building a thread library made scheduling and blocking concrete. A thread was
running, ready, blocked, or finished because the implementation put it there.

That is the part that applies to backend services. Make the state of work
explicit, know what can wake it, and be careful about transitions that change
ownership.

Please feel free to email me at [{{ site.email }}](mailto:{{ site.email }})
with any questions or concerns.
