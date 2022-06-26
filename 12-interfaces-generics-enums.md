# Reading 12: Defining ADTs with Interfaces, Generics, Enums, and Functions


## Interfaces

Java’s **interface** is a useful language mechanism for expressing an abstract data type. An interface in Java is a list of method signatures without method bodies. A class implements an interface if it declares the interface in its **implements** clause, and provides method bodies for **all of the interface’s methods**. So one way to define an abstract data type in Java is as an interface. The type’s implementation is then a class implementing that interface.

One advantage of this approach is that the interface specifies the contract for the client and nothing more. The interface is all a client programmer needs to read to understand the ADT. The client can’t create inadvertent dependencies on the ADT’s rep, because fields can’t be put in an interface at all. The implementation is kept well and truly separated, in a different class altogether.

Another advantage is that multiple different representations of the abstract data type can coexist in the same program, as different classes implementing the interface. When an abstract data type is represented just as a single class, without an interface, it’s harder to have multiple representations. 

### About interfaces in Java

Syntactically, a Java interface contains the spec of an ADT, namely its public method signatures. Each instance method signature ends with a semicolon.

An interface does not include information about the rep, so **it should not declare any instance variables or include instance method bodies**.

Because an interface has no rep, Java doesn’t allow it to be instantiated with `new`. The expression `new List<String>()` is a static error. To make a `List`, you need to instead instantiate a class that provides a rep for it, like `new ArrayList<String>()`.

For the same reason, Java interfaces are not allowed to declare constructors. There is no `List()` constructor in the `List` interface. The creator operations of an interface ADT must either be constructors of their implementation classes, like `ArrayList()` and `LinkedList()`, or static methods like `List.of()`.


## Subtypes

Recall that a type is a set of values. The Java `List` type is defined by an interface. If we think about all possible List values, none of them are actually objects of type List: we cannot create instances of an interface. Instead, those values are all `ArrayList` objects, or `LinkedList` objects, or objects of another class that implements `List`. A subtype is simply a subset of the supertype: `ArrayList` and `LinkedList` are subtypes of `List`.

“B is a subtype of A” means “every B is an A.” In terms of specifications: “every B satisfies the specification for A.”

That means B is only a subtype of A if B’s specification is at least as strong as A’s specification. When we declare a class that implements an interface, the Java compiler enforces part of this requirement automatically: for example, it ensures that every method in A appears in B, with a compatible type signature. Class B cannot implement interface A without implementing all of the methods declared in A.

But the compiler cannot check that we haven’t weakened the specification in other ways: strengthening the precondition on some inputs to a method, weakening a postcondition, weakening a guarantee that the interface abstract type advertises to clients. If you declare a subtype in Java — implementing an interface is our current focus — then you must ensure that the subtype’s spec is at least as strong as the supertype’s.

To declare that a class B is a subtype of an interface A, use `implements`:

```java
class ArrayList<E> implements List<E> {
    ...
}
```


## Example: MyString

```java
/**
 * MyString represents an immutable sequence of characters.
 */
public interface MyString { 

    // We'll skip this creator operation for now
    // /**
    //  * @param b a boolean value
    //  * @return string representation of b, either "true" or "false"
    //  */
    // public static MyString valueOf(boolean b) { ... }

    /**
     * @return number of characters in this string
     */
    public int length();

    /**
     * @param i character position (requires 0 <= i < string length)
     * @return character at position i
     */
    public char charAt(int i);

    /**
     * Get the substring between start (inclusive) and end (exclusive).
     * @param start starting index
     * @param end ending index.  Requires 0 <= start <= end <= string length.
     * @return string consisting of charAt(start)...charAt(end-1)
     */
    public MyString substring(int start, int end);
}
```

```java
public class FastMyString implements MyString {

    private char[] a;
    private int start;
    private int end;

    /**
     * Create a string representation of b, either "true" or "false".
     * @param b a boolean value
     */
    public FastMyString(boolean b) {
        a = b ? new char[] { 't', 'r', 'u', 'e' } 
              : new char[] { 'f', 'a', 'l', 's', 'e' };
        start = 0;
        end = a.length;
    }

    // private constructor, used internally by producer operations.
    private FastMyString(char[] a, int start, int end) {
        this.a = a;
        this.start = start;
        this.end = end;
    }

    @Override public int length() { return end - start; }

    @Override public char charAt(int i) { return a[start + i]; }

    @Override public MyString substring(int start, int end) {
        return new FastMyString(this.a, this.start + start, this.start + end);
    }
}
```

