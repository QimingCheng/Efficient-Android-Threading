Every Android application should adhere to the multithreaded programming model built-in to the Java language. With multithreading comes improvements to performance and responsiveness that is required for a great user experience, but it is accompanied with increased complexities:

* Handling the concurrent programming model in Java.
* Keeping data consistency in a multithreaded environment.
* Setting up task execution strategies.

## Thread Basics {#_thread_basics}

Software programming is all about instructing the hardware to perform an action—e.g. show images on a monitor, store data on the file system, etc. The instructions are defined by the application code that the CPU processes in an ordered sequence, which is the high level definition of a _thread_. From an application perspective a thread is execution along a code path of Java statements that is executed sequentially. A code path that is sequentially executed on a thread is referred to as a _task_ — a unit of work that coherently executes on one thread. A thread can either execute one or multiple tasks in sequence.

### Execution {#_execution}

A thread in an Android application is represented by `java.lang.Thread`. It is the most basic execution environment in Android, that executes tasks when it starts and terminates when the task is finished or there are no more tasks to execute; the alive time of the thread is determined by the length of the task. `Thread` supports execution of tasks that are implementions of the `java.lang.Runnable` interface. An implementation defines the task in the `run` method:

```java
private class MyTask implements Runnable {
        public void run (){
                int i = 0 ; // Stored on the thread local stack.
        }
}
```

All the local variables in the method calls from within a `run()` method–direct or indirect—will be stored on the local memory stack of the thread. The task’s execution is started by instantiating and starting a `Thread`:

```
Thread myThread = new Thread(new MyTask());
myThread
.
start
();
```

On the operating system level the thread has both an instruction- and a stack pointer. The instruction pointer references the next instruction to be processed and the stack pointer references a private memory area—not available to other threads—where thread local data is stored. Thread local data is typically variable literals that are defined in the Java methods of the application.

A CPU can process instructions from one thread at the time, but a system normally has multiple threads that require processing at the same time, e.g. a system with multiple simultaneously running applications. For the user to perceive that applications can run in parallel the CPU has to share its processing time between the application threads. The sharing of a CPU’s processing time is handled by a _scheduler_. It determines what thread the CPU should process and for how long. There scheduling strategy can be implemented in various ways but it is mainly based on the thread _priority_: a high priority-thread gets the CPU allocation before a low-priority thread, which gives more execution time to high-priority threads. Thread priority in Java can be set between 1 \(lowest\) and 10 \(highest\), but—unless explicitly set—the normal priority is 5:

```
myThread
.
setPriority
(
8
);
```

If, however, the scheduling is only priority-based the low-priority threads may not get enough processing time carry out the job it was intended for—known as _starvation_. Hence, schedulers also take the processing time of the threads into account when changing to a new thread. A thread change is known as _context switch_. A context switch starts by storing the state of the executing thread—so that the execution can be resumed at a later point—whereafter that thread has to wait. The scheduler then restores another waiting thread for processing.

Two concurrently running threads—executed by a single processor—are split into execution intervals as [Figure 2-1](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch02.html#fig_parallelism) shows.

```
Thread
T1
=
new
Thread
(
new
MyTask
());
T1
.
start
();
Thread
T2
=
new
Thread
(
new
MyTask
());
T2
.
start
();
```

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter02/parallelism.png "Parallelism.")

Figure 2-1. Two threads executing on one CPU

Every scheduling point includes a context switch, where the operating system has to use the CPU to carry out the switch. One such context switch is noted as C in the figure.

### Single-Threaded Application {#_single_threaded_application}

Each application has at least one thread that defines the code path of execution. If no more threads are created, all of the code will be processed along the same code path, and an instruction has to wait for all preceding intructions to finish before it can be processed.

The single-threaded execution is a simple programming model with deterministic execution order, but most often it is not a sufficient approach as instructions may be postponed for a long time by preceding instructions, although the latter instruction is not depending on the preceeding instructions. For example, a user who presses a button on the device should get immediate visual feedback that the button is pressed, but in a single-threaded environment, the UI event can be delayed until preceding instructions have finished execution, which degrades both performance and responsiveness. To solve this, an application needs to split the execution into multiple code paths, i.e., threads.

