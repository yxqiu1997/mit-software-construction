# Reading 4: Code Review


## Don't repeat yourself (DRY)

* If you have identical or very similar code in two places, then the fundamental risk is that there’s a bug in both copies, and some maintainer fixes the bug in one place but not the other.


## Comments where needed

* Good comments should make the code easier to understand, safer from bugs (because important assumptions have been documented), and readier for change.

* One kind of crucial comment is a specification, which appears above a method or above a class and documents the behavior of the method or class. In Java, this is conventionally written as a Javadoc comment, meaning that it starts with ``/**`` and includes ``@``-syntax, like ``@param`` and ``@return`` for methods. Here’s an example of a spec:

    ```java
    /**
    * Compute the hailstone sequence.
    * See http://en.wikipedia.org/wiki/Collatz_conjecture#Statement_of_the_problem
    * @param n starting number of sequence; requires n > 0.
    * @return the hailstone sequence starting at n and ending with 1.
    *         For example, hailstone(3)=[3,10,5,16,8,4,2,1].
    */
    public static List<Integer> hailstoneSequence(int n) {
        ...
    }
    ```

* Another crucial comment is one that specifies the provenance or source of a piece of code that was copied or adapted from elsewhere. This is vitally important for practicing software developers.


## Fail fast

* Failing fast means that code should reveal its bugs as early as possible. 


## Avoid magic numbers

* One way to explain a number is with a comment, but a far better way is to declare the number as a named constant with a good, clear name.

    * A number is less readable than a name.

    * Constants may need to change in the future. Using a named constant, instead of repeating the literal number in various places, is more ready for change.

    * Constants may be dependent on other constants. Don't hardcode numbers that you've computed by hand. Use a named constant that visibly computes the relationship in terms of other named constants.


## One purpose for each variable

* Don't reuse parameters, and don't reuse variables. **Variables are not a scarce resource in programming**. **Introduce them freely**, give them good names, and just stop using them when you stop needing them.

* Method parameters, in particular, should generally be left **unmodified**. (This is important for being ready-for-change — in the future, some other part of the method may want to know what the original parameters of the method were, so you shouldn’t blow them away while you’re computing.) It’s a good idea to use ``final`` for method parameters, and **as many other variables as you can**. The ``final`` keyword says that the variable should never be reassigned, and the Java compiler will check it statically. For example:

    ```java
    public static int dayOfYear(final int month, final int dayOfMonth, final int year) {
        ...
    }
    ```


## Use good names

* Good method and variable names are long and self-descriptive. Comments can often be avoided entirely by making the code itself more readable, with better names that describe the methods and variables.

* Java also uses capitalization to distinguish global constants (``public static final``) from variables and local constants. ``ALL_CAPS_WITH_UNDERSCORES`` is used for ``static final`` constants. But the local variables declared inside a method, **including local constants** like ``secondsPerDay`` above, use camelCaseNames.

* Choose short words, and be concise, but avoid abbreviations. 


## Use whitespace to help the reader

* Use consistent indentation.

* Put spaces within code lines to make them easy to read.

* Never use tab characters for indentation, only space characters.


## Don't use global variables

* In Java, a global variable is declared ``public static``. The ``public`` modifier makes it accessible anywhere, and ``static`` means there is a single instance of the variable.

* However, with one additional modifier, ``public static final``, and if furthermore the variable’s type is immutable, then the name becomes a **global constant**. A global constant can be read anywhere in the program but never reassigned or mutated, so the risks go away. Global constants are common and useful.

* In general, convert global variables into parameters and return values, or put them inside objects that you’re calling methods on. 


## Methods should return results, not print them

* In general, only the highest-level parts of a program should interact with the human user or the console. Lower-level parts should take their input as parameters and return their output as results. The sole exception here is debugging output, which can of course be printed to the console. But that kind of output shouldn’t be a part of your design, only a part of how you debug your design.


## Avoid special-case code

* **Actively resist the temptation to handle special cases separately**. If you find yourself writing an ``if`` statement for a special case, stop what you’re doing, and instead think harder about the general-case code, either to confirm that it can actually already handle the special case you’re worrying about (which is often true!), or put in a little more effort to make it handle the special case. If you haven’t even written the general-case code yet, but are just trying to deal with the easy cases first, then you’re doing it in the wrong order. Tackle the general case first.

* Writing broader, general-case code pays off. It results in a shorter method, which is easier to understand and has fewer places for bugs to hide. It is likely to be safer from bugs, because it makes fewer assumptions about the values it is working with. And it is more ready for change, because there are fewer places to update when a change to the method’s behavior is made.

* Some programmers justify handling special cases separately with a belief that it increases the overall performance of the method, by returning a hardcoded answer for a special case right away. For example, when writing a sort algorithm, it can be tempting to check whether the size of the list is 0 or 1 at the very start of the method, since you can then return immediately with no need to sort at all. Or if the size is 2, just do a comparison and a possible swap. These optimizations might indeed make sense — but not until you have evidence that they actually would matter to the speed of the program! If the sort method is almost never called with these special cases, then adding code for them just adds complexity, overhead, and hiding-places for bugs, without any practical improvement in performance. Write a clean, simple, general-case algorithm first, and optimize it later, only if it would actually help.
