Multicore processors are standard in the handheld devices of today and applications should utilize the opportunity to process data in parallel. With parallel data processing comes—in most cases—better performance and responsiveness as more processors can actively take part in running the application. But the other side of the coin is that execution becomes more complex, which can induce indeterministic behavior and timing related bugs that can be hard to reproduce. Every application developer need to manage a multi-threaded programming model and how to steer clear of the problem it induces.

This book focuses on application threading on the Android platform, and how multi-threading cooperates with the Android programming model. It provides an understanding of underlying execution mechanisms and how to utilize the asynchronous techniques efficiently. Multiple-CPU hardware holds many benefits for applications and developers: the application code can be executed on multiple CPUs concurrently to gain performance. Concurrent execution and responsiveness is a key factor for an application to succeed; a snappy application is crucial to preserve the user experience of an otherwise great application. But this book does not stop at making the application fast—it should be fast with minimal implementation effort, a good code design, and minimal risk for unexpected errors \(not too uncommon in the world of concurrency…\). Choosing the right asynchronous mechanism for the right use case is crucial to achieve simple and robust code execution with coherent code design.

But before we immerse ourselves in the world of threading, we will start with an introduction to the Android platform, the application architecture, and its execution. This chapter provides a baseline of knowledge required for an effective discussion of threading in the rest of the book, but a complete information on the Android platform can be found in the [official documentation](https://developer.android.com/)or in most of the numerous Android programming books on the market.

## Android Software Stack {#_android_software_stack}

Applications run on top of a software stack that is based on a Linux kernel, native C/C++ libraries, and a runtime that executes the application code \([Figure 1-1](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#fig_software_stack)\). The elements are:



![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter01/android_software_stack.png "The software stack in Android.")

Figure 1-1. Android Software Stack

###### Applications

Top layer with Java applications, that use the Core Java library and the Android Application Framework classes.

###### Core Java

The core Java libraries used by applications and the Application Framework. It is not a fully compliant Java SE or ME implementation, but a subset of the retired [Apache Harmony](http://harmony.apache.org/) implementation, based on Java 5. It provides the fundamental Java threading mechanisms as the java.lang.Thread -class and java.util.concurrent-package.

###### Application framework

The Android classes that handle the window system, UI toolkit, resources, etc.—basically everything that is required to write an Android application in Java. The framework defines and manages the lifecycles of the Android components and their inter-communication. Furthermore, it defines a set of Android specific asynchronous mechanisms that applications can utilize to simplify the thread management:`HandlerThread`,`AsyncTask`,`IntentService`,`AsyncQueryHandler`and`Loaders`. All these mechanisms will be described in this book.

###### Native libraries

C/C++ libraries that handle graphics, media, database, fonts, OpenGL, etc. Java applications do not interact directly with the native libraries, because the Application framework provides Java wrappers for the native code.

###### Runtime

The Dalvik Virtual Machine, which provides a sandbox for each Java application and executes compiled Android application code, with an internal bytecode representation stored in Dalvik Executable \(_.dex_\) files. Every application runs in its own Dalvik runtime.

###### Linux kernel

Underlying operating system that allows applications to use the hardware functions of the device, e.g., sound, network, camera, etc. It also manages processes and threads. A process is started for every application, and every process holds a Dalvik runtime with a running application. Within the process, multiple threads can execute the application code. The kernel splits the available CPU execution time for processes and their threads through _scheduling_.

## Application Architecture {#_application_architecture}

The cornerstones of an application are theApplication cocmponents:`Activity`,`Service`,`BroadcastReceiver`, and`ContentProvider`.

### APPLICATION {#_application}

The representation of an executing application in Java is the`android.app.Application`object, which is instantiated upon application start and destroyed when the application stops–i.e., an instance of the`Application`class lasts for the lifetime of the Linux process of the application. When the process is terminated and restarted, a new`Application`instance is created.

### COMPONENTS {#_components}

The fundamental pieces of an Android application are the components managed by the Dalvik runtime:`Activity`,`Service`,`BroadcastReceiver`, and`ContentProvider`. The configuration and interaction of these components define the application’s behavior. These entities have different responsibilities and lifecycles, but they all represent application entry points, where the application can be started. Once a component is started, it can trigger another component, and so on, throughout the application’s lifecycle.

A component triggers the start of another component through an`Intent`, either within the application or between applications. The`Intent`specifies actions for the receiver to act upon—for instance, sending an email message or taking a photograph—and can also provide data from the sender to the receiver. An`Intent`can be explicit or implicit:

###### Explicit `Intent`

Defines the fully classified name of the component, which is known within the application at compile time.

###### Implicit `Intent`

A run-time binding to a component that has defined a set of characteristics in an IntentFilter. If the`Intent`matches the characteristics of a component’s `IntentFilter`, the component can be started.

Components and their lifecycles are Android-specific terminologies, and they are not directly matched by the underlying Java objects. A Java object can outlive its component, and the runtime can contain multiple Java objects related to the same live component. This is a source of confusion, and as we will see in[Chapter 6](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch06.html), it poses a risk for memory leaks.

An application implements a component by subclassing it, and all components in an application must be registered in the_AndroidManifest.xml_file.

#### Activity {#_activity}

An`Activity`is a screen—almost always taking up the device’s full screen—shown to the user. It displays information, handles user input, etc. It contains all the UI components–buttons, texts, images, and so forth–shown on the screen and holds an object reference to the view hierarchy with all the`View`instances. Hence, the memory footprint of an`Activity`can grow large.

When the user navigates between screens,`Activity`instances form a stack. Navigation to a new screen pushes an`Activity`to the stack, whereas backward navigation causes a corresponding pop.

In[Figure 1-2](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#fig_activity_stack), the user has started an initial`Activity`A and navigated to B while A was finished, then on to C and D. A, B and C are full-screen, but D covers only a part of the display. Thus, A is destroyed, B is totally obscured, C is partly shown, and D is fully shown at the top of the stack. Hence,_D_has focus and receives user input. The position in the stack determines the state of each`Activity`:



![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter01/activity_stack.png.jpg "Activity stack.")

Figure 1-2. Activity stack

* Active in the foreground: D
* Paused and partly visible: C
* Stopped and invisible: B
* Inactive and destroyed: A

The state of an application’s topmost`Activity`has an impact on the application’s system priority—a.k.a._process rank_—which in turn affects both the chances of terminating an application \([Application Termination](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#section_application_termination)\) and the scheduled execution time of the application threads \([Chapter 3](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch03.html)\).

An Activity lifecycle ends either when the user navigates back—i.e. presses the back button–or when the`Activity`explicitly calls`finish()`.

#### Service {#_service}

A`Service`can execute invisibly in the background without direct user interaction. It is typically used to offload execution from other components, when the operations can outlive their lifetime. A Service can be executed in either a _started _or a _bound _mode:

##### Started Service

The Service is started with a call to `Context.startService(Intent)`with an explicit or implicit`Intent`. It terminates when

`Context.stopService(Intent)`is called.

##### Bound Service

Multiple components can bind to a`Service`through`Context.bindService(Intent, ServiceConnection, int)`with explicit or implicit`Intent`parameters. After the binding, a component can interact with the`Service`through the`ServiceConnection `interface, and it unbinds from the`Service`through `Context.unbindService(ServiceConnection) `. When the last component unbinds from the `Service`, it is destroyed.

#### ContentProvider {#_contentprovider}

An application that wants to share substantial amounts of data within or between applications can utilize a`ContentProvider`. It can provide access to any data source, but it is most commonly used in collaboration with SQLite databases, which are always private to an application. With the help of a`ContentProvider`, an application can publish that data to applications that execute in remote processes.

#### BroadcastReceiver {#_broadcastreceiver}

This component has a very restricted function: it listens for Intents sent from within the application, remote applications, or the platform. It filters incoming Intents to determine which ones are sent to the`BroadcastReceiver`. A`BroadcastReceiver`should be registered dynamically when you want to start listening for Intents, and unregistered when it stops listening. If it is statically registered in the`AndroidManifest`, it listens for Intents while the application is installed. Thus, the`BroadcastReceiver`can start its associated application if an Intent matches the filter.

## Application Execution {#_application_execution}

Android is a multi-user, multi-tasking system that can run multiple applications at the same time and let the user switch between applications without noticing a significant delay. The multi-tasking is handled by the Linux kernel, and application execution is based on Linux processes.

### LINUX PROCESS {#_linux_process}

Linux assigns every user a unique user ID, basically a number tracked by the OS to keep the users apart. Every user has access to private resources protected by permissions, and no user \(except _root_, the superuser, which does not concern us here\) can access another user’s private resources. Thus, sandboxes are created to isolate users. In Android, every application package has a unique user ID; i.e., an application in Android corresponds to a unique user in Linux and cannot access other applications’ resources.

What Android adds to each process is a runtime execution environment, the Dalvik virtual machine, for each instance of an application.[Figure 1-3](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#fig_app_execution)shows the relationship between the Linux process model, the VM, and the application.



![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter01/app_execution.png "An image of two apps executing in Linux processes.")

Figure 1-3. Applications execute in different processes and VMs

By default, applications and processes have a one-to-one relationship, but if required, it is possible for an application to run in several processes, or for several applications to run in the same process.

### LIFECYCLE {#_lifecycle}

The application lifecycle is encapsulated within its Linux process, which–in Java—maps to the`android.app.Application`class. The`Application`object for each app starts when the runtime calls its`onCreate()`method. Ideally, the app terminates with a call by the runtime to its`onTerminate()`, but an application can not rely upon this. The underlying Linux process may have been killed before the runtime had a chance to call`onTerminate()`. The`Application`object is the first component to be instantiated in a process and the last to be destroyed.

#### Application Start {#section_application_start}

An application is started when one of its components is initiated for execution. Any component can be the entry point for the application and once the first component is triggered to start a Linux process is started—unless it is already running—which leads to the following start up sequence:

1. Start Linux process.
2. Create Dalvik Virtual Machine.
3. Create Application instance.
4. Create the entry point component for the application.

Setting up a new Linux process and the runtime is not an instantaneous operation. It can degrade performance and have a noticeable impact on the user experience. Thus, the system tries to shorten the startup time for Android applications by starting **a special process called Zygote** on system boot. Zygote has the entire set of core libraries preloaded. New application processes are forked from the Zygote process without copying the core libraries, which are shared across all applications.

#### Application Termination {#section_application_termination}

A process is created at the start of the application and finishes when the system wants to free up resources. Because a user may request an application at any later time, the runtime avoids destroying all its resources until the number of live applications leads to an actual shortage of resources across the system. Hence, an application isn’t automatically terminated even when all of its components have been destroyed.

When the system is low on resources, it’s up to the runtime to decide which process should be killed. To make this decision, the system imposes a ranking on each process depending on the application’s visibility and the components that are currently executing. In the following ranking, the bottom-ranked processes are forced to quit before the higher-ranked. The process ranks are \(highest first\):

##### Foreground

Application has a visible component in front,`Service`is bound to an`Activity`in front in a remote process or`BroadcastReceiver `is running.

##### Visible

Application has a visible component but is partly obscured.

##### Service

Service is executing in the background and is not tied to a visible component.

##### Background

A non-visible `Activity`. This is the process level that contains most applications.

##### Empty

A process without active components. Empty processes are kept around to improve startup times, but they are the first to be terminated when the system reclaims resources.

In practice, the ranking system ensures that no visible applications will be terminated by the platform when it runs out of resources.



LIFECYCLES OF TWO INTERACTING APPLICATIONS.

This example illustrates the lifecycles of two processes P1 and P2 that interacts in a typical way \([Figure 1-4](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#fig_app_lifecycle)\). P1 is a client application that invokes a`Service`in P2, a server application. The client process, P1, starts when it is triggered by a broadcasted`Intent`. At startup, the process starts both a`BroadcastReceiver`and the`Application`instance. After a while, an`Activity`is started, and during all of this time P1 has the highest possible process rank: Foreground.



![](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/images/chapter01/application_lifecycle.png "Starting remote service.")

Figure 1-4. Client application starts `Service`in other process.

The`Activity`offloads work to a`Service`that runs in process P2, which starts the`Service`and the associated`Application`instance. Hence, the application has split the work into two different processes. The P1`Activity`can terminate while the P2`Service`keeps running.

Once all components have finished—the user has navigated back from the`Activity`in P1, and the`Service`in P2 is asked by some other process or the runtime to stop—both processes are ranked as Empty, making them plausible candidates for termination by the system when it requires resources.

A detailed list of the process ranks during the execution appears in[Table 1-1](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#table_process_rank).



Table 1-1. Process rank transitions.

| Application state | P1 Process Rank | P2 Process Rank |
| :--- | :--- | :--- |
| P1 starts with BroadcastReceiver entry point | Foreground | N.A. |
| P1 starts Activity | Foreground | N.A. |
| P1 starts Service entry point in P2 | Foreground | Foreground |
| P1 Activity is destroyed | Empty | Service |
| P2 Service is stopped | Empty | Empty |

It should be noted that there is a difference between the actual application lifecycle—defined by the Linux process–and the perceived application lifecycle. The system can have multiple application processes running even while the user perceives them as terminated. The empty processes are lingering–if system resources permit it—to shorten the startup time on restarts.

## Structuring Applications for Performance {#_structuring_applications_for_performance}

Android devices are multiprocessor systems that can run multiple operations simultaneously, but it is up to each application to ensure that operations can be partitioned and executed concurrently to optimize application performance. If the application does not enable partitioned operations, but prefers to run everything as one long operation, it can exploit only one CPU, leading to suboptimal performance. Unpartitioned operations must run_synchronously_, whereas partitioned operations can run_asynchronously_. With asynchronous operations, the system can share the execution among multiple CPUs and therefore increase throughput.

An application with multiple independent tasks should be structured to utilize asynchronous execution. One approach is to split application execution into several processes, because those can run concurrently. However, every process allocates memory for its own substantial resources, so the execution of an application in multiple processes will use more memory than an application in one process. Furthermore, starting and communicating between processes is slow, and not an efficient way of achieving asynchronous execution. Multiple processes may still be a valid design, but that decision should be independent of performance. To achieve higher throughput and better performance, an application should utilize multiple threads within each process.

### CREATING RESPONSIVE APPLICATIONS THROUGH THREADS {#introduction_responsive_threads}

An application can utilize asynchronous execution on multiple CPU’s with high throughput, but that doesn’t guarantee a responsive application. Responsiveness is the way the user perceives the application during interaction: that the UI responds quickly to button clicks, smooth animations, etc. Basically, performance from the perspective of the user experienced is determined by how fast the application can update the UI components. The responsibility for updating the UI components lies with the_UI thread_,\[[1](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#ftn.id881793)\]which is the only thread the system allows to handle UI updates.

To make the application responsive, it should ensure that no long-running tasks are executed on the UI thread. If they do, all the other execution on that thread will be delayed. Typically, the first symptom of executing long running tasks on the UI thread is that the UI becomes unresponsive because it is not allowed to update the screen or accept user button presses properly. If the application delays the UI thread too long, typically 5-10 seconds, the runtime displays an “Application Not Responding” \(ANR\) dialog to the user, giving her an option to close the application. Clearly, you want to avoid this. In fact, the run-time prohibits certain time-consuming operations, such as network downloads, from running on the UI thread.

So long operations should be handled on a background thread. Long-running tasks typically include:

* Network communication
* Reading or writing to a file
* Creating, deleting, and updating elements in databases
* Reading or writing to`SharedPreferences`
* Image processing
* Text parsing

WHAT IS A LONG TASK?

There is no fixed definition of a long task or a clear indication when a task should execute on a background thread, but as soon as a user perceives a lagging UI–e.g., slow button feedback and stuttering animations—it is a signal that the task is too long to run on the UI thread. Typically, animations are a lot more sensitive to competing tasks on the UI thread than button clicks, because the human brain is a bit vague about when a screen touch actually happened. Hence, let us do some coarse reasoning with animations as the most demanding use case.

Animations are updated in an event loop where every event updates the animation with one frame, i.e., one drawing cycle. The more drawing cycles that can be executed per time frame, the better the animation is perceived. If the goal is to do 60 drawing cycles per second—a.k.a. frames per second \(fps\)—every frame has to render within 16ms. If another task is running on the UI thread simultaneously, both the drawing cycle and the secondary task have to finish within 16ms to avoid a stuttering animation. Consequently, a task may require less than 16ms execution time and still be considered as long.

The example and calculations are coarse and meant as an indication of how an applications responsiveness can be affected not only by network connections that last for several seconds, but also tasks that at a first glance look harmless. Bottlenecks in your application can hide anywhere.

Threads in Android applications are as fundamental as any of the component building blocks. All Android components and system callbacks—unless denoted otherwise—runs on the UI thread, and should use background threads when executing longer tasks.

## Summary {#_summary}

An Android application runs on top of a Linux OS in a Dalvik runtime, which is contained in a Linux process. Android applies a process ranking system that priorities the importance of each running application to ensure that it is only the least prioritized applications that are terminated. To increase performance, an application should split operations among several threads so that the code is executed concurrently. Every Linux process contains a specific thread that is responsible for updating the UI. All long operations should be kept off the UI thread and executed on other threads.

  


\[[1](https://www.safaribooksonline.com/library/view/efficient-android-threading/9781449364120/ch01.html#id881793)\]Also known as the_main thread_, but throughout this book we stick to the convention of calling it the “UI thread.”

