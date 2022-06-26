# Reading 8: Mutability & Immutability

## Mutability

* The difference between mutability and immutability doesn’t matter much when there’s only one reference to the object. But there are big differences in how they behave when there are other references to the object. For example, when another variable ``t`` points to the same ``String`` object as ``s``, and another variable ``tb`` points to the same ``StringBuilder`` as ``sb``, then the differences between the immutable and mutable objects become more evident:

    ```java
    String t = s;
    t = t + "c";

    StringBuilder tb = sb;
    tb.append("c");
    ```

    Changing ``t`` had no effect on ``s``, but changing ``tb`` affected ``sb`` too — possibly to the surprise of the programmer. That’s the essence of the problem we’re going to look at in this reading.

* Since we have the immutable ``String`` class already, why do we even need the mutable ``StringBuilder`` in programming? A common use for it is to concatenate a large number of strings together. Consider this code:

    ```java
    String s = "";
    for (int i = 0; i < n; ++i) {
        s = s + i;
    }
    ```

    Using immutable strings, this makes a lot of temporary copies — the first number of the string (``"0"``) is actually copied n times in the course of building up the final string, the second number is copied n-1 times, and so on. It costs O(n2) time to do all that copying, even though we only concatenated n elements.

    ``StringBuilder`` is designed to minimize this copying. It uses a simple but clever internal data structure to avoid doing any copying at all until the very end, when you ask for the final ``String`` with a ``toString()`` call. So ``StringBuilder`` enables fast string concatenation, and an “Implementation Note” in the String API docs says that Java may actually implement the ``+`` operator using a StringBuilder behind the scenes.

* Optimizing performance is one reason why we might use mutable objects. Another reason is convenient sharing: two parts of your program can communicate by deliberately sharing a common mutable data structure.


## Risks of mutation

* This example also illustrates why using mutable objects can actually be bad for performance. The simplest solution to this bug, which avoids changing any of the specifications or method signatures, is for ``startOfSpring()`` to always return a copy of the groundhog’s answer:

    ```java
    return new Date(groundhogAnswer.getTime());
    ```

* This pattern is **defensive copying**, and we’ll see much more of it when we talk about abstract data types. The defensive copy means ``partyPlanning()`` can freely stomp all over the returned date without affecting ``startOfSpring()``’s cached date. But defensive copying forces ``startOfSpring()`` to do extra work and use extra space for every client — even if 99% of the clients never mutate the date it returns. We may end up with lots of copies of the first day of spring throughout memory. If we used an immutable type instead, then different parts of the program could safely share the same values in memory, so less copying and less memory space is required. Immutability can be more efficient than mutability, because immutable types never need to be defensively copied.

    
## Specifications for mutating methods

* Here’s an example of a mutating method:

    ```
    static void sort(List<String> list)
    requires: nothing
    effects: puts list in sorted order, i.e. list[i] ≤ list[j] for all 0 ≤ i < j < list.size()
    ```

* And an example of a method that does not mutate its argument:

    ```
    static List<String> toLowerCase(List<String> list)
    requires: nothing
    effects: returns a new list t where t[i] = list[i].toLowerCase()
    ```


## Iterating over arrays and lists

* Iterators are used under the covers in Java when you’re using a ``for (... : ...)`` loop to step through a List or array. This code:

    ```java
    List<String> list = ...;
    for (String str : list) {
        System.out.println(str);
    }
    ```

    is rewritten by the compiler into something like this:

    ```java
    List<String> list = ...;
    Iterator<String> iter = list.iterator();
    while (iter.hasNext()) {
        String str = iter.next();
        System.out.println(str);
    }
    ```

* An iterator has two methods:

    * ``next()`` returns the next element in the collection

    * ``hasNext()`` tests whether the iterator has reached the end of the collection.

* Note that the ``next()`` method is a **mutator** method, not only returning an element but also advancing the iterator so that the subsequent call to ``next()`` will return a different element.


## Mutation undermines an iterator

Let’s try using our iterator for a simple job. Suppose we have a list of MIT subjects represented as strings, like ``["6.031", "8.03", "9.00"]``. We want a method ``dropCourse6`` that will delete the Course 6 subjects from the list, leaving the other subjects behind. Following good practices, we first write the spec:

```java
/**
* Drop all subjects that are from Course 6. 
* Modifies subjects list by removing subjects that start with "6."
* 
* @param subjects list of MIT subject numbers
*/
public static void dropCourse6(ArrayList<String> subjects)
```

Note that ``dropCourse6`` explicitly says in its spec that its ``subjects`` argument may be mutated.

Next, following test-first programming, we devise a testing strategy that partitions the input space, and choose test cases to cover that partition:

```java
// Testing strategy:
//   partition on subjects.size: 0, 1, n > 1
//   partition on contents: no 6.xx, some 6.xx, all 6.xx
//   partition on position: 6.xx at start, 6.xx in middle, 6.xx at end

// Test cases:
//   [] => []
//   ["8.03"] => ["8.03"]
//   ["14.03", "9.00", "21L.005"] => ["14.03", "9.00", "21L.005"]
//   ["2.001", "6.01", "18.03"] => ["2.001", "18.03"]
//   ["6.045", "6.031", "6.036"] => []
```

Finally, we implement it:

