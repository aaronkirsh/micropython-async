# Application of uasyncio to hardware interfaces

This tutorial is intended for users having varying levels of experience with
asyncio and includes a section for complete beginners. It is for use with the
new version of `uasyncio`, currently V3.0.0.

#### WARNING currently this is a work in progress.

This is a work in progress, with some sections being so marked. There will be
typos and maybe errors - please report errors of fact! Most code samples are
now complete scripts which can be cut and pasted at the REPL.

# Contents

 0. [Introduction](./TUTORIAL.md#0-introduction)  
  0.1 [Installing uasyncio on bare metal](./TUTORIAL.md#01-installing-uasyncio-on-bare-metal)  
 1. [Cooperative scheduling](./TUTORIAL.md#1-cooperative-scheduling)  
  1.1 [Modules](./TUTORIAL.md#11-modules)  
 2. [uasyncio](./TUTORIAL.md#2-uasyncio)  
  2.1 [Program structure](./TUTORIAL.md#21-program-structure)  
  2.2 [Coroutines and Tasks](./TUTORIAL.md#22-coroutines-and-tasks)  
   2.2.1 [Queueing a task for scheduling](./TUTORIAL.md#221-queueing-a-task-for-scheduling)  
   2.2.2 [Running a callback function](./TUTORIAL.md#222-running-a-callback-function)  
   2.2.3 [Notes](./TUTORIAL.md#223-notes) Coros as bound methods. Returning values.  
  2.3 [Delays](./TUTORIAL.md#23-delays)  
 3. [Synchronisation](./TUTORIAL.md#3-synchronisation)  
  3.1 [Lock](./TUTORIAL.md#31-lock)  
  3.2 [Event](./TUTORIAL.md#32-event)  
  3.3 [gather](./TUTORIAL.md#33-gather)  
  3.4 [Semaphore](./TUTORIAL.md#34-semaphore)  
   3.4.1 [BoundedSemaphore](./TUTORIAL.md#341-boundedsemaphore)  
  3.5 [Queue](./TUTORIAL.md#35-queue)  
  3.6 [Message](./TUTORIAL.md#36-message)  
  3.7 [Barrier](./TUTORIAL.md#37-barrier)  
 4. [Designing classes for asyncio](./TUTORIAL.md#4-designing-classes-for-asyncio)  
  4.1 [Awaitable classes](./TUTORIAL.md#41-awaitable-classes)  
   4.1.1 [Use in context managers](./TUTORIAL.md#411-use-in-context-managers)  
   4.1.2 [Portable code](./TUTORIAL.md#412-portable-code)  
  4.2 [Asynchronous iterators](./TUTORIAL.md#42-asynchronous-iterators)  
  4.3 [Asynchronous context managers](./TUTORIAL.md#43-asynchronous-context-managers)  
 5. [Exceptions timeouts and cancellation](./TUTORIAL.md#5-exceptions-timeouts-and-cancellation)  
  5.1 [Exceptions](./TUTORIAL.md#51-exceptions)  
   5.1.1 [Global exception handler](./TUTORIAL.md#511-global-exception-handler)  
   5.1.2 [Keyboard interrupts](./TUTORIAL.md#512-keyboard-interrupts)  
  5.2 [Cancellation and Timeouts](./TUTORIAL.md#52-cancellation-and-timeouts)  
   5.2.1 [Task cancellation](./TUTORIAL.md#521-task-cancellation)  
   5.2.2 [Tasks with timeouts](./TUTORIAL.md#522-tasks-with-timeouts)  
   5.2.3 [Cancelling running tasks](./TUTORIAL.md#523-cancelling-running-tasks) A "gotcha".
 6. [Interfacing hardware](./TUTORIAL.md#6-interfacing-hardware)  
  6.1 [Timing issues](./TUTORIAL.md#61-timing-issues)  
  6.2 [Polling hardware with a task](./TUTORIAL.md#62-polling-hardware-with-a-task)  
  6.3 [Using the stream mechanism](./TUTORIAL.md#63-using-the-stream-mechanism)  
   6.3.1 [A UART driver example](./TUTORIAL.md#631-a-uart-driver-example)  
  6.4 [Writing streaming device drivers](./TUTORIAL.md#64-writing-streaming-device-drivers)  
  6.5 [A complete example: aremote.py](./TUTORIAL.md#65-a-complete-example-aremotepy)
  A driver for an IR remote control receiver.  
  6.6 [Driver for HTU21D](./TUTORIAL.md#66-htu21d-environment-sensor) A
  temperature and humidity sensor.  
 7. [Hints and tips](./TUTORIAL.md#7-hints-and-tips)  
  7.1 [Program hangs](./TUTORIAL.md#71-program-hangs)  
  7.2 [uasyncio retains state](./TUTORIAL.md#72-uasyncio-retains-state)  
  7.3 [Garbage Collection](./TUTORIAL.md#73-garbage-collection)  
  7.4 [Testing](./TUTORIAL.md#74-testing)  
  7.5 [A common error](./TUTORIAL.md#75-a-common-error) This can be hard to find.  
  7.6 [Socket programming](./TUTORIAL.md#76-socket-programming)  
   7.6.1 [WiFi issues](./TUTORIAL.md#761-wifi-issues)  
  7.7 [CPython compatibility and the event loop](./TUTORIAL.md#77-cpython-compatibility-and-the-event-loop) Compatibility with CPython 3.5+  
  7.8 [Race conditions](./TUTORIAL.md#78-race-conditions)  
 8. [Notes for beginners](./TUTORIAL.md#8-notes-for-beginners)  
  8.1 [Problem 1: event loops](./TUTORIAL.md#81-problem-1:-event-loops)  
  8.2 [Problem 2: blocking methods](./TUTORIAL.md#8-problem-2:-blocking-methods)  
  8.3 [The uasyncio approach](./TUTORIAL.md#83-the-uasyncio-approach)  
  8.4 [Scheduling in uasyncio](./TUTORIAL.md#84-scheduling-in-uasyncio)  
  8.5 [Why cooperative rather than pre-emptive?](./TUTORIAL.md#85-why-cooperative-rather-than-pre-emptive)  
  8.6 [Communication](./TUTORIAL.md#86-communication)  
  8.7 [Polling](./TUTORIAL.md#87-polling)  

###### [Main README](../README.md)

# 0. Introduction

Most of this document assumes some familiarity with asynchronous programming.
For those new to it an introduction may be found
[in section 7](./TUTORIAL.md#8-notes-for-beginners).

The MicroPython `uasyncio` library comprises a subset of Python's `asyncio`
library. It is designed for use on microcontrollers. As such it has a small RAM
footprint and fast context switching with zero RAM allocation. This document
describes its use with a focus on interfacing hardware devices. The aim is to
design drivers in such a way that the application continues to run while the
driver is awaiting a response from the hardware. The application remains
responsive to events such as user interaction.

Another major application area for asyncio is in network programming: many
guides to this may be found online.

Note that MicroPython is based on Python 3.4 with minimal Python 3.5 additions.
This version of `uasyncio` supports a subset of CPython 3.8 `asyncio`. This
document identifies supported features. Except where stated program samples run
under MicroPython and CPython 3.8.

This tutorial aims to present a consistent programming style compatible with
CPython V3.8 and above.

## 0.1 Installing uasyncio on bare metal

No installation is necessary if a daily build of firmware is installed. The
version may be checked by issuing at the REPL:
```python
import uasyncio
print(uasyncio.__version__)
```
Version 3 will print a version number. Older versions will throw an exception.

###### [Main README](../README.md)

# 1. Cooperative scheduling

The technique of cooperative multi-tasking is widely used in embedded systems.
It offers lower overheads than pre-emptive scheduling and avoids many of the
pitfalls associated with truly asynchronous threads of execution.

###### [Contents](./TUTORIAL.md#contents)

## 1.1 Modules

### Primitives

The directory `primitives` contains a collection of synchronisation primitives
and classes for debouncing switches and pushbuttons, along with a software
retriggerable delay class. Pushbuttons are a generalisation of switches with
logical rather than physical status along with double-clicked and long pressed
events.

These are implemented as a Python package: copy the `primitives` directory tree
to your hardware.

### Demo Programs

The directory `as_demos` contains various demo programs implemented as a Python
package. Copy the directory and its contents to the target hardware.

The first two are the most immediately rewarding as they produce visible
results by accessing Pyboard hardware. With all demos, issue ctrl-d between
runs to soft reset the hardware.

 1. [aledflash.py](./as_demos/aledflash.py) Flashes three Pyboard LEDs
 asynchronously for 10s. Requires any Pyboard.
 2. [apoll.py](./as_demos/apoll.py) A device driver for the Pyboard
 accelerometer. Demonstrates the use of a task to poll a device. Runs for 20s.
 Requires a Pyboard V1.x.
 3. [roundrobin.py](./as_demos/roundrobin.py) Demo of round-robin scheduling.
 Also a benchmark of scheduling performance. Runs for 5s on any target.
 4. [auart.py](./as_demos/auart.py) Demo of streaming I/O via a Pyboard UART.
 Requires a link between X1 and X2.
 5. [auart_hd.py](./as_demos/auart_hd.py) Use of the Pyboard UART to communicate
 with a device using a half-duplex protocol e.g. devices such as those using
 the 'AT' modem command set. Link X1-X4, X2-X3.
 6. [gather.py](./as_demos/gether.py) Use of `gather`. Any target.
 7. [iorw.py](./as_demos/iorw.py) Demo of a read/write device driver using the
 stream I/O mechanism. Requires a Pyboard.

Demos are run using this pattern:
```python
import as_demos.aledflash
```

### Device drivers

These are installed by copying the `as_drivers` directory and contents to the
target. They have their own documentation as follows:

 1. [A driver for GPS modules](./GPS.md) Runs a background task to
 read and decode NMEA sentences, providing constantly updated position, course,
 altitude and time/date information.
 2. [HTU21D](./HTU21D.md) An I2C temperature and humidity sensor. A task
 periodically queries the sensor maintaining constantly available values.
 3. [NEC IR](./NEC_IR) A decoder for NEC IR remote controls. A callback occurs
 whenever a valid signal is received.

###### [Contents](./TUTORIAL.md#contents)

# 2. uasyncio

The asyncio concept is of cooperative multi-tasking based on coroutines
(coros). A coro is similar to a function, but is intended to run concurrently
with other coros. The illusion of concurrency is achieved by periodically
yielding to the scheduler, enabling other coros to be scheduled.

## 2.1 Program structure

Consider the following example:

```python
import uasyncio as asyncio
async def bar():
    count = 0
    while True:
        count += 1
        print(count)
        await asyncio.sleep(1)  # Pause 1s

asyncio.run(bar())
```

Program execution proceeds normally until the call to `asyncio.run(bar())`. At
this point execution is controlled by the scheduler. A line after
`asyncio.run(bar())` would never be executed. The scheduler runs `bar`
because this has been placed on the scheduler's queue by `asyncio.run(bar())`.
In this trivial example there is only one task: `bar`. If there were others,
the scheduler would schedule them in periods when `bar` was paused:

```python
import uasyncio as asyncio
async def bar(x):
    count = 0
    while True:
        count += 1
        print('Instance: {} count: {}'.format(x, count))
        await asyncio.sleep(1)  # Pause 1s

async def main():
    for x in range(3):
        asyncio.create_task(bar(x))
    await asyncio.sleep(10)

asyncio.run(main())
```
In this example, three instances of `bar` run concurrently. The
`asyncio.create_task` method returns immediately but schedules the passed coro
for execution. When `main` sleeps for 10s the `bar` instances are scheduled in
turn, each time they yield to the scheduler with `await asyncio.sleep(1)`.

In this instance `main()` terminates after 10s. This is atypical of embedded
`uasyncio` systems. Normally the application is started at power up by a one
line `main.py` and runs forever.

###### [Contents](./TUTORIAL.md#contents)

## 2.2 Coroutines and Tasks

The fundmental building block of `uasyncio` is a coro. This is defined with
`async def` and usually contains at least one `await` statement. This minimal
example waits 1 second before printing a message:

```python
async def bar():
    await asyncio.sleep(1)
    print('Done')
```

V3 `uasyncio` introduced the concept of a `Task`. A `Task` instance is created
from a coro by means of the `create_task` method, which causes the coro to be
scheduled for execution and returns a `Task` instance. In many cases coros and
tasks are interchangeable: the official docs refer to them as `awaitable`, for
the reason that either may be the target of an `await`. Consider this:

```python
import uasyncio as asyncio
async def bar(t):
    print('Bar started: waiting {}secs'.format(t))
    await asyncio.sleep(t)
    print('Bar done')

async def main():
    await bar(1)  # Pauses here until bar is complete
    task = asyncio.create_task(bar(5))
    await asyncio.sleep(0)  # bar has now started
    print('Got here: bar running')  # Can run code here
    await task  # Now we wait for the bar task to complete
    print('All done')
asyncio.run(main())
```
There is a crucial difference between `create_task` and `await`: the former
is synchronous code and returns immediately, with the passed coro being
converted to a `Task` and queued to run "in the background". By contrast
`await` causes the passed `Task` or coro to run to completion before the next
line executes. Consider these lines of code:

```python
await asyncio.sleep(delay_secs)
await asyncio.sleep(0)
```

The first causes the code to pause for the duration of the delay, with other
tasks being scheduled for the duration. A delay of 0 causes any pending tasks
to be scheduled in round-robin fashion before the following line is run. See
the `roundrobin.py` example.

If a `Task` is run concurrently with `.create_task` it may be cancelled. The
`.create_task` method returns the `Task` instance which may be saved for status
checking or cancellation.

In the following code sample three `Task` instances are created and scheduled
for execution. The "Tasks are running" message is immediately printed. The
three instances of the task `bar` appear to run concurrently: in fact when one
pauses, the scheduler grants execution to the next giving the illusion of
concurrency:

```python
import uasyncio as asyncio
async def bar(x):
    count = 0
    while True:
        count += 1
        print('Instance: {} count: {}'.format(x, count))
        await asyncio.sleep(1)  # Pause 1s

async def main():
    for x in range(3):
        asyncio.create_task(bar(x))
    print('Tasks are running')
    await asyncio.sleep(10)

asyncio.run(main())
```

###### [Contents](./TUTORIAL.md#contents)

### 2.2.1 Queueing a task for scheduling

 * `asyncio.create_task` Arg: the coro to run. The scheduler converts the coro
 to a `Task` and queues the task to run ASAP. Return value: the `Task`
 instance. It returns immediately. The coro arg is specified with function call
 syntax with any required arguments passed.
 * `asyncio.run` Arg: the coro to run. Return value: any value returned by the
 passed coro. The scheduler queues the passed coro to run ASAP. The coro arg is
 specified with function call syntax with any required arguments passed. In the
 current version the `run` call returns when the task terminates. However under
 CPython the `run` call does not terminate.
 * `await`  Arg: the task or coro to run. If a coro is passed it must be
 specified with function call syntax. Starts the task ASAP. The awaiting task
 blocks until the awaited one has run to completion.

The above are compatible with CPython 3.8 or above.

It is possible to `await` a task which has already been started:
```python
import uasyncio as asyncio
async def bar(x):
    count = 0
    for _ in range(5):
        count += 1
        print('Instance: {} count: {}'.format(x, count))
        await asyncio.sleep(1)  # Pause 1s

async def main():
    my_task =  asyncio.create_task(bar(1))
    print('Task is running')
    await asyncio.sleep(2)  # Do something else
    print('Awaiting task')
    await my_task  # If the task has already finished, this returns immediately
    return 10

a = asyncio.run(main())
print(a)
```

###### [Contents](./TUTORIAL.md#contents)

### 2.2.2 Running a callback function

Callbacks should be Python functions designed to complete in a short period of
time. This is because tasks will have no opportunity to run for the
duration. If it is necessary to schedule a callback to run after `t` seconds,
it may be done as follows:
```python
async def schedule(cb, t, *args, **kwargs):
    await asyncio.sleep(t)
    cb(*args, **kwargs)
```
In this example the callback runs after three seconds:
```python
import uasyncio as asyncio

async def schedule(cbk, t, *args, **kwargs):
    await asyncio.sleep(t)
    cbk(*args, **kwargs)

def callback(x, y):
    print('x={} y={}'.format(x, y))

async def bar():
    asyncio.create_task(schedule(callback, 3, 42, 100))
    for count in range(6):
        print(count)
        await asyncio.sleep(1)  # Pause 1s

asyncio.run(bar())
```

###### [Contents](./TUTORIAL.md#contents)

### 2.2.3 Notes

Coros may be bound methods. A coro usually contains at least one `await`
statement, but nothing will break (in MicroPython or CPython 3.8) if it has
none.

Similarly to a function or method, a coro can contain a `return` statement. To
retrieve the returned data issue:

```python
result = await my_task()
```

It is possible to await completion of multiple asynchronously running tasks,
accessing the return value of each. This is done by `uasyncio.gather` which
launches a number of tasks and pauses until the last terminates. It returns a
list containing the data returned by each task:
```python
import uasyncio as asyncio

async def bar(n):
    for count in range(n):
        await asyncio.sleep_ms(200 * n)  # Pause by varying amounts
    print('Instance {} stops with count = {}'.format(n, count))
    return count * count

async def main():
    tasks = (bar(2), bar(3), bar(4))
    print('Waiting for gather...')
    res = await asyncio.gather(*tasks)
    print(res)

asyncio.run(main())
```

###### [Contents](./TUTORIAL.md#contents)

## 2.3 Delays

Where a delay is required in a task there are two options. For longer delays and
those where the duration need not be precise, the following should be used:

```python
async def foo(delay_secs, delay_ms):
    await asyncio.sleep(delay_secs)
    print('Hello')
    await asyncio.sleep_ms(delay_ms)
```

While these delays are in progress the scheduler will schedule other tasks.
This is generally highly desirable, but it does introduce uncertainty in the
timing as the calling routine will only be rescheduled when the one running at
the appropriate time has yielded. The amount of latency depends on the design
of the application, but is likely to be on the order of tens or hundreds of ms;
this is discussed further in [Section 6](./TUTORIAL.md#6-interfacing-hardware).

Very precise delays may be issued by using the `utime` functions `sleep_ms`
and `sleep_us`. These are best suited for short delays as the scheduler will
be unable to schedule other tasks while the delay is in progress.

###### [Contents](./TUTORIAL.md#contents)

# 3 Synchronisation

There is often a need to provide synchronisation between tasks. A common
example is to avoid what are known as "race conditions" where multiple tasks
compete to access a single resource. These are discussed
[in section 7.8](./TUTORIAL.md#78-race-conditions). Another hazard is the
"deadly embrace" where two tasks each wait on the other's completion.

In simple applications communication may be achieved with global flags or bound
variables. A more elegant approach is to use synchronisation primitives.
CPython provides the following classes:  
 * `Lock` - already incorporated in new `uasyncio`.
 * `Event` - already incorporated.
 * `ayncio.gather` - already incorporated.
 * `Semaphore` In this repository.
 * `BoundedSemaphore`. In this repository.
 * `Condition`. In this repository.
 * `Queue`. In this repository.

As the table above indicates, not all are yet officially supported. In the
interim, implementations may be found in the `primitives` directory, along with
the following classes:
 * `Message` An ISR-friendly `Event` with an optional data payload.
 * `Barrier` Based on a Microsoft class, enables multiple coros to synchronise
 in a similar (but not identical) way to `gather`.
 * `Delay_ms` A useful software-retriggerable monostable, akin to a watchdog.
 Calls a user callback if not cancelled or regularly retriggered.
 * `Switch` A debounced switch with open and close user callbacks.
 * `Pushbutton` Debounced pushbutton with callbacks for pressed, released, long
 press or double-press.

To install these priitives, copy the `primitives` directory and contents to the
target. A primitive is loaded by issuing (for example):
```python
from primitives.semaphore import Semaphore, BoundedSemaphore
from primitives.pushbutton import Pushbutton
```
When `uasyncio` acquires an official version (which will be more efficient) the
invocation line alone should be changed:
```python
from uasyncio import Semaphore, BoundedSemaphore
```

Another synchronisation issue arises with producer and consumer tasks. The
producer generates data which the consumer uses. Asyncio provides the `Queue`
object. The producer puts data onto the queue while the consumer waits for its
arrival (with other tasks getting scheduled for the duration). The `Queue`
guarantees that items are removed in the order in which they were received.
Alternatively a `Barrier` instance can be used if the producer must wait
until the consumer is ready to access the data.

The following provides a discussion of the primitives.

###### [Contents](./TUTORIAL.md#contents)

## 3.1 Lock

This describes the use of the official `Lock` primitive.

This guarantees unique access to a shared resource. In the following code
sample a `Lock` instance `lock` has been created and is passed to all tasks
wishing to access the shared resource. Each task attempts to acquire the lock,
pausing execution until it succeeds.

```python
import uasyncio as asyncio
from uasyncio import Lock

async def task(i, lock):
    while 1:
        await lock.acquire()
        print("Acquired lock in task", i)
        await asyncio.sleep(0.5)
        lock.release()

async def main():
    lock = asyncio.Lock()  # The Lock instance
    for n in range(1, 4):
        asyncio.create_task(task(n, lock))
    await asyncio.sleep(10)

asyncio.run(main())  # Run for 10s
```

Methods:

 * `locked` No args. Returns `True` if locked.
 * `release` No args. Releases the lock.
 * `acquire` No args. Coro which pauses until the lock has been acquired. Use
 by executing `await lock.acquire()`.

A task waiting on a lock may be cancelled or may be run subject to a timeout.
The normal way to use a `Lock` is in a context manager:

```python
import uasyncio as asyncio
from uasyncio import Lock

async def task(i, lock):
    while 1:
        async with lock:
            print("Acquired lock in task", i)
            await asyncio.sleep(0.5)
 
async def main():
    lock = asyncio.Lock()  # The Lock instance
    for n in range(1, 4):
        asyncio.create_task(task(n, lock))
    await asyncio.sleep(10)

asyncio.run(main())  # Run for 10s
```

###### [Contents](./TUTORIAL.md#contents)

## 3.2 Event

This describes the use of the official `Event` primitive.

This provides a way for one or more tasks to pause until another flags them to
continue. An `Event` object is instantiated and made accessible to all tasks
using it:

```python
import uasyncio as asyncio
from uasyncio import Event

event = Event()
async def waiter():
    print('Waiting for event')
    await event.wait()  # Pause here until event is set
    print('Waiter got event.')
    event.clear()  # Flag caller and enable re-use of the event

async def main():
    asyncio.create_task(waiter())
    await asyncio.sleep(2)
    print('Setting event')
    event.set()
    await asyncio.sleep(1)
    # Caller can check if event has been cleared
    print('Event is {}'.format('set' if event.is_set() else 'clear'))

asyncio.run(main())
```
Constructor: no args.  
Synchronous Methods:
 * `set` Initiates the event. Currently may not be called in an interrupt
 context.
 * `clear` No args. Clears the event.
 * `is_set` No args. Returns `True` if the event is set.

Asynchronous Method:
 * `wait` Pause until event is set.

Coros waiting on the event issue `await event.wait()` when execution pauses until
another issues `event.set()`.

This presents a problem if `event.set()` is issued in a looping construct; the
code must wait until the event has been accessed by all waiting tasks before
setting it again. In the case where a single task is awaiting the event this
can be achieved by the receiving task clearing the event:

```python
async def eventwait(event):
    await event.wait()
    # Process the data
    event.clear()  # Tell the caller it's ready for more
```

The task raising the event checks that it has been serviced:

```python
async def foo(event):
    while True:
        # Acquire data from somewhere
        while event.is_set():
            await asyncio.sleep(1) # Wait for task to be ready
        # Data is available to the task, so alert it:
        event.set()
```

Where multiple tasks wait on a single event synchronisation can be achieved by
means of an acknowledge event. Each task needs a separate event.

```python
async def eventwait(event, ack_event):
    await event.wait()
    ack_event.set()
```

This is cumbersome. In most cases - even those with a single waiting task - the
Barrier class offers a simpler approach.

**NOTE NOT YET SUPPORTED - see Message class**  
An Event can also provide a means of communication between an interrupt handler
and a task. The handler services the hardware and sets an event which is tested
in slow time by the task.

###### [Contents](./TUTORIAL.md#contents)

## 3.3 gather

This official `uasyncio` asynchronous method causes a number of tasks to run,
pausing until all have either run to completion or been terminated by
cancellation or timeout. It returns a list of the return values of each task.

Its call signature is
```python
res = await asyncio.gather(*tasks, return_exceptions=True)
```
The keyword-only boolean arg `return_exceptions` determines the behaviour in
the event of a cancellation or timeout of tasks. If `False` the `gather`
terminates immediately, raising the relevant exception which should be trapped
by the caller. If `True` the `gather` continues to block until all have either
run to completion or been terminated by cancellation or timeout. In this case
tasks which have been terminated will return the exception object in the list
of return values.

The following script may be used to demonstrate this behaviour

```python
try:
    import uasyncio as asyncio
except ImportError:
    import asyncio

async def barking(n):
    print('Start barking')
    for _ in range(6):
        await asyncio.sleep(1)
    print('Done barking.')
    return 2 * n

async def foo(n):
    print('Start timeout coro foo()')
    while True:
        await asyncio.sleep(1)
        n += 1
    return n

async def bar(n):
    print('Start cancellable bar()')
    while True:
        await asyncio.sleep(1)
        n += 1
    return n

async def do_cancel(task):
    await asyncio.sleep(5)
    print('About to cancel bar')
    task.cancel()

async def main():
    tasks = [asyncio.create_task(bar(70))]
    tasks.append(barking(21))
    tasks.append(asyncio.wait_for(foo(10), 7))
    asyncio.create_task(do_cancel(tasks[0]))
    res = None
    try:
        res = await asyncio.gather(*tasks, return_exceptions=True)
    except asyncio.TimeoutError:  # These only happen if return_exceptions is False
        print('Timeout')  # With the default times, cancellation occurs first
    except asyncio.CancelledError:
        print('Cancelled')
    print('Result: ', res)

asyncio.run(main())
```

###### [Contents](./TUTORIAL.md#contents)

## 3.4 Semaphore

This is currently an unofficial implementation. Its API is as per CPython
asyncio.

A semaphore limits the number of tasks which can access a resource. It can be
used to limit the number of instances of a particular task which can run
concurrently. It performs this using an access counter which is initialised by
the constructor and decremented each time a task acquires the semaphore.

Constructor: Optional arg `value` default 1. Number of permitted concurrent
accesses.

Synchronous method:
 * `release` No args. Increments the access counter.

Asynchronous method:
 * `acquire` No args. If the access counter is greater than 0, decrements it
 and terminates. Otherwise waits for it to become greater than 0 before
 decrementing it and terminating.

The easiest way to use it is with an asynchronous context manager. The
following illustrates tasks accessing a resource one at a time:

```python
import uasyncio as asyncio
from primitives.semaphore import Semaphore

async def foo(n, sema):
    print('foo {} waiting for semaphore'.format(n))
    async with sema:
        print('foo {} got semaphore'.format(n))
        await asyncio.sleep_ms(200)

async def main():
    sema = Semaphore()
    for num in range(3):
        asyncio.create_task(foo(num, sema))
    await asyncio.sleep(2)

asyncio.run(main())
```

There is a difference between a `Semaphore` and a `Lock`. A `Lock` instance is
owned by the coro which locked it: only that coro can release it. A
`Semaphore` can be released by any coro which acquired it.

###### [Contents](./TUTORIAL.md#contents)

### 3.4.1 BoundedSemaphore

This is currently an unofficial implementation. Its API is as per CPython
asyncio.

This works identically to the `Semaphore` class except that if the `release`
method causes the access counter to exceed its initial value, a `ValueError`
is raised.

###### [Contents](./TUTORIAL.md#contents)

## 3.5 Queue

This is currently an unofficial implementation. Its API is as per CPython
asyncio.

The `Queue` class provides a means of synchronising producer and consumer
tasks: the producer puts data items onto the queue with the consumer removing
them. If the queue becomes full, the producer task will block, likewise if
the queue becomes empty the consumer will block.

Constructor: Optional arg `maxsize=0`. If zero, the queue can grow without
limit subject to heap size. If >0 the queue's size will be constrained.

Synchronous methods (immediate return):  
 * `qsize` No arg. Returns the number of items in the queue.
 * `empty` No arg. Returns `True` if the queue is empty.
 * `full` No arg. Returns `True` if the queue is full.
 * `put_nowait` Arg: the object to put on the queue. Raises an exception if the
 queue is full.
 * `get_nowait` No arg. Returns an object from the queue. Raises an exception
 if the queue is empty.

Asynchronous methods:  
 * `put` Arg: the object to put on the queue. If the queue is full, it will
 block until space is available.
 * `get` No arg. Returns an object from the queue. If the queue is empty, it
 will block until an object is put on the queue.

```python
import uasyncio as asyncio
from primitives.queue import Queue

async def slow_process():
    await asyncio.sleep(2)
    return 42

async def produce(queue):
    print('Waiting for slow process.')
    result = await slow_process()
    print('Putting result onto queue')
    await queue.put(result)  # Put result on queue

async def consume(queue):
    print("Running consume()")
    result = await queue.get()  # Blocks until data is ready
    print('Result was {}'.format(result))

async def queue_go(delay):
    queue = Queue()
    asyncio.create_task(consume(queue))
    asyncio.create_task(produce(queue))
    await asyncio.sleep(delay)
    print("Done")

asyncio.run(queue_go(4))
```

###### [Contents](./TUTORIAL.md#contents)

## 3.6 Message

This is an unofficial primitive and has no analog in CPython asyncio.

This is a minor adaptation of the `Event` class. It provides the following:
 * `.set()` has an optional data payload.
 * `.set()` is capable of being called from an interrupt service routine - a
 feature not yet available in the more efficient official `Event`.
 * It is an awaitable class.

The `.set()` method can accept an optional data value of any type. A task
waiting on the `Message` can retrieve it by means of `.value()`. Note that
`.clear()` will set the value to `None`. One use for this is for the task
setting the `Message` to issue `.set(utime.ticks_ms())`. A task waiting on the
`Message` can determine the latency incurred, for example to perform
compensation for this.

Like `Event`, `Message` provides a way for one or more tasks to pause until
another flags them to continue. A `Message` object is instantiated and made
accessible to all tasks using it:

```python
import uasyncio as asyncio
from primitives.message import Message

async def waiter(msg):
    print('Waiting for message')
    await msg
    res = msg.value()
    print('waiter got', res)
    msg.clear()

async def main():
    msg = Message()
    asyncio.create_task(waiter(msg))
    await asyncio.sleep(1)
    msg.set('Hello')  # Optional arg
    await asyncio.sleep(1)

asyncio.run(main())
```

A `Message` can provide a means of communication between an interrupt handler
and a task. The handler services the hardware and issues `.set()` which is
tested in slow time by the task.

###### [Contents](./TUTORIAL.md#contents)

## 3.7 Barrier

I implemented this unofficial primitive before `uasyncio` had support for
`gather`. It is based on a Microsoft primitive. I doubt there is a role for it
in new applications but I will leave it in place to avoid breaking code.

It two uses. Firstly it can cause a task to pause until one or more other tasks
have terminated. For example an application might want to shut down various
peripherals before issuing a sleep period. The task wanting to sleep initiates
several shut down tasks and waits until they have triggered the barrier to
indicate completion.

Secondly it enables multiple coros to rendezvous at a particular point. For
example producer and consumer coros can synchronise at a point where the
producer has data available and the consumer is ready to use it. At that point
in time the `Barrier` can optionally run a callback before releasing the
barrier to allow all waiting coros to continue.

Constructor.  
Mandatory arg:  
 * `participants` The number of coros which will use the barrier.  
Optional args:  
 * `func` Callback to run. Default `None`.  
 * `args` Tuple of args for the callback. Default `()`.

Public synchronous methods:  
 * `busy` No args. Returns `True` if at least one coro is waiting on the
 barrier, or if at least one non-waiting coro has not triggered it.
 * `trigger` No args. The barrier records that the coro has passed the critical
 point. Returns "immediately".

The callback can be a function or a coro. In most applications a function will
be used as this can be guaranteed to run to completion beore the barrier is
released.

Participant coros issue `await my_barrier` whereupon execution pauses until all
other participants are also waiting on it. At this point any callback will run
and then each participant will re-commence execution. See `barrier_test` and
`semaphore_test` in `asyntest.py` for example usage.

A special case of `Barrier` usage is where some coros are allowed to pass the
barrier, registering the fact that they have done so. At least one coro must
wait on the barrier. That coro will pause until all non-waiting coros have
passed the barrier, and all waiting coros have reached it. At that point all
waiting coros will resume. A non-waiting coro issues `barrier.trigger()` to
indicate that is has passed the critical point.

```python
import uasyncio as asyncio
from uasyncio import Event
from primitives.barrier import Barrier

def callback(text):
    print(text)

async def report(num, barrier, event):
    for i in range(5):
        # De-synchronise for demo
        await asyncio.sleep_ms(num * 50)
        print('{} '.format(i), end='')
        await barrier
    event.set()

async def main():
    barrier = Barrier(3, callback, ('Synch',))
    event = Event()
    for num in range(3):
        asyncio.create_task(report(num, barrier, event))
    await event.wait()

asyncio.run(main())
```

multiple instances of `report` print their result and pause until the other
instances are also complete and waiting on `barrier`. At that point the
callback runs. On its completion the tasks resume.

###### [Contents](./TUTORIAL.md#contents)

# 4 Designing classes for asyncio

In the context of device drivers the aim is to ensure nonblocking operation.
The design should ensure that other tasks get scheduled in periods while the
driver is waiting for the hardware. For example a task awaiting data arriving
on a UART or a user pressing a button should allow other tasks to be scheduled
until the event occurs.

###### [Contents](./TUTORIAL.md#contents)

## 4.1 Awaitable classes

A task can pause execution by waiting on an `awaitable` object. There is a
difference between CPython and MicroPython in the way an `awaitable` class is
defined: see [Portable code](./TUTORIAL.md#412-portable-code) for a way to
write a portable class. This section describes a simpler MicroPython specific
solution.

In the following code sample the `__iter__` special method runs for a period.
The calling coro blocks, but other coros continue to run. The key point is that
`__iter__` uses `yield from` to yield execution to another coro, blocking until
it has completed.

```python
import uasyncio as asyncio

class Foo():
    def __iter__(self):
        for n in range(5):
            print('__iter__ called')
            yield from asyncio.sleep(1) # Other tasks get scheduled here
        return 42

async def bar():
    foo = Foo()  # Foo is an awaitable class
    print('waiting for foo')
    res = await foo  # Retrieve result
    print('done', res)

asyncio.run(bar())
```

### 4.1.1 Use in context managers

Awaitable objects can be used in synchronous or asynchronous CM's by providing
the necessary special methods. The syntax is:

```python
with await awaitable as a:  # The 'as' clause is optional
    # code omitted
async with awaitable as a:  # Asynchronous CM (see below)
    # do something
```

To achieve this the `__await__` generator should return `self`. This is passed
to any variable in an `as` clause and also enables the special methods to work.

###### [Contents](./TUTORIAL.md#contents)

### 4.1.2 Portable code

The Python language requires that `__await__` is a generator function. In
MicroPython generators and tasks are identical, so the solution is to use
`yield from task(args)`.

This tutorial aims to offer code portable to CPython 3.8 or above. In CPython
tasks and generators are distinct. CPython tasks have an `__await__` special
method which retrieves a generator. This is portable and was tested under
CPython 3.8:

```python
up = False  # Running under MicroPython?
try:
    import uasyncio as asyncio
    up = True  # Or can use sys.implementation.name
except ImportError:
    import asyncio

async def times_two(n):  # Coro to await
    await asyncio.sleep(1)
    return 2 * n

class Foo():
    def __await__(self):
        res = 1
        for n in range(5):
            print('__await__ called')
            if up:  # MicroPython
                res = yield from times_two(res)
            else:  # CPython
                res = yield from times_two(res).__await__()
        return res

    __iter__ = __await__

async def bar():
    foo = Foo()  # foo is awaitable
    print('waiting for foo')
    res = await foo  # Retrieve value
    print('done', res)

asyncio.run(bar())
```

In `__await__`, `yield from asyncio.sleep(1)` was allowed in CPython 3.6. In
V3.8 it produces a syntax error. It must now be put in the task as in the above
example.

###### [Contents](./TUTORIAL.md#contents)

## 4.2 Asynchronous iterators

These provide a means of returning a finite or infinite sequence of values
and could be used as a means of retrieving successive data items as they arrive
from a read-only device. An asynchronous iterable calls asynchronous code in
its `next` method. The class must conform to the following requirements:

 * It has an `__aiter__` method defined with  `async def`and returning the
 asynchronous iterator.
 * It has an ` __anext__` method which is a task - i.e. defined with
 `async def` and containing at least one `await` statement. To stop
 iteration it must raise a `StopAsyncIteration` exception.

Successive values are retrieved with `async for` as below:

```python
import uasyncio as asyncio
class AsyncIterable:
    def __init__(self):
        self.data = (1, 2, 3, 4, 5)
        self.index = 0

    async def __aiter__(self):
        return self

    async def __anext__(self):
        data = await self.fetch_data()
        if data:
            return data
        else:
            raise StopAsyncIteration

    async def fetch_data(self):
        await asyncio.sleep(0.1)  # Other tasks get to run
        if self.index >= len(self.data):
            return None
        x = self.data[self.index]
        self.index += 1
        return x

async def run():
    ai = AsyncIterable()
    async for x in ai:
        print(x)
asyncio.run(run())
```

###### [Contents](./TUTORIAL.md#contents)

## 4.3 Asynchronous context managers

Classes can be designed to support asynchronous context managers. These are
CM's having enter and exit procedures which are tasks. An example is the `Lock`
class. Such a class has an `__aenter__` task which is logically required to run
asynchronously. To support the asynchronous CM protocol its `__aexit__` method
also must be a task. Such classes are accessed from within a task with the
following syntax:
```python
async def bar(lock):
    async with lock as obj:  # "as" clause is optional, no real point for a lock
        print('In context manager')
```
As with normal context managers an exit method is guaranteed to be called when
the context manager terminates, whether normally or via an exception. To
achieve this the special methods `__aenter__` and `__aexit__` must be
defined, both being tasks waiting on a task or `awaitable` object. This example
comes from the `Lock` class:
```python
    async def __aenter__(self):
        await self.acquire()  # a coro defined with async def
        return self

    async def __aexit__(self, *args):
        self.release()  # A synchronous method
```
If the `async with` has an `as variable` clause the variable receives the
value returned by `__aenter__`. The following is a complete example:
```python
import uasyncio as asyncio

class Foo:
    def __init__(self):
        self.data = 0

    async def acquire(self):
        await asyncio.sleep(1)
        return 42

    async def __aenter__(self):
        print('Waiting for data')
        self.data = await self.acquire()
        return self

    def close(self):
        print('Exit')

    async def __aexit__(self, *args):
        print('Waiting to quit')
        await asyncio.sleep(1)  # Can run asynchronous
        self.close()  # or synchronous methods

async def bar():
    foo = Foo()
    async with foo as f:
        print('In context manager')
        res = f.data
    print('Done', res)

asyncio.run(bar())
```

###### [Contents](./TUTORIAL.md#contents)

# 5 Exceptions timeouts and cancellation

These topics are related: `uasyncio` enables the cancellation of tasks, and the
application of a timeout to a task, by throwing an exception to the task.

## 5.1 Exceptions

Consider a task `foo` created with `asyncio.create_task(foo())`. This task
might `await` other tasks, with potential nesting. If an exception occurs, it
will propagate up the chain until it reaches `foo`. This behaviour is as per
function calls: the exception propagates up the call chain until trapped. If
the exception is not trapped, the `foo` task stops with a traceback. Crucially
other tasks continue to run.

This does not apply to the main task started with `asyncio.run`. If an
exception propagates to that task, the scheduler will stop. This can be
demonstrated as follows:

```python
import uasyncio as asyncio

async def bar():
    await asyncio.sleep(0)
    1/0  # Crash

async def foo():
    await asyncio.sleep(0)
    print('Running bar')
    await bar()
    print('Does not print')  # Because bar() raised an exception

async def main():
    asyncio.create_task(foo())
    for _ in range(5):
        print('Working')  # Carries on after the exception
        await asyncio.sleep(0.5)
    1/0  # Stops the scheduler
    await asyncio.sleep(0)
    print('This never happens')
    await asyncio.sleep(0)

asyncio.run(main())
```
If `main` issued `await foo()` rather than `create_task(foo())` the exception
would propagate to `main`. Being untrapped, the scheduler and hence the script
would stop.

#### Warning

Using `throw` or `close` to throw an exception to a task is unwise. It subverts
`uasyncio` by forcing the task to run, and possibly terminate, when it is still
queued for execution.

### 5.1.1 Global exception handler

During development it is often best if untrapped exceptions stop the program
rather than merely halting a single task. This can be achieved by setting a
global exception handler. This debug aid is not CPython compatible:
```python
import uasyncio as asyncio
import sys

def _handle_exception(loop, context):
    print('Global handler')
    sys.print_exception(context["exception"])
    #loop.stop()
    sys.exit()  # Drastic - loop.stop() does not work when used this way

async def bar():
    await asyncio.sleep(0)
    1/0  # Crash

async def main():
    loop = asyncio.get_event_loop()
    loop.set_exception_handler(_handle_exception)
    asyncio.create_task(bar())
    for _ in range(5):
        print('Working')
        await asyncio.sleep(0.5)

asyncio.run(main())
```

### 5.1.2 Keyboard interrupts

There is a "gotcha" illustrated by the following code sample. If allowed to run
to completion it works as expected.

```python
import uasyncio as asyncio
async def foo():
    await asyncio.sleep(3)
    print('About to throw exception.')
    1/0

async def bar():
    try:
        await foo()
    except ZeroDivisionError:
        print('foo was interrupted by zero division')  # Happens
        raise  # Force shutdown to run by propagating to loop.
    except KeyboardInterrupt:
        print('foo was interrupted by ctrl-c')  # NEVER HAPPENS
        raise

async def shutdown():
    print('Shutdown is running.')  # Happens in both cases
    await asyncio.sleep(1)
    print('done')

try:
    asyncio.run(bar())
except ZeroDivisionError:
    asyncio.run(shutdown())
except KeyboardInterrupt:
    print('Keyboard interrupt at loop level.')
    asyncio.run(shutdown())
```

However issuing a keyboard interrupt causes the exception to go to the
outermost scope. This is because `uasyncio.sleep` causes execution to be
transferred to the scheduler. Consequently applications requiring cleanup code
in response to a keyboard interrupt should trap the exception at the outermost
scope.

###### [Contents](./TUTORIAL.md#contents)

## 5.2 Cancellation and Timeouts

Cancellation and timeouts work by throwing an exception to the task. Unless
explicitly trapped this is transparent to the user: the task simply stops when
next scheduled. It is possible to trap the exception, for example to perform
cleanup code, typically in a `finally` clause. The exception thrown to the task
is `uasyncio.CancelledError` in both cancellation and timeout. There is no way
for the task to distinguish between these two cases.

The `uasyncio.CancelledError` can be trapped, but it is wise to re-raise it: if
the task is `await`ed, the exception can be trapped in the outer scope to
determne the reason for the task's ending.

## 5.2.1 Task cancellation

The `Task` class has a `cancel` method. This throws a `CancelledError` to the
task. This works with nested tasks. Usage is as follows:
```python
import uasyncio as asyncio
async def printit():
    print('Got here')
    await asyncio.sleep(1)

async def foo():
    while True:
        await printit()
        print('In foo')

async def bar():
    foo_task = asyncio.create_task(foo())  # Create task from task
    await asyncio.sleep(4)  # Show it running
    foo_task.cancel()
    await asyncio.sleep(0)
    print('foo is now cancelled.')
    await asyncio.sleep(4)  # Proof!

asyncio.run(bar())
```
The exception may be trapped as follows
```python
import uasyncio as asyncio
async def printit():
    print('Got here')
    await asyncio.sleep(1)

async def foo():
    try:
        while True:
            await printit()
    except asyncio.CancelledError:
        print('Trapped cancelled error.')
        raise  # Enable check in outer scope
    finally:  # Usual way to do cleanup
        print('Cancelled - finally')

async def bar():
    foo_task = asyncio.create_task(foo())
    await asyncio.sleep(4)
    foo_task.cancel()
    await asyncio.sleep(0)
    print('Task is now cancelled')
asyncio.run(bar())
```

###### [Contents](./TUTORIAL.md#contents)

## 5.2.2 Tasks with timeouts

Timeouts are implemented by means of `uasyncio` methods `.wait_for()` and
`.wait_for_ms()`. These take as arguments a task and a timeout in seconds or ms
respectively. If the timeout expires a `uasyncio.CancelledError` is thrown to
the task, while the caller receives a `TimeoutError`. Trapping the exception in
the task is optional. The caller must trap the `TimeoutError` otherwise the
exception will interrupt program execution.

```python
import uasyncio as asyncio

async def forever():
    try:
        print('Starting')
        while True:
            await asyncio.sleep_ms(300)
            print('Got here')
    except asyncio.CancelledError:  # Task sees CancelledError
        print('Trapped cancelled error.')
        raise
    finally:  # Usual way to do cleanup
        print('forever timed out')

async def foo():
    try:
        await asyncio.wait_for(forever(), 3)
    except asyncio.TimeoutError:  # Mandatory error trapping
        print('foo got timeout')  # Caller sees TimeoutError
    await asyncio.sleep(2)

asyncio.run(foo())
```

## 5.2.3 Cancelling running tasks

This useful technique can provoke counter intuitive behaviour. Consider a task
`foo` created using `create_task`. Then tasks `bar`, `cancel_me` (and possibly
others) are created with code like:
```python
async def bar():
    await foo
    # more code
```
All will pause waiting for `foo` to terminate. If any one of the waiting tasks
is cancelled, the cancellation will propagate to `foo`. This would be expected
behaviour if `foo` were a coro. The fact that it is a running task means that
the cancellation impacts the tasks waiting on it; it actually causes their
cancellation. Again, if `foo` were a coro and a task or coro was waiting on it,
cancelling `foo` would be expected to propagate to the caller. In the context
of running tasks, this may be unwelcome.

The behaviour is "correct": CPython `asyncio` behaves identically. Ref
[this forum thread](https://forum.micropython.org/viewtopic.php?f=2&t=8158).

###### [Contents](./TUTORIAL.md#contents)

# 6 Interfacing hardware

At heart all interfaces between `uasyncio` and external asynchronous events
rely on polling. Hardware requiring a fast response may use an interrupt. But
the interface between the interrupt service routine (ISR) and a user task will
be polled. For example the ISR might trigger an `Event` or set a global flag,
while a task awaiting the outcome polls the object each time it is
scheduled.

Polling may be effected in two ways, explicitly or implicitly. The latter is
performed by using the `stream I/O` mechanism which is a system designed for
stream devices such as UARTs and sockets. At its simplest explicit polling may
consist of code like this:

```python
async def poll_my_device():
    global my_flag  # Set by device ISR
    while True:
        if my_flag:
            my_flag = False
            # service the device
        await asyncio.sleep(0)
```

In place of a global, an instance variable, an `Event` object or an instance of
an awaitable class might be used. Explicit polling is discussed
further [below](./TUTORIAL.md#62-polling-hardware-with-a-task).

Implicit polling consists of designing the driver to behave like a stream I/O
device such as a socket or UART, using `stream I/O`. This polls devices using
Python's `select.poll` system: because the polling is done in C it is faster
and more efficient than explicit polling. The use of `stream I/O` is discussed
[here](./TUTORIAL.md#63-using-the-stream-mechanism).

Owing to its efficiency implicit polling benefits most fast I/O device drivers:
streaming drivers can be written for many devices not normally considered as
streaming devices [section 6.4](./TUTORIAL.md#64-writing-streaming-device-drivers).

###### [Contents](./TUTORIAL.md#contents)

## 6.1 Timing issues

Both explicit and implicit polling are currently based on round-robin
scheduling. Assume I/O is operating concurrently with N user tasks each of
which yields with a zero delay. When I/O has been serviced it will next be
polled once all user tasks have been scheduled. The implied latency needs to be
considered in the design. I/O channels may require buffering, with an ISR
servicing the hardware in real time from buffers and tasks filling or
emptying the buffers in slower time.

The possibility of overrun also needs to be considered: this is the case where
something being polled by a task occurs more than once before the task is
actually scheduled.

Another timing issue is the accuracy of delays. If a task issues

```python
    await asyncio.sleep_ms(t)
    # next line
```

the scheduler guarantees that execution will pause for at least `t`ms. The
actual delay may be greater depending on the system state when `t` expires.
If, at that time, all other tasks are waiting on nonzero delays, the next line
will immediately be scheduled. But if other tasks are pending execution (either
because they issued a zero delay or because their time has also elapsed) they
may be scheduled first. This introduces a timing uncertainty into the `sleep()`
and `sleep_ms()` functions. The worst-case value for this overrun may be
calculated by summing, for every other task, the worst-case execution time
between yielding to the scheduler.

The [fast_io](./FASTPOLL.md) version of `uasyncio` in this repo provides a way
to ensure that stream I/O is polled on every iteration of the scheduler. It is
hoped that official `uasyncio` will adopt code to this effect in due course.

###### [Contents](./TUTORIAL.md#contents)

## 6.2 Polling hardware with a task

This is a simple approach, but is most appropriate to hardware which may be
polled at a relatively low rate. This is primarily because polling with a short
(or zero) polling interval may cause the task to consume more processor time
than is desirable.

The example `apoll.py` demonstrates this approach by polling the Pyboard
accelerometer at 100ms intervals. It performs some simple filtering to ignore
noisy samples and prints a message every two seconds if the board is not moved.

Further examples may be found in `aswitch.py` which provides drivers for
switch and pushbutton devices.

An example of a driver for a device capable of reading and writing is shown
below. For ease of testing Pyboard UART 4 emulates the notional device. The
driver implements a `RecordOrientedUart` class, where data is supplied in
variable length records consisting of bytes instances. The object appends a
delimiter before sending and buffers incoming data until the delimiter is
received. This is a demo and is an inefficient way to use a UART compared to
stream I/O.

For the purpose of demonstrating asynchronous transmission we assume the
device being emulated has a means of checking that transmission is complete
and that the application requires that we wait on this. Neither assumption is
true in this example but the code fakes it with `await asyncio.sleep(0.1)`.

Link pins X1 and X2 to run.

```python
import uasyncio as asyncio
from pyb import UART

class RecordOrientedUart():
    DELIMITER = b'\0'
    def __init__(self):
        self.uart = UART(4, 9600)
        self.data = b''

    def __iter__(self):  # Not __await__ issue #2678
        data = b''
        while not data.endswith(self.DELIMITER):
            yield from asyncio.sleep(0) # Necessary because:
            while not self.uart.any():
                yield from asyncio.sleep(0) # timing may mean this is never called
            data = b''.join((data, self.uart.read(self.uart.any())))
        self.data = data

    async def send_record(self, data):
        data = b''.join((data, self.DELIMITER))
        self.uart.write(data)
        await self._send_complete()

    # In a real device driver we would poll the hardware
    # for completion in a loop with await asyncio.sleep(0)
    async def _send_complete(self):
        await asyncio.sleep(0.1)

    def read_record(self):  # Synchronous: await the object before calling
        return self.data[0:-1] # Discard delimiter

async def run():
    foo = RecordOrientedUart()
    rx_data = b''
    await foo.send_record(b'A line of text.')
    for _ in range(20):
        await foo  # Other tasks are scheduled while we wait
        rx_data = foo.read_record()
        print('Got: {}'.format(rx_data))
        await foo.send_record(rx_data)
        rx_data = b''

asyncio.run(run())
```

###### [Contents](./TUTORIAL.md#contents)

## 6.3 Using the stream mechanism

This can be illustrated using a Pyboard UART. The following code sample
demonstrates concurrent I/O on one UART. To run, link Pyboard pins X1 and X2
(UART Txd and Rxd).

```python
import uasyncio as asyncio
from pyb import UART
uart = UART(4, 9600)

async def sender():
    swriter = asyncio.StreamWriter(uart, {})
    while True:
        await swriter.awrite('Hello uart\n')
        await asyncio.sleep(2)

async def receiver():
    sreader = asyncio.StreamReader(uart)
    while True:
        res = await sreader.readline()
        print('Received', res)

async def main():
    rx = asyncio.create_task(receiver())
    tx = asyncio.create_task(sender())
    await asyncio.sleep(10)
    print('Quitting')
    tx.cancel()
    rx.cancel()
    await asyncio.sleep(1)
    print('Done')

asyncio.run(main())
```

The mechanism works because the device driver (written in C) implements the
following methods: `ioctl`, `read`, `readline` and `write`. See
[Writing streaming device drivers](./TUTORIAL.md#64-writing-streaming-device-drivers)
for details on how such drivers may be written in Python.

A UART can receive data at any time. The stream I/O mechanism checks for pending
incoming characters whenever the scheduler has control. When a task is running
an interrupt service routine buffers incoming characters; these will be removed
when the task yields to the scheduler. Consequently UART applications should be
designed such that tasks minimise the time between yielding to the scheduler to
avoid buffer overflows and data loss. This can be ameliorated by using a larger
UART read buffer or a lower baudrate. Alternatively hardware flow control will
provide a solution if the data source supports it.

### 6.3.1 A UART driver example

The program [auart_hd.py](./as_demos/auart_hd.py) illustrates a method of
communicating with a half duplex device such as one responding to the modem
'AT' command set. Half duplex means that the device never sends unsolicited
data: its transmissions are always in response to a command from the master.

The device is emulated, enabling the test to be run on a Pyboard with two wire
links.

The (highly simplified) emulated device responds to any command by sending four
lines of data with a pause between each, to simulate slow processing.

The master sends a command, but does not know in advance how many lines of data
will be returned. It starts a retriggerable timer, which is retriggered each
time a line is received. When the timer times out it is assumed that the device
has completed transmission, and a list of received lines is returned.

The case of device failure is also demonstrated. This is done by omitting the
transmission before awaiting a response. After the timeout an empty list is
returned. See the code comments for more details.

###### [Contents](./TUTORIAL.md#contents)

## 6.4 Writing streaming device drivers

The `stream I/O` mechanism is provided to support I/O to stream devices. Its
typical use is to support streaming I/O devices such as UARTs and sockets. The
mechanism may be employed by drivers of any device which needs to be polled:
the polling is delegated to the scheduler which uses `select` to schedule the
handlers for any devices which are ready. This is more efficient than running
multiple tasks each polling a device, partly because `select` is written in C
but also because the task performing the polling is descheduled until the
`poll` object returns a ready status.

A device driver capable of employing the stream I/O mechanism may support
`StreamReader`, `StreamWriter` instances or both. A readable device must
provide at least one of the following methods. Note that these are synchronous
methods. The `ioctl` method (see below) ensures that they are only called if
data is available. The methods should return as fast as possible with as much
data as is available.

`readline()` Return as many characters as are available up to and including any
newline character. Required if you intend to use `StreamReader.readline()`  
`read(n)` Return as many characters as are available but no more than `n`.
Required to use `StreamReader.read()` or `StreamReader.readexactly()`  

A writeable driver must provide this synchronous method:  
`write` Arg `buf`: the buffer to write. This can be a `memoryview`.  
It should return immediately. The return value is the number of characters
actually written (may well be 1 if the device is slow). The `ioctl` method
ensures that this is only called if the device is ready to accept data.

Note that this has changed relative to `uasyncio` V2. Formerly `write` had
two additional mandatory args. Existing code will fail because `Stream.drain`
calls `write` with a single arg (which can be a `memoryview`).

All devices must provide an `ioctl` method which polls the hardware to
determine its ready status. A typical example for a read/write driver is:

```python
import io
MP_STREAM_POLL_RD = const(1)
MP_STREAM_POLL_WR = const(4)
MP_STREAM_POLL = const(3)
MP_STREAM_ERROR = const(-1)

class MyIO(io.IOBase):
    # Methods omitted
    def ioctl(self, req, arg):  # see ports/stm32/uart.c
        ret = MP_STREAM_ERROR
        if req == MP_STREAM_POLL:
            ret = 0
            if arg & MP_STREAM_POLL_RD:
                if hardware_has_at_least_one_char_to_read:
                    ret |= MP_STREAM_POLL_RD
            if arg & MP_STREAM_POLL_WR:
                if hardware_can_accept_at_least_one_write_character:
                    ret |= MP_STREAM_POLL_WR
        return ret
```

The following is a complete awaitable delay class:

```python
import uasyncio as asyncio
import utime
import io
MP_STREAM_POLL_RD = const(1)
MP_STREAM_POLL = const(3)
MP_STREAM_ERROR = const(-1)

class MillisecTimer(io.IOBase):
    def __init__(self):
        self.end = 0
        self.sreader = asyncio.StreamReader(self)

    def __iter__(self):
        await self.sreader.readline()

    def __call__(self, ms):
        self.end = utime.ticks_add(utime.ticks_ms(), ms)
        return self

    def readline(self):
        return b'\n'

    def ioctl(self, req, arg):
        ret = MP_STREAM_ERROR
        if req == MP_STREAM_POLL:
            ret = 0
            if arg & MP_STREAM_POLL_RD:
                if utime.ticks_diff(utime.ticks_ms(), self.end) >= 0:
                    ret |= MP_STREAM_POLL_RD
        return ret

async def timer_test(n):
    timer = MillisecTimer()
    for x in range(n):
        await timer(100)  # Pause 100ms
        print(x)

asyncio.run(timer_test(20))
```

This currently confers no benefit over `await asyncio.sleep_ms()`, however if
`uasyncio` implements fast I/O scheduling it will be capable of more precise
timing. This is because I/O will be tested on every scheduler call. Currently
it is polled once per complete pass, i.e. when all other pending tasks have run
in round-robin fashion.

It is possible to use I/O scheduling to associate an event with a callback.
This is more efficient than a polling loop because the task doing the polling
is descheduled until `ioctl` returns a ready status. The following runs a
callback when a pin changes state.

```python
import uasyncio as asyncio
import io
MP_STREAM_POLL_RD = const(1)
MP_STREAM_POLL = const(3)
MP_STREAM_ERROR = const(-1)

class PinCall(io.IOBase):
    def __init__(self, pin, *, cb_rise=None, cbr_args=(), cb_fall=None, cbf_args=()):
        self.pin = pin
        self.cb_rise = cb_rise
        self.cbr_args = cbr_args
        self.cb_fall = cb_fall
        self.cbf_args = cbf_args
        self.pinval = pin.value()
        self.sreader = asyncio.StreamReader(self)
        asyncio.create_task(self.run())

    async def run(self):
        while True:
            await self.sreader.read(1)

    def read(self, _):
        v = self.pinval
        if v and self.cb_rise is not None:
            self.cb_rise(*self.cbr_args)
            return b'\n'
        if not v and self.cb_fall is not None:
            self.cb_fall(*self.cbf_args)
        return b'\n'

    def ioctl(self, req, arg):
        ret = MP_STREAM_ERROR
        if req == MP_STREAM_POLL:
            ret = 0
            if arg & MP_STREAM_POLL_RD:
                v = self.pin.value()
                if v != self.pinval:
                    self.pinval = v
                    ret = MP_STREAM_POLL_RD
        return ret
```

Once again latency can be high: if implemented fast I/O scheduling will improve
this.

The demo program [iorw.py](./as_demos/iorw.py) illustrates a complete example.

###### [Contents](./TUTORIAL.md#contents)

## 6.5 A complete example: aremote.py

See [aremote.py](../as_drivers/nec_ir/aremote.py) documented
[here](./NEC_IR.md). This is a complete device driver: a receiver/decoder for
an infra red remote controller. The following notes are salient points
regarding its `asyncio` usage.

A pin interrupt records the time of a state change (in μs) and sets an event,
passing the time when the first state change occurred. A task waits on the
event, yields for the duration of a data burst, then decodes the stored data
before calling a user-specified callback.

Passing the time to the `Event` instance enables the task to compensate for
any `asyncio` latency when setting its delay period.

###### [Contents](./TUTORIAL.md#contents)

## 6.6 HTU21D environment sensor

This chip provides accurate measurements of temperature and humidity. The
driver is documented [here](./HTU21D.md). It has a continuously running
task which updates `temperature` and `humidity` bound variables which may be
accessed "instantly".

The chip takes on the order of 120ms to acquire both data items. The driver
works asynchronously by triggering the acquisition and using
`await asyncio.sleep(t)` prior to reading the data. This allows other tasks to
run while acquisition is in progress.

```python
import as_drivers.htu21d.htu_test
```

###### [Contents](./TUTORIAL.md#contents)

# 7 Hints and tips

## 7.1 Program hangs

Hanging usually occurs because a task has blocked without yielding: this will
hang the entire system. When developing it is useful to have a task which
periodically toggles an onboard LED. This provides confirmation that the
scheduler is running.

## 7.2 uasyncio retains state

If a `uasyncio` application terminates, state is retained. Embedded code seldom
terminates, but in testing it is useful to re-run a script without the need for
a soft reset. This may be done as follows:

```python
import uasyncio as asyncio

async def main():
    await asyncio.sleep(5)  # Dummy test script

def test():
    try:
        asyncio.run(main())
    except KeyboardInterrupt:  # Trapping this is optional
        print('Interrupted')  # or pass
    finally:
        asyncio.new_event_loop()  # Clear retained state
```

###### [Contents](./TUTORIAL.md#contents)

## 7.3 Garbage Collection

You may want to consider running a task which issues:

```python
    gc.collect()
    gc.threshold(gc.mem_free() // 4 + gc.mem_alloc())
```

This assumes `import gc` has been issued. The purpose of this is discussed
[here](http://docs.micropython.org/en/latest/pyboard/reference/constrained.html)
in the section on the heap.

###### [Contents](./TUTORIAL.md#contents)

## 7.4 Testing

It's advisable to test that a device driver yields control when you intend it
to. This can be done by running one or more instances of a dummy task which
runs a loop printing a message, and checking that it runs in the periods when
the driver is blocking:

```python
async def rr(n):
    while True:
        print('Roundrobin ', n)
        await asyncio.sleep(0)
```

As an example of the type of hazard which can occur, in the `RecordOrientedUart`
example above the `__await__` method was originally written as:

```python
    def __await__(self):
        data = b''
        while not data.endswith(self.DELIMITER):
            while not self.uart.any():
                yield from asyncio.sleep(0)
            data = b''.join((data, self.uart.read(self.uart.any())))
        self.data = data
```

In testing this hogged execution until an entire record was received. This was
because `uart.any()` always returned a nonzero quantity. By the time it was
called, characters had been received. The solution was to yield execution in
the outer loop:

```python
    def __await__(self):
        data = b''
        while not data.endswith(self.DELIMITER):
            yield from asyncio.sleep(0) # Necessary because:
            while not self.uart.any():
                yield from asyncio.sleep(0) # timing may mean this is never called
            data = b''.join((data, self.uart.read(self.uart.any())))
        self.data = data
```

It is perhaps worth noting that this error would not have been apparent had
data been sent to the UART at a slow rate rather than via a loopback test.
Welcome to the joys of realtime programming.

###### [Contents](./TUTORIAL.md#contents)

## 7.5 A common error

If a function or method is defined with `async def` and subsequently called as
if it were a regular (synchronous) callable, MicroPython does not issue an
error message. This is [by design](https://github.com/micropython/micropython/issues/3241).
A coro instance is created and discarded, typically leading to a program
silently failing to run correctly:

```python
import uasyncio as asyncio
async def foo():
    await asyncio.sleep(1)
    print('done')

async def main():
    foo()  # Should read: await foo

asyncio.run(main())
```

###### [Contents](./TUTORIAL.md#contents)

## 7.6 Socket programming

There are two basic approaches to socket programming under `uasyncio`. By
default sockets block until a specified read or write operation completes.
`uasyncio` supports blocking sockets by using `select.poll` to prevent them
from blocking the scheduler. In most cases it is simplest to use this
mechanism. Example client and server code may be found in the `client_server`
directory. The `userver` application uses `select.poll` explicitly to poll
the server socket. The client sockets use it implicitly in that the `uasyncio`
stream mechanism employs it.

Note that `socket.getaddrinfo` currently blocks. The time will be minimal in
the example code but if a DNS lookup is required the blocking period could be
substantial.

The second approach to socket programming is to use nonblocking sockets. This
adds complexity but is necessary in some applications, notably where
connectivity is via WiFi (see below).

At the time of writing (March 2019) support for TLS on nonblocking sockets is
under development. Its exact status is unknown (to me).

The use of nonblocking sockets requires some attention to detail. If a
nonblocking read is performed, because of server latency, there is no guarantee
that all (or any) of the requested data is returned. Likewise writes may not
proceed to completion.

Hence asynchronous read and write methods need to iteratively perform the
nonblocking operation until the required data has been read or written. In
practice a timeout is likely to be required to cope with server outages.

A further complication is that the ESP32 port had issues which required rather
unpleasant hacks for error-free operation. I have not tested whether this is
still the case.

The file [sock_nonblock.py](./sock_nonblock.py) illustrates the sort of
techniques required. It is not a working demo, and solutions are likely to be
application dependent.

### 7.6.1 WiFi issues

The `uasyncio` stream mechanism is not good at detecting WiFi outages. I have
found it necessary to use nonblocking sockets to achieve resilient operation
and client reconnection in the presence of outages.

[This doc](https://github.com/peterhinch/micropython-samples/blob/master/resilient/README.md)
describes issues I encountered in WiFi applications which keep sockets open for
long periods, and outlines a solution.

[This repo](https://github.com/peterhinch/micropython-mqtt.git) offers a
resilent asynchronous MQTT client which ensures message integrity over WiFi
outages. [This repo](https://github.com/peterhinch/micropython-iot.git)
provides a simple asynchronous full-duplex serial channel between a wirelessly
connected client and a wired server with guaranteed message delivery.

###### [Contents](./TUTORIAL.md#contents)

## 7.7 CPython compatibility and the event loop

The samples in this tutorial are compatible with CPython 3.8. If you need
compatibility with versions 3.5 or above, the `asyncio.run()` method is absent.
Replace:
```python
asyncio.run(my_task())
```
with:
```python
loop = asyncio.get_event_loop()
loop.run_until_complete(my_task())
```
The `create_task` method is a member of the `event_loop` instance. Replace
```python
asyncio.create_task(my_task())
```
with
```python
loop = asyncio.get_event_loop()
loop.create_task(my_task())
```
Event loop methods are supported in `uasyncio` and in CPython 3.8 but are
deprecated. To quote from the official docs:

Application developers should typically use the high-level asyncio functions,
such as asyncio.run(), and should rarely need to reference the loop object or
call its methods. This section is intended mostly for authors of lower-level
code, libraries, and frameworks, who need finer control over the event loop
behavior. [reference](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_event_loop).

This doc offers better alternatives to `get_event_loop` if you can confine
support to CPython V3.8+.

There is an event loop method `run_forever` which takes no args and causes the
event loop to run. This is supported by `uasyncio`. This has use cases, notably
when all an application's tasks are instantiated in other modules.

## 7.8 Race conditions

These occur when coroutines compete for access to a resource, each using the
resource in a mutually incompatible manner.

This behaviour can be demonstrated by running [the switch test](./primitives/tests/switches.py).
In `test_sw()` coroutines are scheduled by events. If the switch is cycled
rapidly the LED behaviour may seem surprising. This is because each time the
switch is closed a coro is launched to flash the red LED; on each open event
one is launched for the green LED. With rapid cycling a new coro instance will
commence while one is still running against the same LED. This race condition
leads to the LED behaving erratically.

This is a hazard of asynchronous programming. In some situations it is
desirable to launch a new instance on each button press or switch closure, even
if other instances are still incomplete. In other cases it can lead to a race
condition, leading to the need to code an interlock to ensure that the desired
behaviour occurs. The programmer must define the desired behaviour.

In the case of this test program it might be to ignore events while a similar
one is running, or to extend the timer to prolong the LED illumination.
Alternatively a subsequent button press might be required to terminate the
illumination. The "right" behaviour is application dependent.

###### [Contents](./TUTORIAL.md#contents)

# 8 Notes for beginners

These notes are intended for those new to asynchronous code. They start by
outlining the problems which schedulers seek to solve, and give an overview of
the `uasyncio` approach to a solution.

[Section 8.5](./TUTORIAL.md#85-why-cooperative-rather-than-pre-emptive)
discusses the relative merits of `uasyncio` and the `_thread` module and why
you may prefer use cooperative (`uasyncio`) over pre-emptive (`_thread`)
scheduling.

###### [Contents](./TUTORIAL.md#contents)

## 8.1 Problem 1: event loops

A typical firmware application runs continuously and is required to respond to
external events. These might include a voltage change on an ADC, the arrival of
a hard interrupt, a character arriving on a UART, or data being available on a
socket. These events occur asynchronously and the code must be able to respond
regardless of the order in which they occur. Further the application may be
required to perform time-dependent tasks such as flashing LED's.

The obvious way to do this is with an event loop. The following is not
practical code but serves to illustrate the general form of an event loop.

```python
def event_loop():
    led_1_time = 0
    led_2_time = 0
    switch_state = switch.state()  # Current state of a switch
    while True:
        time_now = utime.time()
        if time_now >= led_1_time:  # Flash LED #1
            led1.toggle()
            led_1_time = time_now + led_1_period
        if time_now >= led_2_time:  # Flash LED #2
            led2.toggle()
            led_2_time = time_now + led_2_period
        # Handle LEDs 3 upwards

        if switch.value() != switch_state:
            switch_state = switch.value()
            # do something
        if uart.any():
            # handle UART input
```

This works for simple examples but event loops rapidly become unwieldy as the
number of events increases. They also violate the principles of object oriented
programming by lumping much of the program logic in one place rather than
associating code with the object being controlled. We want to design a class
for an LED capable of flashing which could be put in a module and imported. An
OOP approach to flashing an LED might look like this:

```python
import pyb
class LED_flashable():
    def __init__(self, led_no):
        self.led = pyb.LED(led_no)

    def flash(self, period):
        while True:
            self.led.toggle()
            # somehow wait for period but allow other
            # things to happen at the same time
```

A cooperative scheduler such as `uasyncio` enables classes such as this to be
created.

###### [Contents](./TUTORIAL.md#contents)

## 8.2 Problem 2: blocking methods

Assume you need to read a number of bytes from a socket. If you call
`socket.read(n)` with a default blocking socket it will "block" (i.e. fail to
return) until `n` bytes have been received. During this period the application
will be unresponsive to other events.

With `uasyncio` and a non-blocking socket you can write an asynchronous read
method. The task requiring the data will (necessarily) block until it is
received but during that period other tasks will be scheduled enabling the
application to remain responsive.

## 8.3 The uasyncio approach

The following class provides for an LED which can be turned on and off, and
which can also be made to flash at an arbitrary rate. A `LED_async` instance
has a `run` method which can be considered to run continuously. The LED's
behaviour can be controlled by methods `on()`, `off()` and `flash(secs)`.

```python
import pyb
import uasyncio as asyncio

class LED_async():
    def __init__(self, led_no):
        self.led = pyb.LED(led_no)
        self.rate = 0
        asyncio.create_task(self.run())

    async def run(self):
        while True:
            if self.rate <= 0:
                await asyncio.sleep_ms(200)
            else:
                self.led.toggle()
                await asyncio.sleep_ms(int(500 / self.rate))

    def flash(self, rate):
        self.rate = rate

    def on(self):
        self.led.on()
        self.rate = 0

    def off(self):
        self.led.off()
        self.rate = 0
```

Note that `on()`, `off()` and `flash()` are conventional synchronous methods.
They change the behaviour of the LED but return immediately. The flashing
occurs "in the background". This is explained in detail in the next section.

The class conforms with the OOP principle of keeping the logic associated with
the device within the class. Further, the way `uasyncio` works ensures that
while the LED is flashing the application can respond to other events. The
example below flashes the four Pyboard LED's at different rates while also
responding to the USR button which terminates the program.

```python
import pyb
import uasyncio as asyncio
from led_async import LED_async  # Class as listed above

async def main():
    leds = [LED_async(n) for n in range(1, 4)]
    for n, led in enumerate(leds):
        led.flash(0.7 + n/4)
    sw = pyb.Switch()
    while not sw.value():
        await asyncio.sleep_ms(100)

asyncio.run(main())
```

In contrast to the event loop example the logic associated with the switch is
in a function separate from the LED functionality. Note the code used to start
the scheduler:

```python
asyncio.run(main())  # Execution passes to tasks.
 # It only continues here once main() terminates, when the
 # scheduler has stopped.
```

###### [Contents](./TUTORIAL.md#contents)

## 8.4 Scheduling in uasyncio

Python 3.5 and MicroPython support the notion of an asynchronous function,
known as a task. A task normally includes at least one `await` statement.

```python
async def hello():
    for _ in range(10):
        print('Hello world.')
        await asyncio.sleep(1)
```

This function prints the message ten times at one second intervals. While the
function is paused pending the time delay `asyncio` will schedule other tasks,
providing an illusion of concurrency.

When a task issues `await asyncio.sleep_ms()` or `await asyncio.sleep()` the
current task pauses: it is placed on a queue which is ordered on time due, and
execution passes to the task at the top of the queue. The queue is designed so
that even if the specified sleep is zero other due tasks will run before the
current one is resumed. This is "fair round-robin" scheduling. It is common
practice to issue `await asyncio.sleep(0)` in loops to ensure a task doesn't
hog execution. The following shows a busy-wait loop which waits for another
task to set the global `flag`. Alas it monopolises the CPU preventing other
tasks from running:

```python
async def bad_code():
    global flag
    while not flag:
        pass
    flag = False
    # code omitted
```

The problem here is that while the `flag` is `False` the loop never yields to
the scheduler so no other task will get to run. The correct approach is:

```python
async def good_code():
    global flag
    while not flag:
        await asyncio.sleep(0)
    flag = False
    # code omitted
```

For the same reason it's bad practice to issue delays like `utime.sleep(1)`
because that will lock out other tasks for 1s; use `await asyncio.sleep(1)`.
Note that the delays implied by `uasyncio` methods `sleep` and  `sleep_ms` can
overrun the specified time. This is because while the delay is in progress
other tasks will run. When the delay period completes, execution will not
resume until the running task issues `await` or terminates. A well-behaved task
will always issue `await` at regular intervals. Where a precise delay is
required, especially one below a few ms, it may be necessary to use
`utime.sleep_us(us)`.

###### [Contents](./TUTORIAL.md#contents)

## 8.5 Why cooperative rather than pre-emptive?

The initial reaction of beginners to the idea of cooperative multi-tasking is
often one of disappointment. Surely pre-emptive is better? Why should I have to
explicitly yield control when the Python virtual machine can do it for me?

When it comes to embedded systems the cooperative model has two advantages.
Firstly, it is lightweight. It is possible to have large numbers of tasks
because unlike descheduled threads, paused tasks contain little state.
Secondly it avoids some of the subtle problems associated with pre-emptive
scheduling. In practice cooperative multi-tasking is widely used, notably in
user interface applications.

To make a case for the defence a pre-emptive model has one advantage: if
someone writes

```python
for x in range(1000000):
    # do something time consuming
```

it won't lock out other threads. Under cooperative schedulers the loop must
explicitly yield control every so many iterations e.g. by putting the code in
a task and periodically issuing `await asyncio.sleep(0)`.

Alas this benefit of pre-emption pales into insignificance compared to the
drawbacks. Some of these are covered in the documentation on writing
[interrupt handlers](http://docs.micropython.org/en/latest/reference/isr_rules.html).
In a pre-emptive model every thread can interrupt every other thread, changing
data which might be used in other threads. It is generally much easier to find
and fix a lockup resulting from a task which fails to yield than locating the
sometimes deeply subtle and rarely occurring bugs which can occur in
pre-emptive code.

To put this in simple terms, if you write a MicroPython task, you can be
sure that variables won't suddenly be changed by another task: your task has
complete control until it issues `await asyncio.sleep(0)`.

Bear in mind that interrupt handlers are pre-emptive. This applies to both hard
and soft interrupts, either of which can occur at any point in your code.

An eloquent discussion of the evils of threading may be found
[in threads are bad](https://glyph.twistedmatrix.com/2014/02/unyielding.html).

###### [Contents](./TUTORIAL.md#contents)

## 8.6 Communication

In non-trivial applications tasks need to communicate. Conventional Python
techniques can be employed. These include the use of global variables or
declaring tasks as object methods: these can then share instance variables.
Alternatively a mutable object may be passed as a task argument.

Pre-emptive systems mandate specialist classes to achieve "thread safe"
communications; in a cooperative system these are seldom required.

###### [Contents](./TUTORIAL.md#contents)

## 8.7 Polling

Some hardware devices such as the Pyboard accelerometer don't support
interrupts, and therefore must be polled (i.e. checked periodically). Polling
can also be used in conjunction with interrupt handlers: the interrupt handler
services the hardware and sets a flag. A task polls the flag: if it's set it
handles the data and clears the flag. A better approach is to use an `Event`.

###### [Contents](./TUTORIAL.md#contents)