### Multi-Threaded Application {#_multi_threaded_application}

With multiple threads, the application code can be split into several code paths, so that operations are perceived to be executing concurrently. If the number of executing threads exceeds the number of processors, true concurrency can not be achieved, but the scheduler switches rapidly between threads to be processed, so that every code path is split into execution intervals that are processed in a sequence.

Multi-threading is a must-have, but the improved performance comes at a cost—increased complexity, increased memory consumption, non-deterministic order of execution—that the application has to manage.

#### Increased Resource Consumption {#_increased_resource_consumption}

Threads come with an overhead in terms of memory and processor usage. Each thread allocates a private memory area that is mainly used to store method local variables and parameters during the execution of the method. The private memory area is allocated when the thread is created and deallocated once the thread terminates, i.e. as long as the thread is active it holds on to system resources—even if it is idle or blocked.

The processor entails overhead for the setup and teardown of threads, and to store and restore threads in context switches. The more threads that executes, the more context switches may occur and deteriorate performance.

#### Increased Complexity {#_increased_complexity}

Analyzing the execution of a single-threaded application is relatively simply, because the order of execution is known. In multi-threaded applications, it is a lot more difficult to analyze how the program is executed and in which order the code is processed. The execution order is indeterministic between threads as it is not known beforehand how the scheduler will allocate execution time to the threads. Hence, multiple threads introduce uncertainty into execution. Not only does this indeterminacy make it much harder to debug errors in the code, but the necessity of coordinating thread poses a risk for introducing new errors.

#### Data Inconsistency {#section_data_inconsistency}

A new set of problems arise in multi-threaded programs when the order of resource access is nondeterministic. If two or more threads uses a shared resource it is not known in which order the threads will reach and process the resource. For example, if two threads `t1` and `t2` tries to modify the member variable `sharedResource`, the access order is indeterminate—it may either be incremented or decremented first.

```
public
class
RaceCondition
{
int
sharedResource
=
0
;
public
void
startTwoThreads
()
{
Thread
t1
=
new
Thread
(
new
Runnable
()
{
@Override
public
void
run
()
{
sharedResource
++;
}
});
t1
.
start
();
Thread
t2
=
new
Thread
(
new
Runnable
()
{
@Override
public
void
run
()
{
sharedResource
--;
}
});
t2
.
start
();
}
}
```

The `sharedResource` is exposed to a _race condition_, that can occur because the ordering of the code execution can differ from every execution; it can not be guaranteed that thread `t1` always comes before thread `t2`. In this case it is not only the ordering that is troublesome, but both the incrementer and decrementer operations are multiple byte code instructions—read, modify and write. Context switches can occur between the byte code instructions, leaving the end result of `sharedResource` dependent on the order of execution; it can be either 0, -1 or 1. The first result occurs if the first thread manages to write the value before the second thread reads it, whereas the two latter results occur if both threads first read the initial value 0 making the last written value determine the end result.

