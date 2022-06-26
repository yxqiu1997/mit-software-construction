# Reading 2: Callbacks

## Input handling in a graphical user interface

Here is how you create a button:

```java
JButton playButton = new JButton("Play");
```

In order to make your program do something when the button is clicked, you attach a listener to it:

```java
playButton.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent event) {
        playSound();
    } 
});
```

This code creates an instance of an `anonymous class` implementing the `ActionListener` interface, which has exactly one method, `actionPerformed`. When this `ActionListener` instance is given to the button with `addActionListener()`, the button promises to call its `actionPerformed` method every time the user presses the button.

GUI event handling is an instance of the Listener pattern, also known as Publish-Subscribe. In the Listener pattern:

* An event source generates (or publishes) a stream of discrete events, which correspond to state transitions in the source.

* One or more listeners register interest (subscribe) to the stream of events, providing a function to be called when a new event occurs.

In this example:

* the `JButton` is the event source;
* its events are button presses;
* the listener is the anonymous `ActionListener` instance
* the function called when the event happens is `actionPerformed`

An event often includes additional information which might be bundled into an event object (like the `ActionEvent` here) or passed as parameters to the listener function.

When an event occurs, the event source distributes it to all subscribed listeners, by calling their listener methods.


## Callbacks

The `actionPerformed` listener function we saw in the previous section is an example of a general design pattern, a `callback`. A callback is a function that a client provides to a module for the module to call. This is in contrast to normal control flow, in which the client is doing all the calling: calling down into functions that the module provides. With a callback, the client is providing a piece of code for the implementer to call.

Here’s one analogy for thinking about this idea. Normal function calling is like picking up the phone and calling a service, like calling your bank to find out the balance of your account. You give the information that the bank operator needs to look up your account, they read back the account balance to you over the phone, and you hang up. You are the client, and the bank is the module that you’re calling into.

Sometimes the bank is slow to give an answer. You’re put on hold, and you wait until they figure out the answer for you. This is like a function call that blocks until it is ready to return, which we saw when we talked about sockets and message passing.

But sometimes the task may take so long that the bank doesn’t want to put you on hold. Then the bank will ask you for a callback phone number, and they will promise to call you back with the answer. This is analogous to providing a callback function.

The kind of callback used in the Listener pattern is not an answer to a one-time request like your account balance. It’s more like a regular service that the bank is promising to provide, using your callback number as needed to reach you. A better analogy for the Listener pattern is account fraud protection, where the bank calls you on the phone whenever a suspicious transaction occurs on your account.


## First-class functions

Using callbacks requires a programming language in which functions are first-class, which means they can be treated like any other value in the language: passed as parameters, returned as return values, and stored in variables and data structures.

In Java, the only first-class values are primitive values (ints, booleans, characters, etc.) and object references. But objects can carry functions with them, in the form of methods. So it turns out that the way to implement a first-class function, in an object-oriented programming language like Java that doesn’t support first-class functions directly, is to use an object with a method representing the function.

We’ve actually seen this before several times already:

* The `Runnable` object that you pass to a `Thread` constructor is a first-class function, `void run()`.
  
* The `Comparator<T>` object that you pass to a sorted collection (e.g. `SortedSet`) is a first-class function, `int compare(T o1, T o2)`.

* The `ActionListener` object that we passed to the `JButton` above is a first-class function, `void actionPerformed(ActionEvent e)`.
  
* The `HttpHandler` object that we passed to the HttpServer above is a first-class function, `void handle(HttpExchange exchange)`.

This design pattern is called a `functional object`, an object whose purpose is to represent a function. The spec for a functional object in Java is given by an interface, called a Single Abstract Method (SAM) interface because it contains just one method.


## Lambda expressions

For example, instead of writing:

```java
new Thread(new Runnable() {
    public void run() {
        System.out.println("Hello!");
    }
}).start();
```

we can use a lambda expression:

```java
new Thread(() -> {
    System.out.println("Hello!");
}).start();
```

There’s no magic here: Java still doesn’t have first-class functions. So you can only use a lambda when the Java compiler can verify two things:

1. It must be able to determine the type of the functional object the lambda will create. In this example, the compiler sees that the `Thread `constructor takes a `Runnable`, so it will infer that the type must be `Runnable`.

2. This inferred type must be a *functional interface*: an interface with only one (abstract) method. In this example, `Runnable` indeed only has a single method — `void run()` — so the compiler knows the code in the body of the lambda belongs in the body of a `run` method of a new `Runnable` object.
