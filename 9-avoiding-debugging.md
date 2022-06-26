# Reading 9: Avoiding Debugging

## Second defense: localize bug

We already talked about **fail fast**: the earlier a problem is observed (the closer to its cause), the easier it is to fix.

```java
/**
 * @param x  requires x >= 0
 * @return approximation to square root of x
 */
public double sqrt(double x) { ... }
```

```java
/**
 * @param x  requires x >= 0
 * @return approximation to square root of x
 */
public double sqrt(double x) { 
    if (! (x >= 0)) throw new IllegalArgumentException("required x >= 0, but was: " + x);
    ...
}
```


## Assertions

A Java assertion may also include a description expression, which is usually a string, but may also be a primitive type or a reference to an object. The description is printed in an error message when the assertion fails, so it can be used to provide additional details to the programmer about the cause of the failure. The description follows the asserted expression, separated by a colon. For example:

```java
assert x >= 0 : "x is " + x;
```

If `x == -1`, then this assertion fails with the error message:

```java
x is -1
```

along with a stack trace that tells you where the assertion was found in your code and the sequence of calls that brought the program to that point. This information is often enough to get started in finding the bug.

**A serious problem with Java assertions is that assertions are off by default.** So you have to enable assertions explicitly by passing `-ea` (which stands for enable assertions) to the Java virtual machine. 

It’s always a good idea to have assertions turned on when you’re running JUnit tests. You can ensure that assertions are enabled using the following test case:

```java
@Test
public void testAssertionsEnabled() {
    assertThrows(AssertionError.class, () -> { assert false; });
}
```

Note that the Java `assert` statement is a different mechanism from the JUnit methods `assertTrue()`, `assertEquals()`, etc. They all assert a predicate about your code, but are designed for use in different contexts. The `assert` statement should be used in implementation code, for defensive checks inside the implementation. JUnit `assert...()` methods should be used in JUnit tests, to check the result of a test. The `assert` statements don’t run without `-ea`, but the JUnit `assert...()` methods always run.


## What not to assert

Many assertion mechanisms are designed so that assertions are executed only during testing and debugging, and turned off when the program is released to users. Java’s `assert` statement behaves this way. Since assertions may be disabled, the correctness of your program should never depend on whether or not the assertion expressions are executed. In particular, asserted expressions should not have side-effects. For example, if you want to assert that an element removed from a list was actually found in the list, don’t write it like this:


Similarly, if a conditional statement or `switch` does not cover all the possible cases, it is good practice to use a check to block the illegal cases. But don’t use the `assert` statement here, because it can be turned off. Instead, throw an exception in the illegal cases, so that the check will always happen:

```java
switch (vowel) {
  case 'a':
  case 'e':
  case 'i':
  case 'o':
  case 'u': return "A";
  default: throw new AssertionError("must be a vowel, but was: " + vowel);
}
```


## Modularity & encapsulation

You can also localize bugs by better software design.

* **Modularity** means dividing up a system into components, or modules, each of which can be designed, implemented, tested, reasoned about, and reused separately from the rest of the system. The opposite of a modular system is a monolithic system – big and with all of its pieces tangled up and dependent on each other.

* **Encapsulation** means building walls around a module so that the module is responsible for its own internal behavior, and bugs in other parts of the system can’t damage its integrity.

One kind of encapsulation is **access control**, using `public` and `private` to control the visibility and accessibility of your variables and methods. A public variable or method can be accessed by any code (assuming the class containing that variable or method is also public). A private variable or method can only be accessed by code in the same class. **Keeping things private as much as possible, especially for variables,** provides encapsulation, since it limits the code that could inadvertently cause bugs.

**Minimizing the scope of variables** is a powerful practice for bug localization. Here are a few rules that are good for Java:

* Always declare a loop variable in the for-loop initializer.

    ```java
    for (int i = 0; i < 100; ++i) {
    ```

* Declare a variable only when you first need it, and in the innermost curly-brace block that you can. 

* Avoid global variables. 