* Also notice the use of `@Override`. This annotation informs the compiler that the method must have the same signature as one of the methods in the interface we’re implementing. `@Override` also tells readers of the code to look for the spec of that method in the interface. Repeating the spec wouldn’t be DRY, but saying nothing at all makes the code harder to understand.

* And notice that we have added a private constructor for producers like `substring(..)` to use to make new instances of the class. Its arguments are the rep fields. We didn’t have to write this special constructor before, because Java provides an empty constructor by default when we don’t declare any other constructors. Adding the constructors that take `boolean b` means we have to declare another constructor explicitly for producer operations to use.

Unfortunately, this pattern breaks the abstraction barrier we’ve worked so hard to build between the abstract type and its concrete representations. Clients must know the name of the concrete representation class. Because interfaces in Java cannot contain constructors, they must directly call one of the concrete class’ constructors. The spec of that constructor won’t appear anywhere in the interface, so there’s no static guarantee that different implementations will even provide the same constructors.

Fortunately, (as of Java 8) interfaces are allowed to contain static methods, so we can implement the creator operation `valueOf` as a static factory method in the interface `MyString`:

```java
public interface MyString { 

    /**
     * @param b a boolean value
     * @return string representation of b, either "true" or "false"
     */
    public static MyString valueOf(boolean b) {
        return new FastMyString(b);
    }

    // ...
```

## Why interfaces?

Interfaces are used pervasively in real Java code. Not every class is associated with an interface, but there are a few good reasons to bring an interface into the picture.

* **Documentation for both the compiler and for humans**. Not only does an interface help the compiler catch ADT implementation bugs, but it is also much more useful for a human to read than the code for a concrete implementation. Such an implementation intersperses ADT-level types and specs with implementation details.

* **Allowing performance trade-offs**. Different implementations of the ADT can provide methods with very different performance characteristics. Different applications may work better with different choices, but we would like to code these applications in a way that is representation-independent. From a correctness standpoint, it should be possible to drop in any new implementation of a key ADT with simple, localized code changes.

* **Methods with intentionally underdetermined specifications**. An ADT for finite sets could leave unspecified the element order one gets when converting to a list. Some implementations might use slower method implementations that manage to keep the set representation in some sorted order, allowing quick conversion to a sorted list. Other implementations might make many methods faster by not bothering to support conversion to sorted lists.

* **Multiple views of one class**. A Java class may implement multiple interfaces. For instance, a user interface widget displaying a drop-down list is natural to view as both a widget and a list. The class for this widget could implement both interfaces. In other words, we don’t always implement an ADT multiple times just because we are choosing different data structures. Those implementations may be various different sorts of objects that can be seen as special cases of the ADT.

* **More and less trustworthy implementations**. Another reason to implement an interface more than once might be that it is easy to build a simple implementation that you believe is correct, while you can work harder to build a fancier version that is more likely to contain bugs. You can choose implementations for applications based on how bad it would be to get bitten by a bug.


# Subclassing

Implementing an interface is one way to make a subtype of another type. Another technique, which you may have seen before, is subclassing. Subclassing means defining a class as an extension, or subclass, of another class, so that:

* the subclass automatically **inherits** the instance methods of its superclass, **including their method bodies**

    * the subclass can override any of those inherited method bodies by substituting its own method body

* the subclass also **inherits the fields** of its superclass

* the subclass can also add new instance methods and fields for its own purposes

The boldfaced parts in the list above show how subclassing differs from implementing an interface. An interface has neither method bodies nor fields, so it’s not possible to inherit them when you implement an interface.

Java provides the same effect with the syntax `class SpottedTurtle extends Turtle { ... }`.


## Generic types

A useful feature of the Java type system, and other modern statically-typed languages too, is a generic type: a type whose specification is in terms of a placeholder type to be filled in later.

Java’s collection classes are good examples of this. `Set<E>` is the ADT of finite sets of elements of some other type E. Instead of writing separate specifications and implementations for `Set<String>`, `Set<Integer>`, and so on, we can design one interface `Set<E>`, which represents an entire family of set ADTs.

