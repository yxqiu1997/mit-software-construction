# Reading 15: Equality

## Implementing `equals()`

```java
@Override
public boolean equals(Object that) {
    return that instanceof Duration && this.sameValue((Duration)that);
}

// returns true iff this and that represent the same abstract value
private boolean sameValue(Duration that) {
    return this.getLength() == that.getLength();
}
```

The first method overrides and replaces the `equals(Object)` method inherited from `Object`. It tests the type of the `that` object passed to it to make sure it’s a `Duration`, and then calls a private helper method `sameValue()` to test equality. The expression `(Duration)that` is a typecast expression, which asserts to the Java compiler that you are confident that `that` points to a `Duration` object.


## Breaking hash tables

To understand the part of the contract relating to the `hashCode` method, you’ll need to have some idea of how hash tables work. Two very common collection implementations, `HashSet` and `HashMap`, use a hash table data structure, and depend on the `hashCode` method to be implemented correctly for the objects stored in the set and used as keys in the map.

A hash table is a representation for a mapping: an abstract data type that maps keys to values. Hash tables offer **constant time lookup**, so they tend to perform better than trees or lists. Keys don’t have to be ordered, or have any particular property, except for offering `equals` and `hashCode`.

Here’s how a hash table works. It contains an array that is initialized to a size corresponding to the number of elements that we expect to be inserted. When a key and a value are presented for insertion, we compute the hash code of the key, and convert it into an index in the array’s range (e.g., by a modulo division). The value is then inserted at that index.

The rep invariant of a hash table includes the fundamental constraint that a key can be found by starting from the slot determined by its hash code.

Hash codes are designed so that the keys will be spread evenly over the indices. But occasionally a conflict occurs, and two keys are placed at the same index. So rather than holding a single value at an index, a hash table actually holds a list of key/value pairs, usually called a hash bucket. A key/value pair is implemented in Java simply as an object with two fields. On insertion, you add a pair to the list in the array slot determined by the hash code. For lookup, you hash the key, find the right slot, and then examine each of the pairs until one is found whose key equals the query key.

Now it should be clear why the `Object` contract requires equal objects to have the same hash code. If two equal objects had distinct hash codes, they might be placed in different slots. So if you attempt to lookup a value using a key equal to the one with which it was inserted, the lookup may fail.

Note, however, that as long as you satisfy the requirement that equal objects have the same hash code value, then the particular hashing technique you use doesn’t make a difference to the correctness of your code. It may affect its performance, by creating unnecessary collisions between different objects, but even a poorly-performing hash function is better than one that breaks the contract.

Most crucially, note that if you don’t override hashCode at all, you’ll get the one from Object, which is based on the address of the object. If you have overridden equals, this will mean that you will have almost certainly violated the contract. So as a general rule:

**Always override hashCode when you override equals.**


## Equality of mutable types

Equality must still be an equivalence relation, as required by the `Object` contract.

We also want equality to respect the abstraction function and respect operations. But with mutable objects, there is a new possibility: by calling a mutator on one of the objects before doing the observation, we may change its state and thus create an observable difference between the two objects.

So let’s refine our definition and allow for two notions of equality based on observation:

* **observational equality** means that two references cannot be distinguished now, in the current state of the program. A client can try to distinguish them only by calling operations that don’t change the state of either object (i.e. only observers and producers, not mutators) and comparing the results of those operations. This tests whether the two references “look” the same for the current state of the object.

* **behavioral equality** means that two references cannot be distinguished now or in the future, even if a mutator is called to change the state of one object but not the other. This tests whether the two references will “behave” the same, in this and all future states.

For immutable objects, observational and behavioral equality are identical, because there aren’t any mutator methods that can change the state of the objects.

For mutable objects, it can be very tempting to use observational equality as the design choice. Java uses observational equality for most of its mutable data types, in fact. If two distinct `List` objects contain the same sequence of elements, then `equals()` reports that they are equal.


## Breaking a HashSet’s rep invariant

Using observational equality for mutable types seems at first like a reasonable idea. It allows, for example, two `List`s of the same integers in the same order to be `equal()`.

But the presence of mutators unfortunately leads to subtle bugs, because it means that equality isn’t consistent over time. Two objects that are observationally equal at one moment in the program may stop being equal after a mutation.

This fact allows us to easily break the rep invariants of other collection data structures. Suppose we make a `List`, and then drop it into a `HashSet`:

```java
List<String> list = new ArrayList<>();
list.add("a");

Set<List<String>> set = new HashSet<List<String>>();
set.add(list);
```

We can check that the set contains the list we put in it, and it does:

```java
set.contains(list) → true
```

But now we mutate the list:

```java
list.add("goodbye");
```

And it no longer appears in the set!

```java
set.contains(list) → false!
```

It’s worse than that, in fact: when we iterate over the members of the set, we still find the list in there, but `contains()` says it’s not there!

```java
for (List<String> l : set) { 
    set.contains(l) → false! 
}
```

What’s going on? `List<String>` is a mutable object. In the standard Java implementation of collection classes like `List`, mutations affect the result of `equals()` and `hashCode()`. When the list is first put into the HashSet, it is stored in the hash bucket corresponding to its `hashCode()` result at that time. When the list is subsequently mutated, its `hashCode()` changes, but `HashSet` doesn’t realize it should be moved to a different bucket. So it can never be found again.

When `equals()` and `hashCode()` can be affected by mutation, we can break the rep invariant of a hash table that uses that object as a key.


## The final rule for equals() and hashCode()

The lesson we should draw from this example is that **equals() should implement behavioral equality**. Otherwise instances of the type cannot be safely stored in a data structure that depends on hashing, like HashMap and HashSet.

**For immutable types**:

* Behavioral equality is the same as observational equality.
* `equals()` must be overridden to compare abstract values.
* `hashCode()` must be overriden to map the abstract value to an integer.

**For mutable types**:

* Behavioral equality is different from observational equality.
* `equals()` should generally not be overriden, but inherit the implementation from Object that compares references, just like ==.
* `hashCode()` should likewise not be overridden, but inherit Object’s implementation that maps the reference into an integer.

For a mutable type that needs a notion of observational equality (whether two mutable objects “look” the same in the current state), it’s better to define a completely new operation, which might be called `similar()` or `sameValue()`. Its implementation would be similar to the private `sameValue()` helper method we have been writing for immutable types, but it would be a public operation available to clients. Unfortunately the Java library did not make that design decision.


## Autoboxing and equality

One more instructive pitfall in Java. We’ve talked about primitive types and their object type equivalents – for example, `int` and `Integer`. The object type implements `equals()` in the correct way, so that if you create two `Integer` objects with the same value, they’ll be `equals()` to each other:

```java
Integer x = new Integer(3);
Integer y = new Integer(3);
x.equals(y) → true
```

But there’s a subtle problem here; `==` is overloaded. For reference types like `Integer`, it implements reference equality:

```java
x == y // returns false
```

But for primitive types like `int`, `==` implements behavioral equality:

```java
(int)x == (int)y // returns true
```

So you can’t really use `Integer` interchangeably with int. The fact that Java automatically converts between `int` and `Integer` (this is called *autoboxing and autounboxing*) can lead to subtle bugs! You have to be aware what the compile-time types of your expressions are. 
