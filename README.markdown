# ZeroMQ Async

ZeroMQ is a message-oriented socket system that supports many communication styles (request/reply, publish/subscribe, fan-out, &c.) on top of many transport layers with bindings to many languages.
However, ZeroMQ sockets are not thread safe---concurrent usage typically requires explicit locking or dedicated threads and queues.
This library handles all of that for you, taking your ZeroMQ sockets and hiding them behind thread safe core.async channels.

[Quick start](#quick-start) | [Caveats](#caveats) | [Architecture](#architecture) | [Thanks!](#thanks) | [Other Clojure ZMQ libs](#other-clojure-zmq-libraries)

## Quick start

Add to your `project.clj`:

    [com.keminglabs/zmq-async "0.0.1-SNAPSHOT"]
    
Your system should have ZeroMQ 3.2 installed:

    brew install zeromq

or

    apt-get install libzmq3
    
There are two interfaces to this library, an easy one and a simple one.
In both cases, you'll end up with two core.async channels for each ZeroMQ socket: `send` (into which you can write strings or byte arrays) and `recv` (whence byte arrays or vectors of byte arrays, in the case of multipart messages).

The easy interface creates and binds/connects ZeroMQ sockets for you, associating them with the send and receive ports you provide:

```clojure
(require '[zmq-async.core :refer [request-socket! reply-socket! create-context initialize!]]
         '[clojure.core.async :refer [>! <! go chan sliding-buffer close!]])

(let [context (doto (create-context) (initialize!))
      n 3, addr "inproc://ping-pong"
      [s-send s-recv c-send c-recv] (repeatedly 4 #(chan (sliding-buffer 64)))]
  
  (reply-socket! context :bind addr s-send s-recv)
  (request-socket! context :connect addr c-send c-recv)
      
  (go (dotimes [_ n]
        (println (String. (<! s-recv)))
        (>! s-send "pong"))
      (close! s-send))

  (go (dotimes [_ n]
        (>! c-send "ping")
        (println (String. (<! c-recv))))
      (close! c-send)))
```

The simple interface accepts a ZeroMQ socket that has already been configured and bound/connected and associates it with the provided send and receive ports:

```clojure
(require '[zmq-async.core :refer [register-socket! create-context initialize!]]
         '[clojure.core.async :refer [>! <! go chan sliding-buffer close!]])
(import '(org.zeromq ZMQ ZContext))

(let [zmq-context (ZContext.)
      context (doto (create-context) (initialize!))
      n 3, addr "inproc://ping-pong" 
      [s-send s-recv c-send c-recv] (repeatedly 4 #(chan (sliding-buffer 10)))
      server-sock (doto (.createSocket zmq-context ZMQ/PAIR)
                    ;;twiddle ZeroMQ socket options here...
                    (.bind addr))
      client-sock (doto (.createSocket zmq-context ZMQ/PAIR)
                    ;;twiddle ZeroMQ socket options here...
                    (.connect addr))]

  (register-socket! context server-sock s-send s-recv)
  (register-socket! context client-sock c-send c-recv)
  
  (go (dotimes [_ n]
        (println (String. (<! s-recv)))
        (>! s-send "pong"))
      (close! s-send))

  (go (dotimes [_ n]
        (>! c-send "ping")
        (println (String. (<! c-recv))))
      (close! c-send)))
```

Take a look at the [jzmq javadocs](http://zeromq.github.io/jzmq/javadocs/) for more info on configurating ZeroMQ sockets.
(Of course, after you've created a ZeroMQ socket and handed it off to the library, you shouldn't read/write against it since the sockets aren't thread safe and doing so may crash your JVM.)

To close a socket, close its associated core.async send channel.

For both interfaces, when you are finished invoke the context's shutdown fn to tidy up after yourself:

```clojure
((:shutdown context))
```

which will close all ZeroMQ sockets and core.async channels associated with the context and stop both threads.

## Caveats

+ The `recv` ports provided to the library should never block on writes, otherwise the async message pump thread will block and no messages will be able to go through that context in either direction.
  This may be enforced in the future with an exception (once core.async provides a mechanism for checking if a port can ever block).
+ The ZeroMQ thread will drop messages on the floor rather than blocking trying to handoff to a socket.


## Architecture

![Architecture Diagram](architecture.png)

All sockets are associated with a context of two threads:

+ One thread manages ZeroMQ sockets and conveys incoming values to the application via a core.async channel (the "ZeroMQ thread")
+ One thread manages core.async channels and writes to a ZeroMQ control socket (the "core.async thread")

Each thread blocks with the appropriate selection construct (`zmq_poll` and `alts!!`, respectively) rather than an explicit polling loop.
Thus, each thread must initially communicate with the other via the other's transport.
The core.async thread notifies the ZeroMQ thread that it needs to do something by writing to an in-process control socket ("the ZeroMQ control socket").
However, since Java objects cannot be serialized over ZeroMQ, the core.async thread communicates "out-of-band" to the ZeroMQ thread via a java.util.concurrent queue (basically just yelling on the ZeroMQ control socket "Unblock yo, I just put something on the queue for you to handle").
The ZeroMQ thread will then take from the queue and:

+ write a value out to a ZeroMQ socket, `[sock-id val]`, where `val` can be a string or byte array.
+ register a new socket, `[:register sock-id sock]`, where `sock-id` is a string and `sock` is a ZeroMQ socket object that is ready to be read from or written to (i.e., it has already been bound or connected).
+ close a socket, `[:close sock-id]`.

The ZeroMQ thread writes `[sock-id val]` to the core.async thread's control channel when it receives value `val` from the socket with identifier `sock-id`.

Sockets are closed when their corresponding core.async send channel is closed.


## Thanks

Thanks to @brandonbloom for the initial architecture idea, @zachallaun for pair programming/debugging, @ztellman for advice on error handling and all of the non-happy code paths, @puredanger for suggestions about naming and daemonizing the Java threads, @richhickey for the suggestions to explicitly handle all blocking combinations in a matrix, require explicit buffering semantics from consumers, and to accept byte buffers instead of just arrays, and @halgari for requesting multiple message pump pairs to avoid large-data reads from blocking potentially high-priority smaller messages.


## Other Clojure ZeroMQ libraries

I looked at several ZeroMQ/Clojure bindings before writing this one: [Zilch](https://github.com/dysinger/zilch), [clj-0MQ](https://github.com/AndreasKostler/clj-0MQ), and [ezmq](https://github.com/tel/ezmq) haven't been updated in the past two years and don't offer much more than a thin layer of Clojure over native Java interop calls.

After I started work on this library, an [official ZeroMQ Clojure binding](https://github.com/zeromq/cljzmq) was released, but it also seems like just a thin layer of Clojure over [jzmq](https://github.com/zeromq/jzmq) (the underlying ZeroMQ Java binding that zmq-async also uses) and doesn't seem to offer any help for using ZeroMQ sockets concurrently.

Finally, this library ships with native Linux 64 and OS X 64 compiled bindings to ZeroMQ 3.2.
As long as you're on x64 Linux or OS X, you don't have to manually compile and install jzmq.
See the [project.clj](project.clj) for the SHA of the jzmq commit compiled into this library.


## TODO (?)

+ Handle ByteBuffers in addition to just strings and byte arrays.
+ Enforce that provided ports never block and/or are read/write only as appropriate.
