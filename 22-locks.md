# Reading 22: Locks and Synchronization

## Synchronization

**The correctness of a concurrent program should not depend on accidents of timing.**

**Locks** are one synchronization technique. A lock is an abstraction that allows at most one thread to *own* it at a time. *Holding a lock* is how one thread informs other threads: “I’m working with this thing, don’t touch it right now.”

Locks have two operations:

* **acquire** allows a thread to take ownership of a lock. If a thread tries to acquire a lock currently owned by another thread, it *blocks* until the other thread releases the lock. At that point, it will contend with any other threads that are trying to acquire the lock. At most one thread can own the lock at a time.

* **release** relinquishes ownership of the lock, allowing another thread to take ownership of it.

Using a lock also tells the compiler and processor that you’re using shared memory concurrently, so that registers and caches will be written back correctly to shared storage. This avoids the problem of reordering, ensuring that the owner of a lock is always looking at up-to-date data.

**Blocking** means, in general, that a thread waits (without doing further work) until an event occurs.

An `acquire(l)` on thread 1 will block if another thread (say thread 2) is holding lock `l`. The event it waits for is thread 2 performing `release(l)`. At that point, if thread 1 can acquire `l`, it continues running its code, with ownership of the lock. It is possible that another thread (say thread 3) was also blocked on `acquire(l)`. If so, either thread 1 or 3 (the winner is nondeterministic) will take the lock `l` and continue. The other will continue to block, waiting for `release(l)` again.


## Deadlock

When used properly and carefully, locks can prevent race conditions. But then another problem rears its ugly head. Because the use of locks requires threads to wait (`acquire` blocks when another thread is holding the lock), it’s possible to get into a situation where two threads are waiting for each other — and hence neither can make progress.

**Deadlock** occurs when concurrent modules are stuck waiting for each other to do something. A deadlock may involve more than two modules, e.g., A may be waiting for B, which is waiting for C, which is waiting for A. None of them can make progress. The essential feature of deadlock is a **cycle of dependencies** like this.

You can also have deadlock without using any locks. For example, a message-passing system can experience deadlock when message buffers fill up. If a client fills up the server’s buffer with requests, and then blocks waiting to add another request, the server may then fill up the client’s buffer with results and then block itself. So the client is waiting for the server, and the server waiting for the client, and neither can make progress until the other one does. Again, deadlock ensues.


## Developing a threadsafe abstract data type

Recall our recipe for designing and implementing an ADT. In all these steps, we’re working entirely single-threaded at first. Multithreaded clients should be in the back of our minds at all times while we’re writing specs and choosing reps (we’ll see later that careful choice of operations may be necessary to avoid race conditions in the clients of your datatype). But get it working, and thoroughly tested, in a sequential, single-threaded environment first.

4. **Synchronize**. Make an argument that your rep is threadsafe. Write it down explicitly as a comment in your class, right by the rep invariant, so that a maintainer knows how you designed thread safety into the class.

5. **Iterate**. You may find that your choice of operations makes it hard to write a threadsafe type with the guarantees clients require. You might discover this in step 1, or in step 2 when you write tests, or in steps 3 or 4 when you implement. If that’s the case, go back and refine the set of operations your ADT provides.


## Locking

Locks are so commonly-used that Java provides them as a built-in language feature.

In Java, every object has a lock implicitly associated with it — a `String`, an array, an `ArrayList`, and every class you create, all of their object instances have a lock. Even a humble `Object` has a lock, so bare `Object`s are often used for explicit locking:

```java
Object lock = new Object();
```

You can’t call `acquire` and `release` on Java’s intrinsic locks, however. Instead you use the `synchronized` statement to acquire the lock for the duration of a statement block:

```java
synchronized (lock) { // thread blocks here until lock is free
    // now this thread has the lock
    balance = balance + 1;
} // exiting the block releases the lock
```

Synchronized regions like this provide **mutual exclusion**: only one thread at a time can be in a synchronized region guarded by a given object’s lock. In other words, you are back in sequential programming world, with only one thread running at a time, at least with respect to other synchronized regions that refer to the same object.

### Locks guard access to data

