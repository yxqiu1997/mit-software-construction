# Reading 24: Sockets & Networking

## Client/server design pattern

In this pattern there are two kinds of processes: clients and servers. A client initiates the communication by connecting to a server. The client sends requests to the server, and the server sends replies back. Finally, the client disconnects. A server might handle connections from many clients concurrently, and clients might also connect to multiple servers.

On the Internet, client and server processes are often running on different machines, connected only by the network, but it doesn’t have to be that way — the server can be a process running on the same machine as the client.


## Sockets and streams

### IP addresses

A network interface is identified by an IP address. IP version 4 addresses are 32-bit numbers written in four 8-bit parts. For example (as of this writing):

* `18.9.22.69` is the IP address of a MIT web server.

* `173.194.193.99` is the address of a Google web server.

* `104.47.42.36` is the address of a Microsoft Outlook email handler.

* `127.0.0.1` is the loopback or localhost address: it always refers to the local machine. Technically, any address whose first octet is `127` is a loopback address, but `127.0.0.1` is standard.

### Hostnames

Hostnames are names that can be translated into IP addresses. A single hostname can map to different IP addresses at different times; and multiple hostnames can map to the same IP address. For example:

* web.mit.edu is the name for MIT’s web server. You can translate this name to an IP address yourself using `dig`, `host`, or `nslookup `on the command line, e.g.:

```shell
$ dig +short web.mit.edu
18.9.22.69
```

* `google.com` is exactly what you think it is. Try using one of the commands above to find `google.com`’s IP address. What do you see?

* `mit-edu.mail.protection.outlook.com` is the name for MIT’s incoming email handler, a spam filtering system hosted by Microsoft.

* `localhost` is a name for `127.0.0.1`. When you want to talk to a server running on your own machine, talk to `localhost`.

Translation from hostnames to IP addresses is the job of the `Domain Name System (DNS)`.

### Port numbers

A single machine might have multiple server applications that clients wish to connect to, so we need a way to direct traffic on the same network interface to different processes.

Network interfaces have multiple ports identified by a 16-bit number. Port 0 is reserved, so port numbers effectively run from 1 to 65535.

A server process binds to a particular port — it is now **listening** on that port. A port can have only one listener at a time, so if some other server process tries to listen to the same port, it will fail.

Clients have to know which port number the server is listening on. There are some well-known ports that are reserved for system-level processes and provide standard ports for certain services. For example:

* Port 22 is the standard SSH port. When you connect to `athena.dialup.mit.edu` using SSH, the software automatically uses port 22.
* Port 25 is the standard email server port.
* Port 80 is the standard web server port. When you connect to the URL `http://web.mit.edu` in your web browser, it connects to `18.9.22.69` on port 80.

When the port is not a standard port, it is specified as part of the address. For example, the URL `http://128.2.39.10:9000` refers to port 9000 on the machine at `128.2.39.10`.

### Network sockets

A socket represents one end of the connection between client and server.

* A **listening socket** is used by a server process to wait for connections from remote clients.

    In Java, use `ServerSocket` to make a listening socket, and use its `accept` method to listen to it. Remember that a port can only have one listener, so this will fail if another thread or process is currently listening at the same port.

* A **connected socket** can send and receive messages to and from the process on the other end of the connection.

    In Java, clients use a `Socket` constructor to establish a socket connection to a server. Servers obtain a connected socket as a `Socket` object returned from `ServerSocket.accept`.

The word “socket” arises by analogy to a hole for a physical plug, like a USB cable. A listening socket is similar to a USB socket waiting for a cable to be plugged into it. A connected socket is like a USB socket with a cable plugged into it, and like a cable with two ends, the connection involves two sockets – one at the client end, and one at the server end.

Be careful with this physical-socket analogy, though, because it’s not quite right. In the real world, a USB socket allows only one cable to be connected to it, so a “listening” USB socket becomes a “connected” USB socket when a cable is physically plugged into it, and stops being available for new connections. But that’s not how network sockets work. Instead, when a new client arrives at a listening socket, a fresh connected socket is created on the server to manage the new connection. The listening socket continues to exist, bound to the same port number and ready to accept another arriving client when the server calls `accept` on it again. This allows a server to have multiple active client connections that originally targeted the same port number.

