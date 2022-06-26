# Reading 2: Basic Java

## Literals

* Java does provide a literal syntax for arrays:

  ```java
  String[] arr = {"a", "b", "c"};
  ```

  But this creates an array, not a ``List``. We can use the utility function ``List.of`` to create a ``List`` from arguments:

  ```java
  List.of("a", "b", "c");
  ```  
  
  A ``List`` created with ``List.of`` comes with an important restriction: it is **immutable**! So we can't add, remove, or replace elements once the list has been created.

* Java also provides ``Set.of`` for creating  **immutable** sets and ``Map.of`` for creating **immutable** maps:

  ```java
  Set.of("a", "b", "c");
  Map.of("apple", 5, "banana", 7);
  ```


## ArrayLists and LinkedLists: creating Lists

* When in doubt, use ``ArrayList``.

* If you want to initialise an ``ArrayList`` or ``LinkedList``, you can give it another collection as an argument:

  ```java
  List<String> firstNames = new ArrayList<>(List.of("Huey", "Dewey", "Louie"));

  List<String> lastNames = new ArrayList<>(Set.of("Duck"));
  ```

  A key difference between ``List.of`` and ``ArrayList`` is mutability. Where ``List.of("Huey", "Dewey", "Louie")``produces an immutable list of three strings, ``new ArrayList<>(List.of("Huey", "Dewey", "Louie"))`` produces a **mutable** list initialized with those strings. 
  
  If you need an initialized mutable list, this is simpler than making multiple calls to add().


## Iteration

* We can’t iterate over ``Maps`` themselves this way, but we can iterate over the keys as we did in Python:

  ```java
  for (String key : turtles.keySet()) {
      System.out.println(key + ": " + turtles.get(key));
  }
  ```

* **Warning**: be careful not to mutate a collection while you’re iterating over it. Adding, removing, or replacing elements disrupts the iteration and can even cause your program to crash.

* Unless we actually need the index value, codes of iterating with indices are verbose and have more places for bugs to hide. **Avoid** it if you can.