Locks are used to **guard** a shared data variable, like the account balance shown here. If all accesses to a data variable are guarded (surrounded by a synchronized block) by the same lock object, then those accesses will be guaranteed to be atomic — uninterrupted by other threads.

Locks only provide mutual exclusion with other threads that acquire the same lock. **All accesses to a data variable must be guarded by the same lock.** You might guard an entire collection of variables behind a single lock, but all modules must agree on which lock they will all acquire and release.

Because every object in Java has a lock implicitly associated with it, you might think that owning an object’s lock would automatically prevent other threads from accessing that object. **That is not the case.** When a thread t acquires an object’s lock using `synchronized (obj) { ... }`, it does one thing and one thing only: it prevents other threads from entering their own `synchronized(expression)` block, where expression refers to the same object as `obj`, until thread t finishes its synchronized block. That’s it. Even while t is in its synchronized block, another thread can dangerously mutate the object, simply by neglecting to use `synchronized` itself. In order to use an object lock for synchronization, you have to explicitly and carefully guard every such access with an appropriate `synchronized` block or method keyword.


## Monitor pattern

When you are writing methods of a class, the most convenient lock is the object instance itself, i.e. `this`. As a simple approach, we can guard the entire rep of a class by wrapping all accesses to the rep inside `synchronized (this)`.

```java
/** SimpleBuffer is a threadsafe EditBuffer with a simple rep. */
public class SimpleBuffer implements EditBuffer {
    private String text;
    ...
    public SimpleBuffer() {
        synchronized (this) {
            text = "";
            checkRep();
        }
    }
    public void insert(int position, String insertion) {
        synchronized (this) {
            text = text.substring(0, position) + insertion + text.substring(position);
            checkRep();
        }
    }
    public void delete(int position, int len) {
        synchronized (this) {
            text = text.substring(0, position) + text.substring(position+len);
            checkRep();
        }
    }
    public int length() {
        synchronized (this) {
            return text.length();
        }
    }
    public String toString() {
        synchronized (this) {
            return text;
        }
    }
}
```

Note the very careful discipline here. Every method is guarded with the lock — even apparently small and trivial ones like `length()` and `toString()`. This is because reads must be guarded as well as writes — if reads are left unguarded, then they may be able to see the rep in a partially-modified state.

This approach is called the **monitor pattern**. A monitor is a class whose methods are mutually exclusive, so that only one thread can be inside an instance of the class at a time.

Java provides some syntactic sugar that helps with the monitor pattern. If you add the keyword `synchronized` to a method signature, then Java will act as if you wrote `synchronized (this)` around the method body. So the code below is an equivalent way to implement the synchronized `SimpleBuffer`:

```java
/** SimpleBuffer is a threadsafe EditBuffer with a simple rep. */
public class SimpleBuffer implements EditBuffer {
    private String text;
    ...
    public SimpleBuffer() {
        text = "";
        checkRep();
    }
    public synchronized void insert(int position, String insertion) {
        text = text.substring(0, position) + insertion + text.substring(position);
        checkRep();
    }
    public synchronized void delete(int position, int len) {
        text = text.substring(0, position) + text.substring(position+len);
        checkRep();
    }
    public synchronized int length() {
        return text.length();
    }
    public synchronized String toString() {
        return text;
    }
}
```

Notice that the `SimpleBuffer` constructor doesn’t have a `synchronized` keyword. Java actually forbids it, syntactically, because an object under construction is expected to be confined to a single thread until it has returned from its constructor. **So synchronizing constructors should be unnecessary.** You can still synchronize a constructor by wrapping its body in a `synchronized(this)` block, if you wish.


## Locking discipline

A locking discipline is a strategy for ensuring that synchronized code is threadsafe. We must satisfy two conditions:

1. Every shared mutable variable must be guarded by some lock. The data may not be read or written except inside a synchronized block that acquires that lock.

2. If an invariant involves multiple shared mutable variables (which might even be in different objects), then all the variables involved must be guarded by the same lock. Once a thread acquires the lock, the invariant must be reestablished before releasing the lock.

The monitor pattern as used here satisfies both rules. All the shared mutable data in the rep — which the rep invariant depends on — are guarded by the same lock.


## Atomic operations

