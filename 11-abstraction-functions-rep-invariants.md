# Reading 11: Abstraction Functions & Rep Invariants


## Invariants

### Immutability

```java
public class Tweet {

    private final String author;
    private final String text;
    private final Date timestamp;

    public Tweet(String author, String text, Date timestamp) {
        this.author = author;
        this.text = text;
        this.timestamp = timestamp;
    }

    /**
     * @return Twitter user who wrote the tweet
     */
    public String getAuthor() {
        return author;
    }

    /**
     * @return text of the tweet
     */
    public String getText() {
        return text;
    }

    /**
     * @return date/time when the tweet was sent
     */
    public Date getTimestamp() {
        return timestamp;
    }

}
```

The rep is still exposed! Consider this perfectly reasonable client code that uses `Tweet`:

```java
/**
 * @return a tweet that retweets t, one hour later
 */
public static Tweet retweetLater(Tweet t) {
    Date d = t.getTimestamp();
    d.setHours(d.getHours()+1);
    return new Tweet("rbmllr", t.getText(), d);
}
```

What’s the problem here? The `getTimestamp` call returns a reference to the same Date object referenced by tweet `t`. `t.timestamp` and d are aliases to the same mutable object. So when that date object is mutated by `d.setHours()`, this affects the date in `t` as well, as shown in the snapshot diagram.

`Tweet`’s immutability invariant has been broken. The problem is that `Tweet` leaked out a reference to a mutable object that its immutability depended on. We exposed the rep, in such a way that `Tweet` can no longer guarantee that its objects are immutable. Perfectly reasonable client code created a subtle bug.

We can patch this kind of rep exposure by using **defensive copying**: making a copy of a mutable object to avoid leaking out references to the rep. Here’s the code:

```java
public Date getTimestamp() {
    return new Date(timestamp.getTime());
}
```

Mutable types often have a copy constructor that allows you to make a new instance that duplicates the value of an existing instance. In this case, `Date`’s copy constructor uses the timestamp value, measured in milliseconds since January 1, 1970. As another example, `StringBuilder`’s copy constructor takes a `String`. Another way to copy a mutable object is `clone()`, which is supported by some types but not all. There are unfortunate problems with the way `clone()` works in Java, and you generally shouldn’t use it.

Again, the immutability of Tweet has been violated. We can fix this problem, too, by using judicious defensive copying, this time in the constructor:

```java
public Tweet(String author, String text, Date timestamp) {
    this.author = author;
    this.text = text;
    this.timestamp = new Date(timestamp.getTime());
}
```

In general, you should carefully inspect the argument types and return types of all your ADT operations. If any of the types are mutable, make sure your implementation doesn’t store those arguments directly in the rep, or return direct references to the rep, because that causes rep exposure.

### Immutable wrappers around mutable data types

The Java collections classes offer an interesting compromise: immutable wrappers.

`Collections.unmodifiableList()` takes a (mutable) `List` and wraps it with an object that looks like a `List`, but whose mutators are disabled – `set()`, `add()`, `remove()`, etc. throw exceptions. So you can construct a list using mutators, then seal it up in an unmodifiable wrapper (and throw away your reference to the original mutable list, as discussed in Mutability & Immutability), and get an immutable list.

The downside here is that you get immutability at runtime, but not at compile time. Java won’t warn you at compile time if you try to `sort()` this unmodifiable list. You’ll just get an exception at runtime. And there will be no error at all if the original mutable list is modified, changing the value of the “unmodifiable” list. But the runtime exception for attempting to mutate the wrapper is still better than nothing, so using unmodifiable lists, maps, and sets can be a very good way to reduce the risk of bugs.

Because the Java library uses these wrappers and doesn’t provide separate types for immutable collections, other creator operations that return unmodifiable lists, like `Collections.emptyList()` or `List.of(..)`, have the same downside: immutability at runtime only, with no compile-time checking.


## Rep invariant and abstraction function

The space of abstract values consists of the values that the type is designed to support, from the client’s point of view. For example, an abstract type for unbounded integers, like Java’s `BigInteger`, would have the mathematical integers as its abstract value space.

The space of representation values (or rep values for short) consists of the Java objects that actually implement the abstract values. For example, a `BigInteger` value might be implemented using an array of digits, represented as primitive `int` values. The rep space would then be the set of all such arrays.

In simple cases, an abstract type will be implemented as a single Java object, but more commonly a small network of objects is needed. For example, a rep value for `List` might be a linked list, a group of objects linked together by next and previous pointers. So a rep value is not necessarily a single object, but often something rather complicated.


## Beneficent mutation

Recall that a type is immutable if and only if a value of the type never changes after being created. With our new understanding of the abstract space A and rep space R, we can refine this definition: the abstract value should never change. But the implementation is free to mutate a rep value as long as it continues to map to the same abstract value, so that the change is invisible to the client. This kind of change is called **beneficent mutation**.


