# Reading 21: Thread Safety

## What threadsafe means

A data type or static method is *threadsafe* if it behaves correctly when used from multiple threads, regardless of how those threads are executed, and without demanding additional coordination from the calling code.

* “Behaves correctly” means satisfying its specification and preserving its rep invariant.

* “Regardless of how threads are executed” means threads might be on multiple processors or timesliced on the same processor.

* And “without additional coordination” means that the data type can’t put preconditions on its caller related to timing, like “you can’t call `get()` while `set()` is in progress.”

For example, remember Iterator? It’s not threadsafe when used with a mutable collection. `Iterator`’s specification says that you can’t modify a collection at the same time as you’re iterating over it (except using the iterator’s own `remove` method). That’s a timing-related precondition put on the caller, and `Iterator` makes no guarantee to behave correctly if you violate it.


## Strategy 1: confinement

Our first way of achieving thread safety is *confinement*. Thread confinement is a simple idea: you avoid races on reassignable references and mutable data by keeping the references or data confined to a single thread. Don’t give any other threads the ability to read or write them directly.

Since shared mutable state is the root cause of a race condition, confinement solves it by not sharing the mutable state.

Local variables are always thread confined. A local variable is stored in the stack, and each thread has its own stack. There may be multiple invocations of a method running at a time (in different threads or even at different levels of a single thread’s stack, if the method is recursive), but each of those invocations has its own private copy of the variable, so the variable itself is confined.

But be careful – the variable is thread confined, but if it’s an object reference, you also need to check the object it points to. If the object is mutable, then we want to check that the object is confined as well – there can’t be references to it that are reachable from any other thread.

Confinement is what makes the accesses to `n`, `i`, and `result` safe in code like this:

```java
public class Factorial {

    /**
     * Computes n! and prints it on standard output.
     * @param n must be >= 0
     */
    private static void computeFact(final int n) {
        BigInteger result = BigInteger.valueOf(1);
        for (int i = 1; i <= n; ++i) {
            System.out.println("working on fact " + n);
            result = result.multiply(BigInteger.valueOf(i));
        }
        System.out.println("fact(" + n + ") = " + result);
    }

    public static void main(String[] args) {
        new Thread(new Runnable() { // create a thread using an
            public void run() {     // anonymous Runnable
                computeFact(99);
            }
        }).start();
        computeFact(100);
    }
}
```

### Avoid global variables

Unlike local variables, static variables are not automatically thread confined.

If you have static variables in your program, then you have to make an argument that only one thread will ever use them, and you have to document that fact clearly. Better, you should eliminate the static variables entirely.

Here’s an example that follows the singleton design pattern, which uses a private static variable:

```java
// This class has a race condition in it.
public class PinballSimulator {

    private static PinballSimulator simulator = null;
    // invariant: there should never be more than one PinballSimulator
    //            object created

    private PinballSimulator() {
        System.out.println("created a PinballSimulator object");
    }

    // factory method that returns the sole PinballSimulator object,
    // creating it if it doesn't exist
    public static PinballSimulator getInstance() {
        if (simulator == null) {
            simulator = new PinballSimulator();
        }
        return simulator;
    }
}
```

This class has a race in the `getInstance()` method – two threads could call it at the same time and end up creating two copies of the `PinballSimulator` object, which we don’t want.

To fix this race using the thread confinement approach, you would specify that only a certain thread (maybe the “pinball simulation thread”) is allowed to call `PinballSimulator.getInstance()`. The risk here is that Java won’t help you guarantee this.

In general, static variables are very risky for concurrency. They might be hiding behind an innocuous function that seems to have no side-effects or mutations. Consider this example:

```java
// is this method threadsafe?
/**
 * @param x integer to test for primeness; requires x > 1
 * @return true if x is prime with high probability
 */
public static boolean isPrime(int x) {
    if (cache.containsKey(x)) return cache.get(x);
    boolean answer = BigInteger.valueOf(x).isProbablePrime(100);
    cache.put(x, answer);
    return answer;
}

private static Map<Integer,Boolean> cache = new HashMap<>();
```