This method makes three different calls to `buf` — to convert it to a string in order to search for `pattern`, to delete the old text, and then to insert `replacement` in its place. Even though each of these calls individually is atomic, the `findReplace` method as a whole is not threadsafe, because other threads might mutate the buffer while `findReplace` is working, causing it to delete the wrong region or put the replacement back in the wrong place.

To prevent this, `findReplace` needs to synchronize with all other clients of `buf`.

```java
public static boolean findReplace(EditBuffer buf, String pattern, String replacement) {
    synchronized (buf) {
        int i = buf.toString().indexOf(pattern);
        if (i == -1) {
            return false;
        }
        buf.delete(i, pattern.length());
        buf.insert(i, replacement);
        return true;
    }
}
```


## Designing a datatype for concurrency

`findReplace`’s problem can be interpreted another way: that the `EditBuffer` interface really isn’t that friendly to multiple simultaneous clients. It relies on integer indexes to specify insert and delete locations, which are extremely brittle to other mutations. If somebody else inserts or deletes before the index position, then the index becomes invalid.

So if we’re designing a datatype specifically for use in a concurrent system, we need to think about providing operations that have better-defined semantics when they are interleaved. For example, it might be better to pair `EditBuffer` with a `Position` datatype representing a cursor position in the buffer, or even a `Selection` datatype representing a selected range. Once obtained, a `Position` could hold its location in the text against the wash of insertions and deletions around it, until the client was ready to use that `Position`. If some other thread deleted all the text around the `Position`, then the `Position` would be able to inform a subsequent client about what had happened (perhaps with an exception), and allow the client to decide what to do. These kinds of considerations come into play when designing a datatype for concurrency.

As another example, consider the `ConcurrentMap` interface in Java. This interface extends the existing `Map` interface, adding a few key methods that are commonly needed as atomic operations on a shared mutable map, e.g.:

* `map.putIfAbsent(key,value)` is an atomic version of
  `if ( ! map.containsKey(key)) map.put(key, value)`;
* `map.replace(key, value)` is an atomic version of
  `if (map.containsKey(key)) map.put(key, value);`


## Deadlock rears its ugly head

Suppose we’re modeling the social network of a series of books:

```java
public class Wizard {
    private final String name;
    private final Set<Wizard> friends;
    // Rep invariant:
    //    friend links are bidirectional: 
    //        for all f in friends, f.friends contains this
    // Concurrency argument:
    //    threadsafe by monitor pattern: all accesses to rep 
    //    are guarded by this object's lock

    public Wizard(String name) {
        this.name = name;
        this.friends = new HashSet<Wizard>();
    }

    public synchronized boolean isFriendsWith(Wizard that) {
        return this.friends.contains(that);
    }

    public synchronized void friend(Wizard that) {
        if (friends.add(that)) {
            that.friend(this);
        } 
    }

    public synchronized void defriend(Wizard that) {
        if (friends.remove(that)) {
            that.defriend(this);
        } 
    }
}
```

Let’s create a couple of wizards:

```java
    Wizard harry = new Wizard("Harry Potter");
    Wizard snape = new Wizard("Severus Snape");
```

And then think about what happens when two independent threads are repeatedly running:

```java
    // thread A                   // thread B
    while (true) {                while (true) {
      harry.friend(snape);          snape.friend(harry);
      harry.defriend(snape);        snape.defriend(harry);
    }                             }
```

We will deadlock very rapidly. Here’s why. Suppose thread A is about to execute `harry.friend(snape)`, and thread B is about to execute `snape.friend(harry)`.

* Thread A acquires the lock on `harry` (because the friend method is synchronized).
* Then thread B acquires the lock on `snape` (for the same reason).
* They both update their individual reps independently, and then try to call `friend()` on the other object — which requires them to acquire the lock on the other object.

So A is holding Harry and waiting for Snape, and B is holding Snape and waiting for Harry. Both threads are stuck in `friend()`, so neither one will ever manage to exit the synchronized region and release the lock to the other. This is a classic deadly embrace. The program simply stops.

The essence of the problem is acquiring multiple locks, and holding some of the locks while waiting for another lock to become free.

