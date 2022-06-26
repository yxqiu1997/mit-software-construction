# Reading 23: Queues and Message-Passing

## Two models for concurrency

The message passing model has several advantages over the shared memory model, which boil down to greater safety from bugs. In message-passing, concurrent modules interact explicitly, by passing messages through the communication channel, rather than implicitly through mutation of shared data. The implicit interaction of shared memory can too easily lead to inadvertent interaction, sharing and manipulating data in parts of the program that don’t know they’re concurrent and aren’t cooperating properly in the thread safety strategy. Message passing also shares only immutable objects (the messages) between modules, whereas shared memory requires sharing mutable objects, which we have already seen can be a source of bugs.

We’ll discuss in this reading how to implement message passing within a single process. We’ll use blocking queues (an existing threadsafe type) to implement message passing between threads within a process. Some of the operations of a blocking queue are blocking in the sense that calling the operation blocks the progress of the thread until the operation can return a result. Blocking makes writing code easier, but it also means we must continue to contend with bugs that cause deadlock.


## Message passing between threads

We saw in Locks and Synchronization that a thread blocks trying to acquire a lock until the lock has been released by its current owner. Blocking means that a thread waits (without doing further work) until an event occurs. We can use this term to describe methods and method calls: if a method is a **blocking method**, then a call to that method can block, waiting until some event occurs before it returns to the caller.

We can use a queue with blocking operations for message passing between threads — and the buffered network communication channel in client/server message passing will work the same way. Java provides the `BlockingQueue` interface for queues with blocking operations.

![](./assets/producer-consumer.png)

A `BlockingQueue` extends this interface:

additionally supports operations that wait for the queue to become non-empty when retrieving an element, and wait for space to become available in the queue when storing an element.

* `put(e)` blocks until it can add element e to the end of the queue (if the queue does not have a size bound, put will not block).

* `take()` blocks until it can remove and return the element at the head of the queue, waiting until the queue is non-empty.

When you are using a BlockingQue`ue for message passing between threads, make sure to use the `put()` and `take()` operations, not ~add()~ and ~remove()~.

We will implement the producer-consumer design pattern for message passing between threads. Producer threads and consumer threads share a synchronized queue. Producers put data or requests onto the queue, and consumers remove and process them. One or more producers and one or more consumers might all be adding and removing items from the same queue. This queue must be safe for concurrency.

Java provides two implementations of `BlockingQueue`:

* `ArrayBlockingQueue` is a fixed-size queue that uses an array representation. Using put to add an item to the queue will block if the queue is full.
* `LinkedBlockingQueue` is a growable queue using a linked-list representation. If no maximum capacity is specified, the queue will never fill up, so `put` will never block.


## Message passing example

The module has a `start` method that creates an internal thread to service requests on its input queue:

```java
/**
 * Start handling drink requests.
 */
public void start() {
    new Thread(new Runnable() {
        public void run() {
            while (true) {
                try {
                    // block until a request arrives
                    int n = in.take();
                    FridgeResult result = handleDrinkRequest(n);
                    out.put(result);
                } catch (InterruptedException ie) {
                    ie.printStackTrace();
                }
            }
        }
    }).start();
}
```


## Stopping

It is also possible to signal a thread that it should stop working by calling that thread’s `interrupt()` method. If the target thread is blocked waiting, the method it’s blocked in will throw an `Interrupted­Exception`. **That’s why we have to try-catch that exception almost any time we call a blocking method.** If the target thread was not blocked, an *interrupted* flag will be set. In order to use this approach to stop a thread, the target thread must both handle any `Interrupted­Exception`s and check for the interrupted flag to see whether it should stop working. For example:

```java
public void run() {
    // handle requests until we are interrupted
    while ( ! Thread.interrupted()) {
        try {
            // block until a request arrives
            int n = in.take();
            FridgeResult result = handleDrinkRequest(n);
            out.put(result);
        } catch (InterruptedException ie) {
            // stop
            break;
        }
    }
}
```


## Thread safety arguments with message passing

A thread safety argument with message passing might rely on:

* **Existing threadsafe data types** for the synchronized queue. This queue is definitely shared and definitely mutable, so we must ensure it is safe for concurrency.

* **Immutability** of messages or data that might be accessible to multiple threads at the same time.

* **Confinement** of data to individual producer/consumer threads. Local variables used by one producer or consumer are not visible to other threads, which only communicate with one another using messages in the queue.

* **Confinement** of mutable messages or data that are sent over the queue but will only be accessible to one thread at a time. This argument must be carefully articulated and implemented. Suppose one thread has some mutable data to send to another thread. If the first thread drops all references to the data like a hot potato as soon as it puts them onto a queue for delivery to the other thread, then only one thread will have access to those data at a time, precluding concurrent access.

In comparison to synchronization, message passing can make it easier for each module in a concurrent system to maintain its own thread safety invariants. We don’t have to reason about multiple threads accessing shared data if the data are instead transferred between modules using a threadsafe communication channel.


## Race conditions

It's still possible for concurrent message-passing processes to interleave their work in bad ways.

This particularly happens when a client must send multiple messages to the module to do what it needs, because those messages (and the client's processing of their responses) may interleave with messages sent by other clients.


## Deadlock

The blocking behavior of blocking queues is very convenient for programming, but blocking also introduces the possibility of deadlock. In a deadlock, two (or more) concurrent modules are both blocked waiting for each other to do something. Since they’re blocked, no module will be able to make anything happen, and none of them will break the deadlock.

Deadlock is much more common with locks than with message-passing — but when the message-passing queues have limited capacity, and become filled up to that capacity with messages, then even a message-passing system can experience deadlock. A message passing system in deadlock appears to simply get stuck.

### Final suggestions for preventing deadlock

One solution to deadlock is to design the system so that there is no possibility of a cycle - so that if A is waiting for B, it cannot be that B was already (or will start) waiting for A.

Another approach to deadlock is *timeouts*. If a module has been blocked for too long (maybe 100 milliseconds? or 10 seconds? how to decide?), then you stop blocking and throw an exception.