This function stores the answers from previous calls in case they’re requested again. This technique is called memoization, and it’s a sensible optimization for slow functions like exact primality testing. But now the `isPrime` method is not safe to call from multiple threads, and its clients may not even realize it. The reason is that the `HashMap` object referenced by the static variable `cache` is shared by all calls to `isPrime()`, and `HashMap` is not threadsafe. If multiple threads mutate the map at the same time, by calling `cache.put()`, then the map can become corrupted in the same way that the bank account became corrupted in the last reading. If you’re lucky, the corruption may cause an exception deep in the hash map, like a `Null­Pointer­Exception` or `Index­OutOfBounds­Exception`. But it also may just quietly give wrong answers, as we saw in the bank account example.


### Confinement and fields

Returning to our `PinballSimulator`, suppose we have a mutable rep to model the balls, flippers, bumpers, etc. in an instance of the simulator:

```java
public class PinballSimulator {

    private final List<Ball> ballsInPlay;
    private final Flipper leftFlipper;
    private final Flipper rightFlipper;
    // ... other instance fields...

    // ... instance observer and mutator methods ...
}
```

Instance variables are not automatically thread confined, even if they are declared `private`. If we want to argue that the `PinballSimulator` ADT is threadsafe, we cannot use confinement. We don’t know whether or how clients have created aliases to a `PinballSimulator` instance from multiple threads. If they have, then concurrent calls to methods of this class will make concurrent access to its fields and their values.

### Confinement and anonymous classes

In the previous reading, we used anonymous Runnable instances to start new threads. This is common, but it also complicates the confinement of local variables. Let’s look at how that can happen.

```java
public class PinballSimulator {
    
    private int highScore;
    // ...
    
    public void simulate() {
        int numberOfLives = 3;
        List<Ball> ballsInPlay = new ArrayList<>();
        
        new Thread(new Runnable() {
            public void run() {
                // TODO
            }
        }).start();
    }
}
```

The `ballsInPlay` variable is a local variable of `simulate()`. Can we argue that the list it points to is confined to the original thread that called `simulate()`? No, because we can mutate `ballsInPlay` inside the new thread, e.g. with `ballsInPlay.add(..)`. Java permits the `Runnable` to read variables from its outer scope! That means the `ArrayList` object is not thread confined (and, as we’ll see, the `ArrayList` type is not threadsafe either).

What about the local variables themselves – are they still threadsafe? For example, what happens if we put `numberOfLives -= 1` in the new thread? The compiler reports a static error: “local variable defined in an enclosing scope must be final or effectively final.” That is, variables from the outer scope are only accessible if they are never reassigned. Neither reassigning them in the `Runnable` nor reading them in the `Runnable` when they are reassigned outside it are allowed. So although we can no longer argue that these shared local variables are threadsafe because of confinement, we can now argue that they are threadsafe because of immutability (as unreassignable references).

Note that the compiler helpfully allows the inner `Runnable` to read these never-reassigned variables even if they haven’t been declared `final`. But to keep the code easy to understand, always declare local variables you intend to share with an inner class as `final`.

And what happens if we try to reassign `highScore`, a field of the enclosing class, e.g. with `highScore += 1000`? Just as `numberOfLives` and `ballsInPlay` are accessible, the `this` pointer to the instance of `PinballSimulator` on which we called `simulate` is also accessible. And since the field `highScore` is reassignable, the `Runnable` will be allowed to update it. Unlike local variables, fields are neither confined nor unreassignable by default.

In a system where you create and start new threads, sharing fields and local variables with inner `Runnable` instances is common. You will need to use one of the upcoming strategies to ensure that sharing those non-confined references and objects is safe.


## Strategy 2: immutability

Our second way of achieving thread safety is by using unreassignable references and immutable data types. Immutability tackles the shared-mutable-state cause of a race condition and solves it simply by making the shared state not mutable.

A variable declared `final` is unreassignable, and is safe to access from multiple threads. You can only read the variable, not write it. Be careful, because this safety applies only to the variable itself, and we still have to argue that the object the variable points to is immutable.

Immutable objects are usually also threadsafe. We say “usually” here because our current definition of immutability is too loose for concurrent programming. We’ve said that a type is immutable if an object of the type always represents the same abstract value for its entire lifetime. But that actually allows the type the freedom to mutate its rep, as long as those mutations are invisible to clients. We’ve seen several examples of this notion, called beneficent mutation. Caching, lazy computation, and data structure rebalancing are typical kinds of beneficent mutation.

