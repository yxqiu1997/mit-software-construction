# Reading 1: Static Checking


## Types

* **Primitive Types**

    * **int**: limited to the range about $\pm 2^{31}$

    * **long**: for large integers up to about $\pm 2^{63}$

    * boolean

    * **double**: for floating-point numbers, which represent a subset of the real numbers

    * char

* **Object Types**

    * String

    * **BigInteger**: represents an integer of arbitrary size

    * ...

    By Java convention, primitive types are lowercases, while object types start with a capital letter.


## Static typing

* Java is a **statically-typed** language. The type of **all variables** are known at compile time (before the program runs), and the compiler can therefore deduce the types of all expressions as well.

* In **dynamically-typed languages** like Python, this kind of checking is deferred until runtime (while the program is running).

* Static typing is a particular kind of **static checking**, which means checking for bugs at compile time.


## Arrays and collections

* Lists only know how to deal with **object types**, **not primitive types**.

* Each primitive type has an equivalent object type: e.g. int and Integer ... Java requires us to use these object type equivalents when we parameterise one type using another. But in other contexts, Java **automatically converts** between int and Integer.


## Methods

* The ``/** ...*/`` comment is a specification of the method, describing the inputs and outputs of the operation.


## Mutating values vs. reassigning variables

* It's good practise to use ``final`` for declaring the parameters of a method and as many local variables as possible.


## Engineering

* Engineers are pessimists:

    * Write a little bit at a time, testing as you go.

    * Document the assumptions that your code depends on.

    * Defend your code against stupidity.