Because context switches can occur while one thread is executing a part of the code that should not be interrupted, it is necessary to create _atomic regions_ of code instructions that always are executed in sequence without interleaving of other threads. If a thread executes in an atomic region other threads will be _blocked_ until no other thread executes in the atomic region. Hence, an atomic region in Java is said to be _mutually exclusive_ as it only allows access to one thread. An atomic region can be created in various ways, see [Intrinsic Lock and Java Monitor](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch02.html#section_synchronized), but the most fundamental synchronization mechanism is the _synchronized_ keyword.

```
synchronized
(
this
)
{
sharedResource
++;
}
```

If every access to the shared resource is synchronized, the data can not be inconsistent in spite of multithreaded access. Many of the threading mechanisms discussed in this book were designed to reduce the risk of such errors.

## Thread Safety {#thread_safety}

Giving multiple threads access to the same object is a great way for threads to communicate quickly—one thread writes another thread reads—but it threatens correctness. Multiple threads can execute the same instance of an object simultaneously, causing concurrent access to the state in shared memory. That imposes a risk of threads either seeing the value of the state before it has been updated, or corrupting the value.

Thread safety is achieved when an object always maintains the correct state when accessed by multiple threads. This is achieved by synchronizing the object’s state, so that the access to the state is controlled. Synchronization should be applied to code that reads or writes any variable that otherwise could be accessed by one thread while being changed by another thread. Such areas of code are called _critical sections_ and must be executed atomically, i.e., by only by one thread at the time. Synchronization is achieved by using a locking mechanism that checks whether there currently is a thread executing in a critical section. If so, all the other threads trying to enter the critical section will block until the thread is finished with executing the critical section.

### Note

If a shared resource is accessible from multiple threads and the state is mutable—i.e. the value can be changed during the lifetime of the resource—every access to the resource needs to be guarded by the same lock.

In short, locks guarantee atomic execution of the regions they lock. Locking mechanisms in Android include:

* Object intrinsic lock

  * The 
    `synchronized`
    keyword

* Explicit locks

  * `java.util.concurrent.locks.ReentrantLock`
  * `java.util.concurrent.locks.ReentrantReadWriteLock`

### Intrinsic Lock and Java Monitor {#section_synchronized}

The `synchronized` keyword operates on the intrinsic lock that is implicitly available in every Java object. The intrinsic lock is mutually exclusive, meaning that thread execution in the critical section is exclusive to one thread. Other threads that try to access a critical region—while being occupied—are blocked and can not continue executing until the lock has been released. The intrinsic lock acts as a _monitor_, see [Figure 2-2](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch02.html#fig_java_monitor). The Java monitor can be modeled with three states:

**Blocked**

Threads that are suspended, while they wait for the monitor to released by another thread.

**Executing**

The one and only thread that owns the monitor, and is currently running the code in the critical section.

**Waiting**

Threads that voluntarely has given up ownership of the monitor before it has reached the end of the critical section. The threads are waiting to be signalled before they can take ownership again.

![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter02/java_monitor.png "Java monitor")

Figure 2-2. Java monitor

Threads transition between the monitor states when it reaches and executes a code block protected by the intrinsic lock:

1. _Enter the monitor_
   . A thread tries to access a section that is guarded by an intrinsic lock. It enters the monitor, but if the lock is already acquired by another thread it is suspended until the
2. _Acquire the lock_
   . If there is no other thread that owns the monitor a blocked thread can take ownership and execute in the critical section. If there are more than one blocked thread the scheduler selects thread to execute. There is no FIFO ordering among the blocked threads, i.e. the first thread to enter the monitor is not necessarily the first one to be selected for execution.
3. _Release the lock and wait_
   . The thread suspends itself through 
   `Object.wait()`
    as it wants to wait for a condition to be fulfilled before it continues to execute.
4. _Acquire the lock after signal_
   . Waiting threads are signalled from another thread through 
   `Object.notify()`
    or 
   `Object.notifyAll()`
    and can take ownership of the monitor again, if selected by the scheduler. However, the waiting threads have no precedence over potentially blocked threads that also want to own the monitor.
5. _Release the lock and exit the monitor_
   . At the end of a critical section the thread exits the monitor and leaves room for another thread to take ownership.

The transitions map to a synchronized code block accordingly:

synchronized \(this\) { 1

```
// Execute code 2

wait\(\);3

// Execute code 4
```

}

### Synchronize Access to Shared Resources {#_synchronize_access_to_shared_resources}

A shared mutable state that can be accessed and altered by multiple threads requires a synchronization strategy to keep the data consistent during the concurrent execution. The strategy involves choosing the right kind of lock for the situation and the setting the scope for the critical section.

#### Using the Intrinsic Lock {#_using_the_intrinsic_lock}

An intrinsic lock can guard a shared mutable state in different ways. The synchronization strategy is about the intrinsic lock itself can differ depending on how the keyword `synchronized` is used:

**Method-level that operates on the intrinsic lock of the enclosing object instance**. +

```
synchronized
void
changeState
()
{
sharedResource
++;
}
```

**Block-level that operates on the intrinsic lock of the enclosing object instance**. +

```
void
changeState
()
{
synchronized
(
this
)
{
sharedResource
++;
}
}
```

**Block-level with other objects intrinsic lock**. +

```
private
final
Object
mLock
=
new
Object
();
void
changeState
()
{
synchronized
(
mLock
)
{
sharedResource
++;
}
}
```

**Method-level that operates on the intrinsic lock of the enclosing class instance**. +

```
synchronized
static
void
changeState
()
{
staticSharedResource
++;
}
```

**Block-level that operates on the intrinsic lock of the enclosing class instance**. +

```
static
void
changeState
()
{
synchronized
(
this
.
getClass
())
{
staticSharedResource
++;
}
}
```

A reference to the `this` object in block-level synchronization uses the same intrinsic lock as method-level synchronization. But by using this syntax, you can control the precise block of code covered by the critical section, and therefore reduce it to cover only the code that actually concerns the state you want to protect. It’s bad practice to create larger atomic areas than necessary, because that may block other threads when not necessary, leading to slower execution across the application.

Synchronizing on other objects’ intrinsic lock enables the use of multiple locks within a class. An application should strive to protect each independent state with a lock of its own. Hence, if a class has more than one independent state, performance is improved by using several locks.

### Warning

The `synchronized` keyword can operate in different intrinsic locks. Keep in mind that synchronization on static methods operates on the intrinsic lock of the class object and not the instance object.

#### Using Explicit Locking Mechanisms {#_using_explicit_locking_mechanisms}

If a more advanced locking strategy is needed, `ReentrantLock` or `ReentrantReadWriteLock` classes can be used instead of the `synchronized` keyword. Critical sections are protected by explicitly locking and unlocking regions in the code:

```
int
sharedResource
;
private
ReentrantLock
mLock
=
new
ReentrantLock
();
public
void
changeState
()
{
mLock
.
lock
();
try
{
sharedResource
++;
}
finally
{
mLock
.
unlock
();
}
}
```

The `synchronized` keyword and `ReentrantLock` have the same semantics: they both block all threads trying to execute a critical section if another thread already has entered that region. This is a defensive strategy that assumes that all concurrent accesses are problematic, but it is not harmful for multiple threads to read a shared variable simultaneously. Hence, `synchronized` and `ReentrantLock` can be overprotective.

The `ReentrantReadWriteLock` lets reading threads execute concurrently but still blocks readers versus writers and writers versus other writers.

```
int
sharedResource
;
private
ReentrantReadWriteLock
mLock
=
new
ReentrantReadWriteLock
();
public
void
changeState
()
{
mLock
.
writeLock
().
lock
();
try
{
sharedResource
++;
}
finally
{
mLock
.
writeLock
().
unlock
();
}
}
public
int
readState
()
{
mLock
.
readLock
().
lock
();
try
{
return
sharedResource
;
}
finally
{
mLock
.
readLock
().
unlock
();
}
}
```

The `ReentrantReadWriteLock` is relatively complex, which leads to a performance penalty as the evaluation to determine whether a thread should be allowed to execute or be blocked requires is longer than with `synchronized` and `ReentrantLock`. Hence, there is a trade off between the performance gain from letting multiple threads read shared resources simultaneously and the performance loss from evaluation complexity. The typically good use case for `ReentrantReadWriteLock` is when there are many reading threads and few writing threads.

### Example: Consumer and Producer {#example_consumer_producer}

A common use case with collaborating threads is the consumer-producer pattern—i.e. one thread that produces data and one thread that consumes the data. The threads collaborate through a list that is shared between the threads. When the list is not full the producer thread adds items to the list, whereas the consumer thread removes items while the list is not empty. If the list is full the producing thread should block, and if the list is empty the consuming thread is blocked.

The `ConsumerProducer` class contains a shared resource `LinkedList` and two methods: `produce()` to add items and `consume` to remove items.

```
public
class
ConsumerProducer
{
private
LinkedList
<
Integer
>
list
=
new
LinkedList
<
Integer
>
();
private
final
int
LIMIT
=
10
;
private
Object
lock
=
new
Object
();
public
void
produce
()
throws
InterruptedException
{
int
value
=
0
;
while
(
true
)
{
synchronized
(
lock
)
{
while
(
list
.
size
()
==
LIMIT
)
{
lock
.
wait
();
}
list
.
add
(
value
++);
lock
.
notify
();
}
}
}
public
void
consume
()
throws
InterruptedException
{
while
(
true
)
{
synchronized
(
lock
)
{
while
(
list
.
size
()
==
0
)
{
lock
.
wait
();
}
int
value
=
list
.
removeFirst
();
lock
.
notify
();
}
}
}
}
```

Both `produce` and `consume` uses the same intrinsic lock for guarding the shared list. Threads that tries to access the list are blocked as long another thread owns the monitor, but producing threads give up execution—i.e. `wait()` if the list is full and consuming threads if the list is empty.

When items are either added or removed from the list, the monitor is signalled—i.e. `notify()` is called—so that waiting threads can execute again. The consumer threads signal producer threads and vice versa.

Two threads that executes the producing and consuming operations.

```
final
ConsumerProducer
cp
=
new
ConsumerProducer
();
Thread
t1
=
new
Thread
(
new
Runnable
()
{
@Override
public
void
run
()
{
try
{
cp
.
produce
();
}
catch
(
InterruptedException
e
)
{
e
.
printStackTrace
();
}
}
}).
start
();
Thread
t2
=
new
Thread
(
new
Runnable
()
{
@Override
public
void
run
()
{
try
{
cp
.
consume
();
}
catch
(
InterruptedException
e
)
{
e
.
printStackTrace
();
}
}
}).
start
();
```

## Task Execution Strategies {#_task_execution_strategies}

To make sure that multiple threads are used properly to create responsive applications, applications should be designed with thread creation and task execution in mind. Two suboptimal designs and extremes are:

One thread for all tasks

All tasks are executed on the same thread. The result is often an unresponsive application that fails to use available processors.

One thread per task

Tasks are always executed on a new thread that is started and terminated for every task. If the tasks are frequently created, with short lifetimes, the overhead of thread creation and teardown can degrade performance.

Although these extremes should be avoided, they both represent variants of sequential and concurrent execution at the extreme.

Sequential execution

Tasks are executed in a sequence that requires one task to finish before the next is processed. Thus, the execution interval of the tasks does not overlap. Advantages of this design are:

* It is inherently thread-safe.
* Can be executed on one thread, which consumes less memory than multiple threads.

Disadvantages include:

* Low throughput.
* The start of each task’s execution depends on previously executed tasks. The start may either be delayed or possibly not executed at all.

  Concurrent execution  
  Task are executed in parallel and interleaved. The advantage is better CPU-utilization, whereas the disadvantage is that the design is not inherently thread-safe, so synchronization may be required.

An effective multi-threaded design utilizes execution environments with both sequential and concurrent execution; the choice depends on the tasks. Isolated and independent tasks can execute concurrently to increase throughput, but tasks that require an ordering or share a common resource without synchronization should be executed sequentially.

### Concurrent Execution Design {#_concurrent_execution_design}

Concurrent execution can be implemented in many ways, so design has to consider how to manage the number of executing threads and their relationships. Basic principles include:

* Favor reuse of threads, instead of always creating new threads, so that the number of times the creation and teardown of resources can be reduced.
* Do not use more threads than required. The more threads that are used, the more memory and processor time is consumed.

## Summary {#_summary_2}

Android applications should be multithreaded to improve performance on both single- and multi-processor platforms. Threads can share execution on a single processor or utilize true concurrency when multiple processors are available. The increased performance comes at a cost of increased complexity and a responsibility to guard resources shared among threads, to preserve data consistency.