For concurrency, though, this kind of hidden mutation is not safe. An immutable data type that uses beneficent mutation will have to make itself threadsafe using locks (the same technique required of mutable data types), which we’ll talk about in a future reading.

### Threadsafe immutability

So in order to be confident that an immutable data type is threadsafe without locks, we need a stronger definition of immutability. A type is *threadsafe immutable* if it has:

* no mutator methods
* all fields declared private and final
* no representation exposure
* no mutation whatsoever of mutable objects in the rep – not even beneficent mutation

If you follow these rules, then you can be confident that your immutable type will also be threadsafe.


## Strategy 3: using threadsafe data types

Our third major strategy for achieving thread safety is to store shared mutable data in existing threadsafe data types.

When a data type in the Java library is threadsafe, its documentation will explicitly state that fact. For example, here’s what StringBuffer says:

* [`StringBuffer` is] A thread-safe, mutable sequence of characters. A string buffer is like a `String`, but can be modified. At any point in time it contains some particular sequence of characters, but the length and content of the sequence can be changed through certain method calls.

    String buffers are safe for use by multiple threads. The methods are synchronized where necessary so that all the operations on any particular instance behave as if they occur in some serial order that is consistent with the order of the method calls made by each of the individual threads involved.

This is in contrast to StringBuilder:

* [`StringBuilder` is] A mutable sequence of characters. This class provides an API compatible with `StringBuffer`, but with no guarantee of synchronization. This class is designed for use as a drop-in replacement for `StringBuffer` in places where the string buffer was being used by a single thread (as is generally the case). Where possible, it is recommended that this class be used in preference to `StringBuffer` as it will be faster under most implementations.
  
It has become common in the Java API to find two mutable data types that do the same thing, one threadsafe and the other not. The reason is what this example indicates: threadsafe data types usually incur a performance penalty compared to an unsafe type.

It’s deeply unfortunate that `StringBuffer` and `StringBuilder` are named so similarly, without any indication in the name that thread safety is the crucial difference between them. It’s also unfortunate that they don’t share a common interface, so you can’t simply swap in one implementation for the other for the times when you need thread safety. The Java collection interfaces do much better in this respect, as we’ll see next.

### Threadsafe collections

The collection interfaces in Java – `List`, `Set`, `Map` – have basic implementations that are not threadsafe. The implementations of these that you’ve been used to using, namely `ArrayList`, `HashMap`, and `HashSet`, cannot be used safely from more than one thread.

Fortunately, just like the Collections API provides wrapper methods that make collections immutable, it provides another set of wrapper methods to make collections threadsafe, while still mutable.

These wrappers effectively make each method of the collection atomic with respect to the other methods. An *atomic* action effectively happens all at once – it doesn’t interleave its internal operations with those of other actions, and none of the effects of the action are visible to other threads until the entire action is complete, so it never looks partially done.

Now we see a way to fix the `isPrime()` method we had earlier in the reading:

```java
private static Map<Integer,Boolean> cache =
                Collections.synchronizedMap(new HashMap<>());
```

* **Don’t circumvent the wrapper**. Make sure to throw away references to the underlying non-threadsafe collection, and access it only through the synchronized wrapper. That happens automatically in the line of code above, since the new `HashMap` is passed only to `synchronizedMap()` and never stored anywhere else. (We saw this same warning with the unmodifiable wrappers: the underlying collection is still mutable, and code with a reference to it can circumvent immutability.)

* **Iterators are still not threadsafe**. Even though method calls on the collection itself (`get()`, `put()`, `add()`, etc.) are now threadsafe, iterators created from the collection are still not threadsafe. So you can’t use `iterator()`, or the for loop syntax:

    ```java
    List<String> list = ...;
    for (String s: list) { ... } // not threadsafe, even if list is a synchronized list wrapper
    ```

    The solution to this iteration problem will be to acquire the collection’s lock when you need to iterate over it, which we’ll talk about in a future reading.

* Finally, **atomic operations aren’t enough to prevent races**: the way that you use the synchronized collection can still have a race condition. 
  
Even the `isPrime()` method still has potential races:

```java
if (cache.containsKey(x)) return cache.get(x);
boolean answer = BigInteger.valueOf(x).isProbablePrime(100);
cache.put(x, answer);
```