```java
public static void dropCourse6(ArrayList<String> subjects) {
    MyIterator iter = new MyIterator(subjects);
    while (iter.hasNext()) {
        String subject = iter.next();
        if (subject.startsWith("6.")) {
            subjects.remove(subject);
        }
    }
}
```

Now we run our test cases, and they work! … almost. The last test case fails:

```java
// dropCourse6(["6.045", "6.031", "6.036"])
//   expected [], actual ["6.031"]
```

We got the wrong answer: ``dropCourse6`` left a course behind in the list! Why? Trace through what happens. It will help to use a snapshot diagram showing the ``MyIterator`` object and the ``ArrayList`` object and update it while you work through the code.

Note that this isn’t just a bug in our ``MyIterator``. The built-in iterator in ``ArrayList`` suffers from the same problem, and so does the ``for`` loop that’s syntactic sugar for it. The problem just has a different symptom. If you used this code instead:

```java
for (String subject : subjects) {
    if (subject.startsWith("6.")) {
        subjects.remove(subject);
    }
}
```

then you’ll get a ``Concurrent­Modification­Exception``. The built-in iterator detects that you’re changing the list under its feet, and cries foul. (How do you think it does that?)

How can you fix this problem? One way is to use the ``remove()`` method of ``Iterator``, so that the iterator adjusts its index appropriately:

```java
Iterator iter = subjects.iterator();
while (iter.hasNext()) {
    String subject = iter.next();
    if (subject.startsWith("6.")) {
        iter.remove();
    }
}
```

This is actually more efficient as well, it turns out, because ``iter.remove()`` already knows where the element it should remove is, while ``subjects.remove()`` had to search for it again.

But this doesn’t fix the whole problem. What if there are other ``Iterator``s currently active over the same list? They won’t all be informed! And what if the mutation of the list is something more complicated than just removing an element or appending an element – say, sorting the list into a different order? How should an active ``Iterator``’s position be adjusted in that case? Mutating a list that is currently being iterated over is simply not safe in general.


## Immutability and performance

As we saw with ``StringBuilder`` above, one common reason for choosing a mutable type is performance optimization. But it’s not correct to conclude that a mutable type is always more efficient than an immutable type.

The key question to think about is how much sharing is possible, and how much copying is required. An immutable value may be safely shared by many different parts of a program, where a mutable value in the same context would have to be defensively copied over and over, at a cost of both time and space. On the other hand, if a value needs to be edited, in the sense of having many small mutations made to it, then a mutable value may be more efficient, because it doesn’t require the whole value to be copied on every edit.

But even when an immutable value needs to be heavily edited, it may still be possible to design it in a way that exploits sharing in order to reduce copying. For example, if you have an immutable string that is a million characters long, editing a single character in the middle of the string seems to require copying all the unchanged characters too. But a clever ``String`` implementation might internally share the unchanged regions of characters before and after the edit, so that even though you get a new ``String`` object as a result of the edit, it actually occupies very little additional memory space. We will see an example of this kind of implementation in a few classes, when we talk about abstract data types. This kind of internal sharing is only possible because of immutability.

The git object graph is another example of the benefit of sharing. Because commits are immutable, other commits can point to them without fear that they will become broken because the parent commit is modified (e.g. reverted or undone). Without immutability, this kind of structural sharing is impossible.


## Useful immutable types

Since immutable types avoid so many pitfalls, let’s enumerate some commonly-used immutable types in the Java API:

* The primitive types and primitive wrappers are all immutable. If you need to compute with large numbers, ``BigInteger`` and ``BigDecimal`` are **immutable**.

* Don’t use mutable ``Date``s, use the appropriate immutable type from ``java.time`` based on the granularity of timekeeping you need.

* To create immutable collections from some known values, use ``List.of``, ``Set.of``, and ``Map.of``.

* The usual implementations of ``List``, ``Set``, and ``Map`` are all mutable: ``ArrayList``, ``HashMap``, etc. The ``Collections`` utility class has methods for obtaining unmodifiable views of these mutable collections:

    * ``Collections.unmodifiableList``
    * ``Collections.unmodifiableSet``
    * ``Collections.unmodifiableMap``

    You can think of the unmodifiable view as a wrapper around the underlying list/set/map. The wrapper behaves just like the underlying collection for non-mutating operations like ``get``, ``size``, and iteration. So ``list2.size()`` is 3, and ``list2.get(0)`` is ``"a"``. But any attempt to mutate the wrapper — ``add``, ``remove``, ``put``, etc. — triggers an ``Unsupported­Operation­Exception``.

    Before we pass a mutable collection to another part of our program, we can wrap it in an unmodifiable wrapper. We should be careful at that point to forget our reference to the mutable collection (``list1`` in the example shown here), lest we accidentally mutate it. Just as a mutable object behind a ``final`` reference can be mutated, the mutable collection inside an unmodifiable wrapper can still be modified by someone with a reference to it, defeating the wrapper. One way to forget the reference is to let the local variable holding it (``list1``) go out of scope.

* Alternatively, use ``List.copyOf``, etc., to create unmodifiable shallow copies of mutable collections.

* Collections also provides methods for obtaining immutable empty collections: ``Collections.emptyList``, etc. Nothing’s worse than discovering your definitely very empty list is suddenly definitely not empty!