### Buffers

The data that clients and servers exchange over the network is sent in chunks. These are rarely just byte-sized chunks, although they might be. The sending side (the client sending a request or the server sending a response) typically writes a large chunk (maybe a whole string like “HELLO, WORLD!” or maybe 20 megabytes of video data). The network chops that chunk up into packets, and each packet is routed separately over the network. At the other end, the receiver reassembles the packets together into a stream of bytes.

The result is a bursty kind of data transmission — the data may already be there when you want to read them, or you may have to wait for them to arrive and be reassembled.

When data arrive, they go into a **buffer**, an array in memory that holds the data until you read it.

## Byte streams

The data going into or coming out of a socket is a **stream** of bytes.

In Java, `InputStream` objects represent sources of data flowing into your program. For example:

* Reading from a file on disk with a `File­Input­Stream`
* User input from `System.in`
* Input from a network socket

`OutputStream` objects represent data sinks, places we can write data to. For example:

* `FileOutputStream` for saving to files
* `System.out` for normal output to the user
* `System.err` for error output
* Output to a network socket

### Character streams

The stream of bytes provided by `InputStream` and `OutputStream` is often too low-level to be useful. We may need to interpret the stream of bytes as a stream of Unicode characters, because *Unicode* can represent a wide variety of human languages (not to mention emoji). A `String` is a sequence of Unicode characters, not a sequence of bytes, so if we want to use strings to manipulate the data inside our program, then we need to convert incoming bytes into Unicode, and convert Unicode back to bytes when we write it out.

In Java, `Reader` and `Writer` represent incoming and outgoing streams of Unicode characters. For example:

* `FileReader` and `FileWriter` treat a file as a sequence of characters rather than bytes
* the wrappers `InputStreamReader` and `OutputStreamWriter` adapt a byte stream into a character stream

One of the pitfalls of I/O is making sure that your program is using the right character encoding, which means the way a sequence of bytes represents a sequence of characters. To avoid character-encoding problems, make sure to explicitly specify the character encoding whenever you construct a `Reader` or `Writer` object. The example code in this reading always specifies UTF-8.


## Using network sockets in java

### Multithreaded server code

The server loop is dedicated to a single client, blocking on `readFromClient.readLine()` and repeatedly reading and replying to messages from that one client, until the client disconnects. Only then does the server go back to its `ServerSocket` and `accept` a connection from the next client waiting in line.

If we want to handle multiple clients at once, using blocking I/O, then the server needs a new thread to handle I/O with each new client. While each client-specific thread is working with its own client, another thread (perhaps the main thread) stands ready to `accept` a new connection.

Here is how the multithreaded `EchoServer` works. The connection-accepting loop is run by the main thread:

```java
while (true) {
    // get the next client connection
    Socket socket = serverSocket.accept();

    // handle the client in a new thread, so that the main thread
    // can resume waiting for another client
    new Thread(new Runnable() {
        public void run() {
            handleClient(socket);
        }
    }).start();
}
```

### Closing streams and sockets with try-with-resources

One new bit of Java syntax is particularly useful for working with streams and sockets: the try-with-resources statement. This statement automatically calls `close()` on variables declared in its parenthesized preamble:

```java
try (
    // preamble: declare variables initialized to objects that need closing after use
) {
    // body: runs with those variables in scope
} catch(...) {
    // catch clauses: optional, handles exceptions thrown by the preamble or body
} finally {
    // finally clause: optional, runs after the body and any catch clause
}
// no matter how the try statement exits, it automatically calls
// close() on all variables declared in the preamble
```

For example, here is how it can be used to ensure that a client socket connection is closed:

```java
try (
    Socket socket = new Socket(hostname, port);
) {
    // read and write to the socket
} catch (IOException ioe) {
    ioe.printStackTrace();
} // socket.close() is automatically called here
```

The try-with-resources statement is useful for any object that should be closed after use:

* byte streams: `InputStream`, `OutputStream`
* character streams: `Reader`, `Writer`
* files: `FileInputStream`, `FileOutputStream`, `FileReader`, `FileWriter`
* sockets: `Socket`, `ServerSocket`