Here is a simplified version of the Set interface:

```java
/**
 * A mutable set.
 * @param <E> type of elements in the set
 */
public interface Set<E> {
```

Let’s start with a creator operation:

```java
    // example creator operation
    /**
     * Make an empty set.
     * @param <F> type of elements in the set
     * @return a new set instance, initially empty
     */
    public static <F> Set<F> make() { ... }
```


## Enumerations

When the set of values is small and finite, it makes sense to define all the values as named constants, called an **enumeration**. Java has the `enum construct` to make this convenient:

```java
public enum Month { JANUARY, FEBRUARY, MARCH, ..., DECEMBER };
```

This enum defines a new type name, `Month`, in the same way that `class` and `interface` define new type names. It also defines a set of named values, which we write in all-caps because they are effectively `public static final` constants. So you can now write:

```java
Month thisMonth = MARCH;
```

Unlike primitive values, a variable of enum type can be null, which we must avoid the same way we do with other object types.

An `enum` declaration can contain all the usual fields and methods that a `class` can. So you can define additional operations for the ADT, and define your own rep as well. Here is an example that has a rep, an observer, and a producer:

```java
public enum Month {
    // the values of the enumeration, written as calls to the private constructor below
    JANUARY(31),
    FEBRUARY(28),
    MARCH(31),
    APRIL(30),
    MAY(31),
    JUNE(30),
    JULY(31),
    AUGUST(31),
    SEPTEMBER(30),
    OCTOBER(31),
    NOVEMBER(30),
    DECEMBER(31);

    // rep
    private final int daysInMonth;

    // enums also have an automatic, invisible rep field:
    //   private final int ordinal;
    // which takes on values 0, 1, ... for each value in the enumeration.

    // rep invariant:
    //   daysInMonth is the number of days in this month in a non-leap year
    // abstraction function:
    //   AF(ordinal,daysInMonth) = the (ordinal+1)th month of the Gregorian calendar
    // safety from rep exposure:
    //   all fields are private, final, and have immutable types

    // Make a Month value. Not visible to clients, only used to initialize the
    // constants above.
    private Month(int daysInMonth) {
        this.daysInMonth = daysInMonth;
    }

    /**
     * @param isLeapYear true iff the year under consideration is a leap year
     * @return number of days in this month in a normal year (if !isLeapYear) 
     *                                           or leap year (if isLeapYear)
     */
    public int getDaysInMonth(boolean isLeapYear) {
        if (this == FEBRUARY && isLeapYear) {
            return daysInMonth+1;
        } else {
            return daysInMonth;
        }
    }

    /**
     * @return first month of the semester after this month
     */
    public Month startOfNextSemester() {
        switch (this) {
            case JANUARY:
                return FEBRUARY;
            case FEBRUARY:   // cases with no break or return
            case MARCH:      // fall through to the next case
            case APRIL:
            case MAY:
                return JUNE;
            case JUNE:
            case JULY:
            case AUGUST:
                return SEPTEMBER;
            case SEPTEMBER:
            case OCTOBER:
            case NOVEMBER:
            case DECEMBER:
                return JANUARY;
            default:
                throw new RuntimeException("can't get here");
        }
    }
}
```

All `enum` types also have some automatically-provided operations, defined by `Enum`:

* `ordinal()` is the index of the value in the enumeration, so J`ANUARY.ordinal()` returns 0.

* `compareTo()` compares two values based on their ordinal numbers.

* `name()` returns the name of the value’s constant as a string, e.g. `JANUARY.name()` returns `"JANUARY"`.

* `toString()` has the same behavior as `name()`.


## ADTs in Java

|ADT concept|Ways to do it in java|Examples|
|--|--|--|
|Abstract data type|Single class<br> Interface + class(es)<br> Enum|`String`<br>`List` and `ArrayList`<br> `DayOfWeek`|
|Creator operation|Constructor<br> Static (factory) method<br> Constant|`ArrayList()`<br>`List.of()`<br>`BigInteger.ZERO`|
|Observer operation|Instance method<br>Static method|`List.get()`<br>`Collections.max()`|
|Producer operation|Instance method<br>Static method|`String.trim()`<br>`Collections.unmodifiableList()`|
|Mutator operation|Instance method<br>Static method|`List.add()`<br>`Collections.copy()`|
|Representation|`private` fields||
