+++
date = "2016-11-24T21:31:20+08:00"
draft = false
title = "I/O Event Handling Design Patterns"

tags = ["High Performance Server"]
+++

Introduction
---

System I/O can be blocking, or non-blocking synchronous, or non-blocking asynchronous:

 - Blocking I/O means that the calling system does not return control to the caller until the operation is finished
 - a non-blocking synchronous call returns control to the caller immediately

I/O multiplexing mechanisms rely on an event demultiplexor: dispatches I/O events from a limited number of sources to the appropriate read/write event handlers.

There are two non-blocking I/O multiplexing mechanisms: reactor && proactor.

Mechanism
---

1. Reactor 

    In the Reactor pattern, the event demultiplexor waits for events that indicate when a file descriptor or socket is ready for a read or write operation.

    Structure:  

     - Request Event Dispatcher (**synchronous** event demultiplexer): uses an event loop to block on all resources, and dispatches resources from the demultiplexer to the associated event handler
     - Request Event Handler (user handler): performs the actual I/O operation, handles data, , declares renewed interest in I/O events, and returns control to the dispatcher

    Benific:

     - separates application specific code from the reactor implementation
     - allows for simple coarse-grain concurrency while not adding the complexity of multiple threads to the system

    Limitations:

     - more difficult to debug than a procedural pattern due to the inverted flow of control
     - by only calling event handlers synchronously, it limits maximum concurrency, especially on symmetric multiprocessing hardware
     - The scalability of the reactor pattern is limited by event handler and demultiplexer
    
2. Proactor
    
    ![proactor](http://7vij5d.com1.z0.glb.clouddn.com/Proactor.png)    

    In the Proactor pattern, the initiator (event demultiplexor) initiates asynchronous I/O operations. The I/O operation itself is **performed by OS**. 

    A completion handler is called after the asynchronous part has terminated. The proactor pattern can be considered to be an **asynchronous variant** of the synchronous reactor pattern.

    The implementation of this classic asynchronous pattern is based on an asynchronous OS-level API, which is called it "system-level" or "true" async.
    
    Structure:

     - Proactive Initiator : initiates asynchronous I/O operations
     - Completion Event Dispatcher (event demultiplexor): waits for events that indicate the completion of the I/O operation, and forwards those events to the appropriate handlers
     - Completion Event Handler (user handler): handles the data from user defined buffer, starts a new asynchronous operation, and returns control to the dispatcher

Emulated Proactor Implementation
---
    
A solution to the challenge of designing a portable framework for the Proactor and Reactor I/O patterns: transform a Reactor demultiplexor I/O solution to an emulated async I/O by moving read/write operations from event handlers inside the demultiplexor (this is "emulated async" approach).

We simply shifted responsibilities between different actors. By adding functionality to the demultiplexor I/O pattern, we were able to convert the Reactor pattern to a Proactor pattern.

Structure:

 - Dispatcher (**asynchronous** event demultiplexer): performs a non-blocking I/O operation and on completion calls the appropriate handler
 - Event Handler (user handler): handles data from the user-defined buffer, declares new interes, then returns control to the dispatcher

----------
Reference：

> [Comparing Two High-Performance I/O Design Patterns](http://www.artima.com/articles/io_design_patternsP.html)

> [Reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern)

> [Proactor pattern](https://en.wikipedia.org/wiki/Proactor_pattern)