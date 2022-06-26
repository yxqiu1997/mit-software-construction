# Reading 26: Abstracting out control flow

## Iterator abstraction

Iterator gives you a sequence of elements from a data structure, without you having to worry about whether the data structure is a set or a hash table or a list or an array — the `Iterator` looks the same no matter what the data structure is.

Using an iterator abstracts away the details:

```java
Iterator<File> iter = files.iterator();
while (iter.hasNext()) {
    File f = iter.next();
    // ...
```

Now the loop will be identical for any type that provides an `Iterator`. There is, in fact, an interface for such types: `Iterable`. Any Iterable can be used with Java’s enhanced for statement — `for (File f : files)` — and under the hood, it uses an iterator.

### Streams

Java has an abstract datatype `Stream<E>` that represents a sequence of elements of type `E`.

Collection types like `List` and `Set` provide a `stream()` operation that returns a `Stream` for the collection, and there’s an `Arrays.stream` function for creating a `Stream` from an array. The `Stream` interface itself provides several static factory creators for streams.

Here are a few examples of streams created in different ways:
            
```java
List<Integer> intList = List.of(1, 4, 9, 16);
Stream<Integer> intStream = intList.stream();

String[] stringArray = new String[] {"a", "b", "c"};
Stream<String> stringStream = Arrays.stream(stringArray);

Stream<String> moreStringsStream = Stream.concat(stringStream,
                                                 Stream.of("d", "e", "f"));

Stream<Integer> numbers0Through99 = IntStream.range(0, 100).boxed();
            // IntStream.range() produces a stream of primitive ints, 
            // just like range() in Python.
            // Then boxed() converts it to a Stream of Integer objects.
```


## Map

**Map** applies a unary function to each element in the stream and returns a new stream containing the results, in the same order:

**map : Stream<‍E> × (E → F) → Stream<‍F>**

```java
List<Integer> list = List.of(1, 4, 9, 16);
Stream<Integer> s = list.stream();
Stream<Double> t = s.map(x -> Math.sqrt(x));
```

We can write the same expression more compactly as:

```java
List.of(1, 4, 9, 16).stream()
    .map(x -> Math.sqrt(x))
```

Another example of a map:

```java
List.of("A", "b", "C").stream()
   .map(s -> s.toLowerCase())
```


## Method references

Java lets us eliminate this level of indirection by referring to the sqrt method directly:

```java
List.of(1, 4, 9, 16).stream()
    .map(Math::sqrt)
```

But let’s pause here for a second, because we’re doing something unusual with functions. The `map` method takes a reference to a function as its argument — not to the result of that function. When we wrote `map(Math::sqrt)`, we didn’t call `sqrt`, like `Math.sqrt(25)` is a call. Instead we referred to the function itself by name. `Math::sqrt` is a reference to an object representing the sqrt function. The type of that object is `Function<T,R>`, which represents unary functions from `T` to `R`. The primary operation of a `Function` is `apply()`, which calls the function with an argument. So `(Math::sqrt).apply(25.0)` returns `5.0`, just like `Math.sqrt(25.0)` would.

But you can also assign that functional object to another variable if you like, and it still behaves like `sqrt`:

```java
Function<Double,Double> mySquareRoot = Math::sqrt;
mySquareRoot.apply(16.0); // returns 4.0
```

### More ways to use map

Map is useful even if you don’t care about the return value of the function. When you have a sequence of mutable objects, for example, you may want to map a mutator operation over them. Because mutator operations typically return `void`, however, Java requires using `forEach` instead of `map`. `forEach` applies the function to each element of the stream, but does not collect their return values into a new stream:

```java
Stream<Thread> threads = threadList.stream();
threadList.stream().forEach(Thread::join);

Stream<Socket> sockets = socketList.stream();
sockets.forEach(Socket::close);
```

Be careful when using functions that have side-effects in `map` and `forEach`. As we will see at the end of this handout, these methods make no guarantees about the order of execution of the function on elements of the stream, because it’s possible for stream operations to be executed in parallel, in separate threads. So the function that you pass to `map` or `forEach` has to be **stateless**, meaning that its behavior should not depend on any state that changes over the course of executing the `map` or `forEach`.