The synchronized map ensures that `containsKey()`, `get()`, and `put()` are now atomic, so using them from multiple threads won’t damage the rep invariant of the map. But those three operations can now interleave in arbitrary ways with each other, which might break the invariant that `isPrime` needs from the cache: if the cache maps an integer x to a value f, then x is prime if and only if f is true. If the cache ever fails this invariant, then we might return the wrong result.

So we have to argue that the races between `containsKey()`, `get()`, and `put()` don’t threaten this invariant. The code has two risky paths:

1. Suppose `containsKey(x)` returns true, but then another thread mutates the cache before the `get(x)` call. This is not harmful because we never remove items from the cache – once it contains a result for `x`, it will continue to do so.
2. 
3. Alternatively, suppose `containsKey(x)` returns false, but another thread mutates the cache before p`ut(x, ...)`. It may end up that the two threads both test the primeness of the same `x` at the same time, and both will race to call `put()` with the answer. But both of them should call `put(x, answer)` with the same value for `answer`, so it doesn’t matter which one wins the race – the result will be the same.

The need to make these kinds of careful arguments about safety – even when you’re using threadsafe data types – is the main reason that concurrency is hard.


## How to make a safety argument

A *safety* argument needs to catalog all the threads that exist in your module or program, and the data that they use, and argue which of the four techniques you are using to protect against races for each data object or variable: confinement, immutability, threadsafe data types, or synchronization. When you use the last two, you also need to argue that all accesses to the data are appropriately atomic – that is, that the invariants you depend on are not threatened by interleaving. We gave one of those arguments for `isPrime` above.

### Thread safety arguments for data types

Remember our four approaches to thread safety: confinement, immutability, threadsafe data types, and synchronization. Since we haven’t talked about synchronization in this reading, we won’t use it in the thread safety arguments below.

We won’t use confinement either. Confinement is not usually an option when we’re making an argument just about a data type, because you have to know what threads exist in the system and what objects they’ve been given access to. If the data type creates its own set of threads, then you can talk about confinement with respect to those threads. Otherwise, the threads are coming in from the outside, carrying client calls, and the data type may have no guarantees about which threads have references to what. So confinement isn’t a useful argument in that case. Usually we use confinement at a higher level, talking about the system as a whole and arguing why we don’t need thread safety for some of our modules or data types, because they won’t be shared across threads by design.

So we’ll focus on thread safety arguments that appeal to immutability and threadsafe data types.

We also have to avoid rep exposure, and document our argument for safety from rep exposure. Rep exposure is bad for any data type, since it threatens the data type’s rep invariant. It’s similarly fatal to thread safety.

### Good safety arguments

For example, here is a good argument that appeals to immutability:

```java
/** MyString is an immutable data type representing a string of characters. */
public class MyString {
    private final char[] a;
    // Thread safety argument:
    //    This class is threadsafe immutable:
    //    - a is private and final
    //    - a points to a mutable char array, but that array is encapsulated
    //      in this object, not shared with any other object or exposed to a
    //      client, and is never mutated
```

Here’s another rep for MyString that requires a little more care in the argument. Even though the rep is not exposed, it is aliased with other instances of the type:

```java
/** MyString is an immutable data type representing a string of characters. */
public class MyString {
    private final char[] a;
    private final int start;
    private final int len;
    // Rep invariant:
    //    0 <= start <= a.length
    //    0 <= len <= a.length-start
    // Abstraction function:
    //    AF(a,start,len) = the string of characters a[start],...,a[start+length-1]
    // Thread safety argument:
    //    This class is threadsafe immutable:
    //    - a, start, and len are final
    //    - a points to a mutable char array, which may be shared with other
    //      MyString objects, but they never mutate it
    //    - the array is never exposed to a client
```

Note that since this `MyString` rep was designed for sharing the array between multiple `MyString` objects, we have to ensure that the sharing doesn’t threaten its thread safety. As long as it doesn’t threaten the `MyString`’s immutability, however, we can be confident that it won’t threaten the thread safety.

Here is an argument that rests on using a threadsafe type in the rep, plus the immutability of the reference to it:

```java
/** Team is a mutable data type representing a team of people. */
public class Team {
    private final Set<String> people = Collections.synchronizedSet(new HashSet<>());
    // Rep invariant:
    //    true
    // Abstraction function:
    //    AF(people) = the group of people whose names are in the set `people` 
    // Thread safety argument:
    //    people is a threadsafe set, and the people variable is unreassignable.
```