## Wire protocols

A **protocol** is a set of messages that can be exchanged by two communicating parties. A **wire protocol** in particular is a set of messages represented as byte sequences, like `hello world` and `bye` (assuming we’ve agreed on a way to encode those characters into bytes).

### Telnet client

`telnet` is a utility that allows you to make a direct network connection to a listening server and communicate with it via a terminal interface. Windows, Linux, and Mac OS X can all run `telnet`, although more recent operating systems no longer have it installed by default.

### HTTP

Hypertext Transfer Protocol (HTTP) is the language of the World Wide Web. We already know that port 80 is the well-known port for speaking HTTP to web servers, so let’s talk to one on the command line.

```shell
$ telnet www.eecs.mit.edu 80
Trying 18.25.4.17...
Connected to www.eecs.mit.edu.
Escape character is '^]'.
GET /↵
<!DOCTYPE html>
... lots of output ...
<title>Homepage | MIT EECS</title>
... lots more output ...
```

The `GET` command gets a web page. The `/` is the path of the page you want on the site. So this command fetches the page at http://www.eecs.mit.edu:80/. Since 80 is the default port for HTTP, this is equivalent to visiting http://www.eecs.mit.edu/ in your web browser. The result is HTML code that your browser renders to display the EECS homepage.

To quit Telnet manually, type the escape character (probably `Ctrl-]`) to bring up the `telnet>` prompt, and type `quit`:

### SMTP

Simple Mail Transfer Protocol (SMTP) is the protocol for sending email (different protocols are used for client programs that retrieve email from your inbox). Because the email system was designed in a time before spam, modern email communication is fraught with traps and heuristics designed to prevent abuse. But we can still try to speak SMTP. Recall that the well-known SMTP port is 25, and MIT’s incoming email handler is `mit-edu.mail.protection.outlook.com`.

### Designing a wire protocol

When designing a wire protocol, apply the same rules of thumb you use for designing the operations of an abstract data type:

* Keep the number of different messages **small**. It’s better to have a few commands and responses that can be combined rather than many complex messages.

* Each message should have a well-defined purpose and **coherent** behavior.

* The set of messages must be **adequate** for clients to make the requests they need to make and for servers to deliver the results.

Just as we demand representation independence from our types, we should aim for **platform-independence** in our protocols. HTTP can be spoken by any web server and any web browser on any operating system. The protocol doesn’t say anything about how web pages are stored on disk, how they are prepared or generated by the server, what algorithms the client will use to render them, etc.

We can also apply the three big ideas in this class:

* **Safe from bugs**

    * The protocol should be easy for clients and servers to generate and parse. Simpler code for reading and writing the protocol (e.g. a parser generated automatically from a grammar, or simple regular expressions with a regular-expression-matching library) will have fewer opportunities for bugs.

    * Consider the ways a broken or malicious client or server could stuff garbage data into the protocol to break the process on the other end.

    Email spam is one example: when we spoke SMTP above, the mail server asked us to say who was sending the email, and there’s nothing in SMTP to prevent us from lying outright. We’ve had to build systems on top of SMTP to try to stop spammers who lie about `From:` addresses.

    Security vulnerabilities are a more serious example. For example, protocols that allow a client to send requests with arbitrary amounts of data require careful handling on the server to avoid running out of buffer space, or worse.

* **Easy to understand**: for example, choosing a text-based protocol means that we can debug communication errors by reading the text of the client/server exchange. It even allows us to speak the protocol “by hand” as we saw above.

* **Ready for change**: for example, HTTP includes the ability to specify a version number, so clients and servers can agree with one another which version of the protocol they will use. If we need to make changes to the protocol in the future, older clients or servers can continue to work by announcing the version they will use.

Serialization is the process of transforming data structures in memory into a format that can be easily stored or transmitted (not the same as serializability from Thread Safety). Rather than invent a new format for serializing your data between clients and servers, use an existing one. For example, JSON (JavaScript Object Notation) is a simple, widely-used format for serializing basic values, arrays, and maps with string keys.