## Filter

Our next important sequence operation is **filter**, which tests each element with a unary predicate, `Predicate<T>`, which represents functions from T to boolean.

Elements that satisfy the predicate are kept; those that don’t are removed. A new sequence is returned; filter doesn’t modify its input sequence.

**filter : Stream<‍E> × (E → boolean) → Stream<‍E>**

Examples:

```java
List.of('x', 'y', '2', '3', 'a').stream()
   .filter(Character::isLetter)
// returns the stream ['x', 'y', 'a']

List.of(1, 2, 3, 4).stream()
   .filter(x -> x%2 == 1)
// returns the stream [1, 3]

List.of("abc", "", "d").stream()
   .filter(s -> !s.isEmpty())
// returns the stream ["abc", "d"]
```


## Reduce

Our final operator, **reduce**, combines the elements of the sequence together, using a binary function. In addition to the function and the list, it also takes an *initial value* that initializes the reduction, and that ends up being the return value if the list is empty:

**reduce : Stream<‍E> × E × (E × E → E) → E**

Adding numbers is probably the most straightforward example:

```java
List.of(1,2,3).stream()
    .reduce(0, (x,y) -> x+y)
// computes (((0+1)+2)+3) to produce the integer 6
```

### Initial value

There are three design choices in the reduce operation. First is whether to require an initial value. In Java, the initial value can be omitted, in which case reduce uses the first element of the stream as the initial value of the reduction. But if the stream is empty, then reduce has no value to return, so this version of the `reduce` operation has return type `Optional<E>`.

This makes it easier to use reducers like `max`, which have no well-defined initial value:

```java
List.of(5, 8, 3, 1).stream()
    .reduce(Math::max)
// computes max(max(max(5,8),3),1) and returns an Optional<Integer> value containing 8
```

In Java, because the operator is required to be associative, the implementation of reduce is free to choose any order of combination, including partial combinations from within the sequence. So the reduction

```java
List.of(1,2,3).stream()
   .reduce(0, (x,y) -> x+y)
```

can be computed in any of these orders:

```java
((0+1)+2)+3 = 6
(0+1)+(2+3) = 6
0+((1+2)+3) = 6
0+(1+(2+3)) = 6
```

### Reduction to another type

The third design choice is the return type of the reduce operation. It doesn’t necessarily have to match the type of the sequence elements. For example, we can use reduce to concatenate a sequence of integers (type E) into a string (type F). But because Java leaves the order of combination unspecified, this requires two binary operators with slightly different type signatures:

* an accumulator ⊙ : F × E → F that adds an element from the sequence (of type E) into the growing result (of type F)

* a combiner ⊗ : F × F → F that combines two partial results, each accumulated from part of the sequence, into a growing result

So the most general form of reduce in Java looks like this:

**reduce : Stream<‍E> × F × (F × E → F) × (F × F → F) → F**

Here’s a simple example that concatenates a sequence of integers into a string:

```java
List.of(1,2,3).stream()
   .reduce(
       "",                           // identity value
       (String s, Integer n) -> s+n, // accumulator
       (String s, String t) -> s+t   // combiner
   )
// returns "123"
```


## Benefits of abstracting out control

Map/filter/reduce can often make code shorter and simpler, and allow the programmer to focus on the heart of the computation rather than on the details of loops, branches, and control flow.

By arranging our program in terms of map, filter, and reduce, and in particular using immutable datatypes and pure functions (functions that do not mutate data) as much as possible, we’ve created more opportunities for safe concurrency. Maps and filters using pure functions over immutable datatypes are instantly parallelizable — invocations of the function on different elements of the sequence can be run in different threads, on different processors, even on different machines, and the result will still be the same.

Java’s map/filter/reduce implementation supports this concurrency automatically. We can take any `Stream` and create a parallelized version of it by calling `parallel()`:

```java
Stream<Path> paths = files.parallel().map(File::toPath);
```

Or on a collection, we can call `parallelStream()`.

Subsequent map/filter/reduce operations on the parallel stream may be executed in separate threads created automatically by Java. So this simple change allows the file loading and word splitting to happen in parallel.