Notice that it is possible for thread A and thread B to interleave such that deadlock does not occur: perhaps thread A acquires and releases both locks before thread B has enough time to acquire the first one. If the locks involved in a deadlock are also involved in a race condition — and very often they are — then the deadlock will be just as difficult to reproduce or debug.

### Deadlock solution 1: lock ordering

One way to prevent deadlock is to put an ordering on the locks that need to be acquired simultaneously, and ensuring that all code acquires the locks in that order.

```java
    public void friend(Wizard that) {
        Wizard first, second;
        if (this.name.compareTo(that.name) < 0) {
            first = this; second = that;
        } else {
            first = that; second = this;
        }
        synchronized (first) {
            synchronized (second) {
                if (friends.add(that)) {
                    that.friend(this);
                } 
            }
        }
    }
```

Although lock ordering is useful (particularly in code like operating system kernels), it has a number of drawbacks in practice.

* First, it’s not modular — the code has to know about all the locks in the system, or at least in its subsystem.

* Second, it may be difficult or impossible for the code to know exactly which of those locks it will need before it even acquires the first one. It may need to do some computation to figure it out. Think about doing a depth-first search on the social network graph, for example — how would you know which nodes need to be locked, before you’ve even started looking for them?

### Deadlock solution 2: coarse-grained locking

A more common approach than lock ordering, particularly for application programming (as opposed to operating system or device driver programming), is to use coarser locking — use a single lock to guard many object instances, or even a whole subsystem of a program.

For example, we might have a single lock for an entire social network, and have all the operations on any of its constituent parts synchronize on that lock. In the code below, all `Wizard`s belong to a `Castle`, and we just use that `Castle` object’s lock to synchronize:

```java
public class Wizard {
    private final Castle castle;
    private final String name;
    private final Set<Wizard> friends;
    ...
    public void friend(Wizard that) {
        synchronized (castle) {
            if (this.friends.add(that)) {
                that.friend(this);
            }
        }
    }
}
```

Coarse-grained locks can have a significant performance penalty. If you guard a large pile of mutable data with a single lock, then you’re giving up the ability to access any of that data concurrently. In the worst case, having a single lock protecting everything, your program might be essentially sequential — only one thread is allowed to make progress at a time.


## Concurrency in practice

### Goals

When we ask whether a concurrent program is safe from bugs, we care about two properties:

* **Safety**. Does the concurrent program satisfy its invariants and its specifications? Races in accessing mutable data threaten safety. Safety asks the question: can you prove that **some bad thing never happens**?

* **Liveness**. Does the program keep running and eventually do what you want, or does it get stuck somewhere waiting forever for events that will never happen? Can you prove that **some good thing eventually happens**?

Deadlocks threaten liveness. Liveness may also require fairness, which means that concurrent modules are given processing capacity to make progress on their computations. Fairness is mostly a matter for the operating system’s thread scheduler, but you can influence it (for good or for ill) by setting thread priorities.

### Strategies

What strategies are typically followed in real programs?

* **Library data structures** either use no synchronization (to offer high performance to single-threaded clients, while leaving it to multithreaded clients to add locking on top) or the monitor pattern.

* **Mutable data structures with many parts** typically use either coarse-grained locking or thread confinement. Most graphical user interface toolkits follow one of these approaches, because a graphical user interface is basically a big mutable tree of mutable objects. Java Swing, the graphical user interface toolkit, uses thread confinement. Only a single dedicated thread is allowed to access Swing’s tree. Other threads have to pass messages to that dedicated thread in order to access the tree.

* **Search** often uses immutable datatypes. Our Boolean formula satisfiability search would be easy to make multithreaded, because all the datatypes involved were immutable. There would be no risk of either races or deadlocks.

* **Operating systems** often use fine-grained locks in order to get high performance, and use lock ordering to deal with deadlock problems.

We’ve omitted one important approach to mutable shared data because it’s outside the scope of this course, but it’s worth mentioning: **a database**. Database systems are widely used for distributed client/server systems like web applications. Databases avoid race conditions using *transactions*, which are similar to synchronized regions in that their effects are atomic, but they don’t have to acquire locks, though a transaction may fail and be rolled back if it turns out that a race occurred. Databases can also manage locks, and handle locking order automatically.