## Documenting the AF, RI, and safety from rep exposure

Here’s an example of `Tweet` with its rep invariant, abstraction function, and safety from rep exposure fully documented:

```java
// Immutable type representing a tweet.
public class Tweet {

    private final String author;
    private final String text;
    private final Date timestamp;

    // Rep invariant:
    //   author is a Twitter username (a nonempty string of letters, digits, underscores)
    //   text.length <= 280
    // Abstraction function:
    //   AF(author, text, timestamp) = a tweet posted by author, with content text, 
    //                                 at time timestamp 
    // Safety from rep exposure:
    //   All fields are private;
    //   author and text are Strings, so are guaranteed immutable;
    //   timestamp is a mutable Date, so Tweet() constructor and getTimestamp() 
    //        make defensive copies to avoid sharing the rep's Date object with clients.

    // Operations (specs and method bodies omitted to save space)
    public Tweet(String author, String text, Date timestamp) { ... }
    public String getAuthor() { ... }
    public String getText() { ... }
    public Date getTimestamp() { ... }
}
```

Here are the arguments for `RatNum`.

```java
// Immutable type representing a rational number.
public class RatNum {
    private final int numerator;
    private final int denominator;

    // Rep invariant:
    //   denominator > 0
    //   numerator/denominator is in reduced form, i.e. gcd(|numerator|,denominator) = 1
    // Abstraction function:
    //   AF(numerator, denominator) = numerator/denominator
    // Safety from rep exposure:
    //   All fields are private, and all types in the rep are immutable.

    // Operations (specs and method bodies omitted to save space)
    public RatNum(int n) { ... }
    public RatNum(int n, int d) { ... }
    ...
}
```

### What an ADT specification may talk about

![](./assets/adt-firewall%20(1).svg)

The specification of an abstract type T – which consists of the specs of its operations – should only talk about things that are visible to the client. This includes parameters, return values, and exceptions thrown by its operations. Whenever the spec needs to refer to a value of type *T*, it should describe the value as an abstract value, i.e. mathematical values in the abstract space A.

The spec should not talk about details of the representation, or elements of the rep space R. It should consider the rep itself (the private fields) invisible to the client, just as method bodies and their local variables are considered invisible. That’s why we write the rep invariant and abstraction function as ordinary comments within the body of the class, rather than Javadoc comments above the class. Writing them as Javadoc comments would commit to them as public parts of the type’s specification, which would interfere with rep independence and information hiding.


## ADT invariants replace preconditions

Now let’s bring a lot of pieces together. An enormous advantage of a well-designed abstract data type is that it encapsulates and enforces properties that we would otherwise have to stipulate in a precondition. For example, instead of a spec like this, with an elaborate precondition:

```java
/** 
 * @param set1 is a sorted set of characters with no repeats
 * @param set2 is likewise
 * @return characters that appear in one set but not the other,
 *  in sorted order with no repeats 
 */
static String exclusiveOr(String set1, String set2);
```

We can instead use an ADT that captures the desired property:

```java
/**
 * @return characters that appear in one set but not the other
 */
static SortedSet<Character> exclusiveOr(SortedSet<Character> set1, SortedSet<Character> set2);
```

This hits all our targets:

* it’s safer from bugs, because the required condition (sorted with no repeats) can be enforced in exactly one place, the `SortedSet` type, and because Java static checking comes into play, preventing values that don’t satisfy this condition from being used at all, with an error at compile-time.

* it’s easier to understand, because it’s much simpler, and the name `SortedSet` conveys what the programmer needs to know.

* it’s more ready for change, because the representation of SortedSet can now be changed without changing exclusiveOr or any of its clients.

Many of the places where we used preconditions on the early problem sets in this course would have benefited from a custom ADT instead.


## How to establish invariants

An invariant is a property that is true for the entire program – which in the case of an invariant about an object, reduces to the entire lifetime of the object.

To make an invariant hold, we need to:

* make the invariant true in the initial state of the object; and
* ensure that all changes to the object keep the invariant true.

Translating this in terms of the types of ADT operations, this means:

* creators and producers must establish the invariant for new object instances; and
* mutators, observers, and producers must preserve the invariant for existing object instances.

The risk of rep exposure makes the situation more complicated. If the rep is exposed, then the object might be changed anywhere in the program, not just in the ADT’s operations, and we can’t guarantee that the invariant still holds after those arbitrary changes.

So the full rule for proving invariants is as follows:

* If an invariant of an abstract data type is

    1. established by creators and producers;
    2. preserved by mutators, observers, and producers; and
    3. no representation exposure occurs,

    then the invariant is true of all instances of the abstract data type.

This rule applies structural induction over the abstract data type.
