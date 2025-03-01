---
layout: documentation
title: Event-driven programming
doc: Learning gem5
parent: part2
permalink: /documentation/learning_gem5/part2/events/
author: Jason Lowe-Power
---


Event-driven programming
========================

gem5 is an event-driven simulator. In this chapter, we will explore how
to create and schedule events. We will be building from the simple
`HelloObject` from [hello-simobject-chapter](../helloobject).

Creating a simple event callback
--------------------------------

In gem5's event-driven model, each event has a callback function in
which the event is *processed*. Generally, this is a class that inherits
from :cppEvent. However, gem5 provides a wrapper function for creating
simple events.

In the header file for our `HelloObject`, we simply need to declare a
new function that we want to execute every time the event fires
(`processEvent()`). This function must take no parameters and return
nothing.

Next, we add an `Event` instance. In this case, we will use an
`EventFunctionWrapper` which allows us to execute any function.

We also add a `startup()` function that will be explained below.

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventFunctionWrapper event;

  public:
    HelloObject(const HelloObjectParams &p);

    void startup() override;
};
```

Next, we must construct this event in the constructor of `HelloObject`.
The `EventFuntionWrapper` takes two parameters, a function to execute
and a name. The name is usually the name of the SimObject that owns the
event. When printing the name, there will be an automatic
".wrapped\_function\_event" appended to the end of the name.

The first parameter is simply a function that takes no parameters and
has no return value (`std::function<void(void)>`). Usually, this is a
simple lambda function that calls a member function. However, it can be
any function you want. Below, we captute `this` in the lambda (`[this]`)
so we can call member functions of the instance of the class.

```cpp
HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params), event([this]{processEvent();}, name())
{
    DPRINTF(HelloExample, "Created the hello object\n");
}
```

We also must define the implementation of the process function. In this
case, we'll simply print something if we are debugging.

```cpp
void
HelloObject::processEvent()
{
    DPRINTF(HelloExample, "Hello world! Processing the event!\n");
}
```

Scheduling events
-----------------

Finally, for the event to be processed, we first have to *schedule* the
event. For this we use the :cppschedule function. This function
schedules some instance of an `Event` for some time in the future
(event-driven simulation does not allow events to execute in the past).

We will initially schedule the event in the `startup()` function we
added to the `HelloObject` class. The `startup()` function is where
SimObjects are allowed to schedule internal events. It does not get
executed until the simulation begins for the first time (i.e. the
`simulate()` function is called from a Python config file).

```cpp
void
HelloObject::startup()
{
    schedule(event, 100);
}
```

Here, we simply schedule the event to execute at tick 100. Normally, you
would use some offset from `curTick()`, but since we know the startup()
function is called when the time is currently 0, we can use an explicit
tick value.

The output when you run gem5 with the "HelloExample" debug flag is now

    gem5 Simulator System.  http://gem5.org
    gem5 is copyrighted software; use the --copyright option for details.

    gem5 compiled Jan  4 2017 11:01:46
    gem5 started Jan  4 2017 13:41:38
    gem5 executing on chinook, pid 1834
    command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

    Global frequency set at 1000000000000 ticks per second
          0: hello: Created the hello object
    Beginning simulation!
    info: Entering event queue @ 0.  Starting simulation...
        100: hello: Hello world! Processing the event!
    Exiting @ tick 18446744073709551615 because simulate() limit reached

More event scheduling
---------------------

We can also schedule new events within an event process action. For
instance, we are going to add a latency parameter to the `HelloObject`
and a parameter for how many times to fire the event. In the [next
chapter](parameters-chapter) we will make these parameters accessible
from the Python config files.

To the HelloObject class declaration, add a member variable for the
latency and number of times to fire.

```cpp
class HelloObject : public SimObject
{
  private:
    void processEvent();

    EventFunctionWrapper event;

    const Tick latency;

    int timesLeft;

  public:
    HelloObject(const HelloObjectParams &p);

    void startup() override;
};
```

Then, in the constructor add default values for the `latency` and
`timesLeft`.

```cpp
HelloObject::HelloObject(const HelloObjectParams &params) :
    SimObject(params), event([this]{processEvent();}, name()),
    latency(100), timesLeft(10)
{
    DPRINTF(HelloExample, "Created the hello object\n");
}
```

Finally, update `startup()` and `processEvent()`.

```cpp
void
HelloObject::startup()
{
    schedule(event, latency);
}

void
HelloObject::processEvent()
{
    timesLeft--;
    DPRINTF(HelloExample, "Hello world! Processing the event! %d left\n", timesLeft);

    if (timesLeft <= 0) {
        DPRINTF(HelloExample, "Done firing!\n");
    } else {
        schedule(event, curTick() + latency);
    }
}
```

Now, when we run gem5, the event should fire 10 times, and the
simulation will end after 1000 ticks. The output should now look like
the following.

    gem5 Simulator System.  http://gem5.org
    gem5 is copyrighted software; use the --copyright option for details.

    gem5 compiled Jan  4 2017 13:53:35
    gem5 started Jan  4 2017 13:54:11
    gem5 executing on chinook, pid 2326
    command line: build/X86/gem5.opt --debug-flags=Hello configs/learning_gem5/part2/run_hello.py

    Global frequency set at 1000000000000 ticks per second
          0: hello: Created the hello object
    Beginning simulation!
    info: Entering event queue @ 0.  Starting simulation...
        100: hello: Hello world! Processing the event! 9 left
        200: hello: Hello world! Processing the event! 8 left
        300: hello: Hello world! Processing the event! 7 left
        400: hello: Hello world! Processing the event! 6 left
        500: hello: Hello world! Processing the event! 5 left
        600: hello: Hello world! Processing the event! 4 left
        700: hello: Hello world! Processing the event! 3 left
        800: hello: Hello world! Processing the event! 2 left
        900: hello: Hello world! Processing the event! 1 left
       1000: hello: Hello world! Processing the event! 0 left
       1000: hello: Done firing!
    Exiting @ tick 18446744073709551615 because simulate() limit reached

You can find the updated header file
[here](/_pages/static/scripts/part2/events/hello_object.hh) and the
implementation file
[here](/_pages/static/scripts/part2/events/hello_object.cc).
