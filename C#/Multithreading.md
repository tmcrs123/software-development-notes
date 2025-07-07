## Concepts

CPU - What process threads. "Threads" is the only language the CPU understands
Thread - strand of logic belonging to one application
Thread Scheduler - what allows OS to assign threads to a given CPU core for processing. There's nothing a developer can do to change the way the thread scheduler works

Note that operations on the main thread are **BLOCKING** but operations on other threads are non-blocking. For example:

```C#
 static void Main(string[] args)
 {
     doThings(); // this is the main thread so code here is top to bottom as is blocking
     BasicSyntax();
     Console.ReadLine();
 }

 static void BasicSyntax()
 {
     Action<string> f = (string threadId) => Console.WriteLine($"hello from {threadId}");
     
     Thread thread1 = new Thread(doThings);
     thread1.Start();

     Thread thread2 = new Thread(doThings);
     thread2.Start();
 }

 static void doThings()
 {
     for (int i = 0; i < 100; i++)
     {
         Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} doing things...");
     }

     Console.WriteLine($"{Thread.CurrentThread.ManagedThreadId} finished doing things");
 }
```

The output of this code is `1` times 100 because it's on the main thread and then we will get two random numbers because the code is being executed in different threads.

Also note that you can influence how the thread scheduler works by using thread priority but that doesn't guarantee the thread with the highest priority will be completed first. The highest priority thread will get more CPU time earlier but the thread scheduler will share CPU time with other threads so not to starve them.

## Joining threads

There's a method called `Thread.join()` which needs to be invoked on the thread and essentially it means "wait until all other threads are finished and re-join the main thread". This is a blocking method. The naming is weird because it derives from Posix stuff. A visual analogy would be:

Start together
│
├─🧍‍♂️ Thread A (Main)
│   └── Waits at Join point 🪑
│
└─🏃‍♂️ Thread B (Worker)
    └── Runs ahead and does some work 🛠️
        └── Finishes and meets back with Thread A

🧍‍♂️ + 🏃‍♂️ → Walk together again 🚶‍♂️🚶‍♂️

## Thread Synchronization Mechanisms

You can run into problems when multiple threads access shared resources. Sometimes these problems are not obvious. Consider this code where you have 2 threads incrementing a counter:

```C#
static void syncProblemMain()
{
    int count = 0;

    Thread t1 = new Thread(IncrementCounter);
    t1.Start();

    t1.Join();
    
    Thread t2 = new Thread(IncrementCounter);
    t2.Start();

    t2.Join();

    Console.WriteLine($"Final counter value is {count}");


    void IncrementCounter()
    {

        for (int i = 0; i < 100000; i++)
        {
            count += 1; // not atomic!
        }
    }
}
```

While you would expect the counter to be exactly 200.000 in the end that is not the case. And this is not because the `for..loops` run more than they are meant to, this is due to the `count += 1;`  part.

Under the hood this compiles to:

```C#
var temp = count;
count = temp + 1
```

If you have 2 threads doing this, you will end-up in a scenario where _both_ threads do `var temp = count;` before actually updating the count, meaning that for that iteration of the loop the count didn't increment. This will happen when t1 does `var temp = 1000` and then the program immediately jumps to t2 and does `var temp = 1000`, without allow thread 1 to actually update the variable. How to fix this? Read on...

## Exclusive locks

An exclusive lock is a way of telling .NET that a section of your code can only be entered by one thread at a time. The way this is written is by using lock objects. The syntax for locks and solution to the problem highlighted above is as follows:

```C#
static void syncProblemMain()
{
    int count = 0;
    object counterLock = new object();

    Thread t1 = new Thread(IncrementCounter);
    t1.Start();

    
    Thread t2 = new Thread(IncrementCounter);
    t2.Start();

    t1.Join();
    t2.Join();

    Console.WriteLine($"Final counter value is {count}");

    void IncrementCounter()
    {

        for (int i = 0; i < 100000; i++)
        {
            lock(counterLock) {
                count += 1;
            }
        }
    }
}
```

A note on the lock object: C# does not have an object of type `Lock`. You need to give it any object (and not a primitive) because under the hood the CLR looks at the object _reference_ to know if that lock is being used or not. Therefore anything that is passed by reference can work as a lock.

## Monitor

The `Monitor` is one step above from using `lock` (in fact, under the hood, `lock` uses `Monitor`). The term `Monitor` comes from the fact that this is an entity that _monitors_ the critical section. In terms of basic syntax is as follows:

```C#
Monitor.Enter(someLockObject)

// critical section logic

Monitor.Exit(someLockObject)
```

So what's the advantage of using `Monitor` over `lock` ?

The main advantage is that `Monitor` has a `TryEnter` method.

Imagine the following scenario: You have a ticket booking system and the processing of the booking takes a lot of time. If a given thread processing a users booking does not acquire a lock, you need to let the user know the system is busy doing other work and that it needs to try again later. `Monitor` allows you to achieve this.

```C#
if(Monitor.TryEnter(somelock, timeout)){
	 // critical section code
} else {
Console.WriteLine("System is busy")
}
```


Note: both `Monitor` and `lock` work at the thread level meaning it only locks threads within the same process. If your app spans across multiple processes these synchronization mechanisms won't work. For that you need a mutex. 
## Mutex

A Mutex is a synchonization mechanism that acts across multiple processes. This allows you to control access to resources at a global operative system level.

For a mutex to be global it needs to have a name. Named/Global mutex are visible throughout the operating system and can be used to synchronize processes. A good example of this would be different processes accessing a shared resource (think spinning up an app that reads and write from one single file 10 times).

Here's an example:

```C#
var path = "./bananas.txt"; 

for (int i = 0; i < 100; i++)
{
    using var mutexLock = new Mutex(false, "bananasMutex");

    mutexLock.WaitOne();
    
    Read(path);
    Write(path);

    mutexLock.ReleaseMutex();

    Console.WriteLine("Process finished");
}


void Write(string path)
{
    var counter = Read(path);
    counter += 1;
    File.WriteAllText(path, counter.ToString());
}

int Read(string path)
{
    using var stream = new FileStream(path, FileMode.OpenOrCreate, FileAccess.ReadWrite, FileShare.ReadWrite);
    using var reader = new StreamReader(stream);

    var counterAsStr = reader.ReadToEnd();

    return string.IsNullOrEmpty(counterAsStr) ? 0 : int.Parse(counterAsStr);
}
```

Because Mutex works at the operating system level it is slower that locks or monitors.

### When should you use local mutex?

Almost never. A local mutex only works at the process level so you can achieve the same thing with locks or monitors. However mutex locks have extend from `WaitHandle`. And with `WaitHandle` you can do event logic because you can wait on one or more things that implement `WaitHandle` and `Mutex` is one of those things. So if you need event-like logic go with `Mutex`.

Here's an example:

```C#
for (int i = 0; i < 10; i++)
{
    using var mutexLock = new Mutex(false, "bananasMutex");
    var s = new ManualResetEvent(false);

    mutexLock.WaitOne();


    Read(path);
    Write(path);

    mutexLock.ReleaseMutex();

    var a = WaitHandle.WaitAll(new WaitHandle[] { mutexLock, s });

    Console.WriteLine("Process finished"); // This never fires because ManualResetEvent never fires
}
```

## Reader / Writer locks

In a multithreaded scenario sometimes it's useful and more performant to allow multiple threads to read a resource. When you think about, read only threads do not alter the underlying resource so there's no problem to allow multiple reads and the same time. Writes on the other hand change the resource so that needs to be locked. This is exactly what these type of locks achieve. It's allows multiple readers to access the resource, but only allow reads when no write threads have the lock.

Here's an example of making a normal dictionary concurrent.

```C#
var RWLock = new ReaderWriterLockSlim();
var cache = new Dictionary<int, string>();

void Add(int key, string value)
{
    var lockAcquired = false;
    try
    {
        RWLock.EnterWriteLock();
        lockAcquired = true;
        cache.Add(key, value);
    }
    finally
    {
        if (lockAcquired) RWLock.ExitWriteLock();
    }
}

string? Read(int key)
{
    var lockAcquired = false;

    try
    {
        RWLock.EnterReadLock();
        lockAcquired = true;
        return cache.TryGetValue(key, out var value) ? value : null;
    }
    finally
    {
        if (lockAcquired) RWLock.ExitReadLock();
    }
}
```

Note: There are 2 classes for these types of lock: `ReaderWriterLockSlim` and `ReaderWriterLock`.  `ReaderWriterLock` is the old legacy version and you should no longer use it, this is kept for backwards compatibility reasons.

## Semaphore

Semaphores are not necessarily a way of synchronizing threads. Rather, they are a way of limiting how many threads are allowed to be executed at a given time.

A use case for this would be a database connection pool. Imagine you have a connection pool of 10 connections and you cannot go over that limit. A semaphore would be a good use case here because it would allow you to limit the number of threads that can actually get a connection from the connection pool.

Similarly to Mutex, Semaphores can be local (meaning they only span across one process) or they can be global, thus spanning across multiple processes.

note: again there are 2 classes for semaphores: `SemaphoreSlim` and `Semaphore`. Use `SemaphoreSlim` for local only semaphores and `Semaphore` for global (in fact only the `Semaphore` class has the "name" property).

Here's a simple example of using a semaphore:

```C#
internal class Program
{
    private static void Main(string[] args)
    {
        SemaphoreSlim semaphore = new SemaphoreSlim(3);

        Thread[] threads = new Thread[10];

        threads = [
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation),
            new Thread(LongRunningOperation)
            ];

        foreach (var t in threads)
        {
            t.Start();
        }

        foreach (var t in threads)
        {
            t.Join();
        }

        Console.WriteLine("All threads finished, disposing of semaphore...");

        semaphore.Dispose();

        void LongRunningOperation()
        {
            semaphore.Wait();
            Console.WriteLine($"Thread ${Thread.CurrentThread.ManagedThreadId} entered the semaphore. There are {semaphore.CurrentCount} threads in the semaphore.");

            try
            {
                Thread.Sleep(5000);
            }
            finally
            {
                var previousCount = semaphore.Release();
                Console.WriteLine($"Thread ${Thread.CurrentThread.ManagedThreadId} released by the semaphore. There are {previousCount} threads in the semaphore.");
            }
        }
    }
}
```

## Thread Interaction

### AutoResetEvent

`AutoResetEvent` works like a signalling mechanism. Think of the analogy of a policeman signalling traffic to move.

`AutoResetEvent` has an initial value of type boolean which signals if the event is signaled or non-signaled. 

The way this works is once the `AutoResetEvent` signals (changes from `true` to `false`), any threads that are waiting on that signal are notified to do their work BUT as soon as one of the threads receives the signal, the signal is automatically reset back to `false`.

This means that only one thread will be able to act on that signal at any given time.

The main implication of this behaviour is that it means the worker threads only start when the event goes from `false -> true` . 

Consider the following example:

```C#
internal class Program
{
    static AutoResetEvent myEvent = new AutoResetEvent(false);
    
    static void Main(string[] args)
    {

        for (int i = 0; i < 3; i++)
        {
            Thread t = new Thread(Consumer);
            t.Name = $"Consumer thread {i}";
            Console.WriteLine($"{t.Name} listening...");
            t.Start();
        }

        while (true)
        {
            var input = Console.ReadLine();

            if(input == "g")
            {
                myEvent.Set();
            }
        }
    }

    static void Consumer()
    {
        while (true)
        {
            myEvent.WaitOne();

            Console.WriteLine($"{Thread.CurrentThread.Name} got event, processing...");
            Thread.Sleep(5000);
            Console.WriteLine($"{Thread.CurrentThread.Name} finished processing...");
        }
    }
}
```

In this case, the user can hammer the "g" key but if a worker thread is already executing work, any clicks that (re)set the signal from `true -> true`  are lost. It's only when on the worker threads finished (thus setting the event back to `false`) that inputs are going to be acted upon.

## ManualResetEvent

This works similarly to `AutoResetEvent` if the exception that you need to manage setting and resetting the event yourself. What this is allows is the behaviour of having multiple threads doing work at the same time unlike before.

In terms of syntax it's pretty similar but now there's a slim version. An example:

```C#
 internal class Program
 {
     static ManualResetEventSlim m = new ManualResetEventSlim();
     
     static void Main(string[] args)
     {

         Console.WriteLine("Press enter to release all threads");
         Console.ReadLine();

         for (int i = 0; i < 3; i++)
         {
             Thread t = new Thread(DoWork);
             t.Start();
         }

         m.Set();

         Console.ReadLine();
     }

     static void DoWork()
     {
         m.Wait();

         Console.WriteLine($"{Thread.CurrentThread.Name} doing work");
         Thread.Sleep(2000);
         Console.WriteLine($"{Thread.CurrentThread.Name} finished doing work");
     }
 }
```

### Important caveat on syntax

The `AutoResetEvent` and `ManualResetEvent` rely on the concept of the event being "signaled". So what does "signaled" mean? It means the event is set to `true`.  It **does not** mean "the event has changed from true to false".

Consider the following code below. In this scenario, the `Wait()` line won't be blocked because the event is signaled to true (it's instantiated as a signaled event)

```C#

static ManualResetEventSlim produceEvent = new ManualResetEventSlim(true);

(...)

 static void Produce()
 {
     while (true)
     {
         produceEvent.Wait(); // THIS DOES NOT BLOCK HERE BECAUSE THE EVENT IS INSTANTIATED AS TRUE
         DoThings();
     }
}
```

Here's a full example of a producer consumer scenario where the initial state of the event matters.

```C#
 internal class Program
 {
     static Queue<int> foodQueue = new Queue<int>();

     static ManualResetEventSlim produceEvent = new ManualResetEventSlim(true);
     static ManualResetEventSlim consumeEvent = new ManualResetEventSlim(false);
     static object consumeLock = new object();

     static void Main(string[] args)
     {
         Console.WriteLine("Animal Farm");

         for (int i = 0; i < 3; i++)
         {
             new Thread(Consume).Start();
         }

         Produce();
     }

     static void Produce()
     {
         while (true)
         {
             produceEvent.Wait();
             produceEvent.Reset();

             Console.WriteLine("Press p to produce");
             var input = Console.ReadLine();

             if (input == "p")
             {
                 Console.WriteLine("Farmer adds more food");

                 for (int i = 0; i < 10; i++)
                 {
                     foodQueue.Enqueue(i);
                 }

                 Console.WriteLine("Food is full");
                 consumeEvent.Set();
             }
         }
     }

     static void Consume()
     {
         while (true)
         {
             consumeEvent.Wait();
             consumeEvent.Reset();

             if (foodQueue.Count == 0) continue;

             lock (consumeLock)
             {
                 while (foodQueue.TryDequeue(out var result))
                 {
                     Console.WriteLine($"Thread {Thread.CurrentThread.ManagedThreadId} consumed {result}");
                 }
             }

             Console.WriteLine("No more food left in the queue");
             produceEvent.Set();
         }
     }
 }
```


## Thread Affinity

Thread affinity means binding a task or operation to a specific thread. This means that some operations need to be perform by the thread they are bound to otherwise an exception may be thrown. This happens mostly when doing UI because re-rendering needs to happen on the UI thread so you may need to sync things if you are doing multithreaded UI.

## Thread Safety

This just means "precautions have been put in place to allow multiple threads to access resource at the same time without corrupting data or creating unexpected results".

## Nested Locks and Deadlocks

Deadlocks happen when you have a thread holding on to a lock (but not releasing) another thread needs and vice versa.

Imagine an example where you're trying to process Users and Orders (something like user has a order AND order has a user). This could look something like this:

```C#
namespace Deadlocks
{
    internal class Program
    {
        static object userLock = new object();
        static object orderLock = new object();

        static void Main(string[] args)
        {
            Thread orderThread = new Thread(ProcessOrder);
            orderThread.Start();

            lock (userLock)
            {
                Console.WriteLine("Main thread - obtained user lock");
                DoUserThings();

                lock (orderLock)
                {
                    Console.WriteLine("Main thread - obtained order lock");
                    DoOrderThings();
                }
            }

            Console.WriteLine("Program finished");
            Console.ReadLine();

        }

        static void ProcessOrder()
        {
            lock (orderLock)
            {
                Console.WriteLine("Worker thread - Process order obtained order lock");
                DoOrderThings();

                lock (userLock)
                {
                    Console.WriteLine("Worker thread - Process order obtained user lock");
                    DoUserThings();
                }
            }
        }

        static void DoUserThings()
        {
            Thread.Sleep(1000);
        }

        static void DoOrderThings()
        {
            Thread.Sleep(2000);
        }
    }
}
```

---
## Misc topics on Threads

### Thread states

Threads have different states. The enum is documented [here](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadstate?view=net-9.0). 

The important things to remember is that the first state of a thread is `unstarted` meaning `start()` as not being invoked on the thread. It's also important to note that a thread that is being waited on your where `sleep()` has been called, is in a  `WaitSleepJoin` state rather than `Stopped`. 

There is an important distinction between `Stopped` and `Aborted` states. A `Stopped` thread is a thread that has finished it's processing and has exited gracefully. A thread cannot be stopped directly, there is no `Thread.Stop()`. The way to cancel it is via a `CancellationToken()`. 
An `Abort` thread is a thread that has been aborted manually with `Thread.Abort()`. This is the old way of doing things and now is deprecated and should not be used.

Here's how to use a cancellation token to stop a thread:

```C#
class Program
{
    static void Main()
    {
        var cts = new CancellationTokenSource();

        Thread t = new Thread(() => DoWork(cts.Token));
        t.Start();

        Console.WriteLine("Press Enter to stop thread...");
        Console.ReadLine();
        cts.Cancel(); // Signal the thread to stop
        t.Join(); // Wait for thread to finish
    }

    static void DoWork(CancellationToken token)
    {
        while (!token.IsCancellationRequested)
        {
            Console.WriteLine("Working...");
            Thread.Sleep(500); // Simulate work
        }

        Console.WriteLine("Thread exiting gracefully.");
    }
}
```

### Debugging multithreaded apps

There are 2 useful windows in Visual Studio. Go to Debug -> Windows and select "Parallel Stacks" and "Threads"

### Making a thread wait

There are 3 ways of making a thread wait:

 - `Thread.Sleep`
 - `Thread.SpinWait`
 - `SpinWait.SpinUntil`

`Thread.Sleep` is the preferable one because it tells the scheduler to actually change the state of the thread from `Running` to `WaitSleepJoin` , thus allow CPU resources to be released for better performance.

The other two are essentially ways of keeping the thread "spinning" (imagine a for loop up to a million here). The key difference here is that the state of the thread never changes, it just keeps running around in circles. This is worse for performance because the state of the thread is never changed (so always `Running`) and CPU resources are never released.

### Returning results from a thread

You cannot do this. This code does not work:

```C#
Thread t = new Thread(DoThings);
var resul = t.start() //WRONG!
```

The only way to "return" something from a thread is be creating a variable and using the thread to set that variable.

### Thread Pool

The idea of thread pools is that creating threads on the fly is expensive. Also you end up in a (very remote) scenario where you exhaust all possible - imagine you have a server that creates a thread for every request and you get 1 million concurrent request.

So a safer way of doing this is using a thread pool.

A thread pool, as the name implies, is a pool of threads ready to be used when you need them.

You can specify a minimum and a maximum. If the pool does not have enough threads to meet the minimum, when a new thread is request the thread pool will create it. If, on the other hand, the maximum number of threads has been meet and a new thread is needed, the thread pool will force the requester of the new thread to wait until a thread is available.

Thread pools are a way of making sure your system never blows up in terms of processing.

The syntax to use the `ThreadPool` is slightly different from using normal threads because the `QueueUserWorkItem` requires a delegate that complies with the `WaitCallback` delegate signature from C# . Here's an example:

```C#
 void Worker(object filename)
 {
     var res = GetOrderInsertForFile(filename.ToString());

     lock (appendLock)
     {
         sb.Append(res);
     }
 }

ThreadPool.QueueUserWorkItem(Worker, fileInfo);
```

### Exception handling in threads

Whenever you spin up a new thread it gets it's own call stack, separate from the main thread. This means that if you thread throws an exception it never bubbles up to the main thread. Any exceptions that your thread throws need to be handled inside that thread. For example, the code below does not catch the exception thrown from the thread.

```C#
static void Main(string[] args)
{
    void Work()
    {
        throw new InvalidOperationException("Thrown for thread");
    }

    try
    {
        Thread t = new Thread(Work);
        t.Start();
        t.Join();
    }
    catch (Exception e)
    {
        Console.WriteLine("Caught exception in main thread"); // this will never be caught!
    }
}
```

The correct way to catch an exception in a thread is like this:

```C#
 static void Main(string[] args)
 {
     void Work()
     {
         try
         {

             throw new InvalidOperationException("Thrown for thread");
         }
         catch (Exception e)
         {

             Console.WriteLine("Caught exception in thread");
         }
     }

     try
     {
         Thread t = new Thread(Work);
         t.Start();
         t.Join();
     }
     catch (Exception e)
     {
         Console.WriteLine("Caught exception in main thread");
     }
 }
```

Note that even if you catch the exception in the thread and re-throw it, it's still not caught by the main thread.

Also the behaviour is the same even if you have nested threads (other than the main thread). **Each thread has it's own call stack and you need to catch exceptions within the context of the thread. Period.**

```C#
 static void Main(string[] args)
 {
     void MoreWork()
     {
         throw new InvalidOperationException("Thrown for thread 2"); // This will NEVER be caught
     }

     void Work()
     {
         Thread t2 = new Thread(MoreWork);

         try
         {
             t2.Start();
         }
         catch (Exception e)
         {
             Console.WriteLine($"Caught exception in thread 1: {e.Message}");
         }
     }

     try
     {
         Thread t1 = new Thread(Work);
         t1.Start();
         t1.Join();
     }
     catch (Exception e)
     {
         Console.WriteLine($"Caught exception in main thread: {e.Message}");
     }
 }
```

But what if we really need to handle exceptions in a centralized way, regardless on which thread they are created? In that case the only way is to save the exception in a centralized resource and then process the exceptions (if any) when all the threads complete.

```C#
    static void Main(string[] args)
    {
        List<Exception> exceptions = new List<Exception>();
        object exceptionsListLock = new object();
        
        void MoreWork()
        {
            try
            {

                throw new InvalidOperationException("Thrown for thread 2");
            }
            catch (Exception e)
            {
                lock (exceptionsListLock)
                {
                    exceptions.Add(e);
                }
            }
        }

        void Work()
        {
            Thread t2 = new Thread(MoreWork);

            try
            {
                t2.Start();
            }
            catch (Exception e)
            {
                Console.WriteLine($"Caught exception in thread 1: {e.Message}"); //Uncaught
            }

            t2.Join();
        }

        try
        {
            Thread t1 = new Thread(Work);
            t1.Start();
            t1.Join();
        }
        catch (Exception e)
        {
            Console.WriteLine($"Caught exception in main thread: {e.Message}"); //Uncaught
        }

        foreach (var e in exceptions)
        {
            Console.WriteLine($"Manually handled exception: {e.Message}"); //This works!
        }
    }
}
```

---

# Task Based Async Programming

## Understanding the difference between multithreading and async programming

The main difference is the emphasis of what you are trying to achieve.

Multithreading implies the concept of "divide and conquer".  This is when you need to want to accomplish something quicker and you can speed up the process by doing things in parallel. A good example is the following: Imagine you need to build a database of users but you have user information split into 10 different csv files. If you do it all in the main thread you can only do one file at a time BUT if you start a thread per file one can read the files in parallel, thus building the database much quicker.

Async programming on the other hand is not about "diving work to do things faster". Instead it's about offloading long running tasks that may not necessarily be CPU intensive. Classic example where is when you need to wait for a database call. You are just sitting there waiting for data to come back. But if you don't offload that task your program is stuck waiting.

Things get a bit blurred because async programming uses multithreading under the hood. When you offload an async task you are essentially putting that long running computation/wait in a different thread. But, again, the emphasis is different!

#### Syntax

Very similar syntax to `Thread`: 

```C#
Task t = new Task(() =>
{
    Thread.Sleep(1000);
    Console.WriteLine("Bananas");
});

t.Start();
t.Wait();
```

Alternatively you can do the following, which is the same as declaring a `Task` and calling `Start` on the task:

```C#
Task t2 = Task.Run(() =>
{
    Thread.Sleep(2000);
    Console.WriteLine("Bananas 2");
});

t2.Wait();
```

## Why use Tasks Vs Threads?

Tasks are an abstraction around threads that have some benefits. Namely:
- Tasks use the thread pool by default
- You can return values from a Task
- Better exception handling
- Async/Await - closer to writing sync code
- Async/Await - easier context management, no thread affinity problems

## Task Api

### Wait, WaitAll and Result

All these methods do what the name says. What needs to be mentioned here is that all these methods are *blocking*. Even `Task.Result`.

For example this code is blocked until a result from the Task is available.

```C#
Task<int> t = new Task<int>(() =>
{
    Task.Delay(2000);
    return 42;
});

Console.WriteLine(t.Result); // blocked for 2 seconds
```

### ContinueWith

`ContinueWith` is like `.then` promises in JavaScript. The only aspect where it's slightly different is that in JavaScript the argument passed to the `.then` callback is the value of the previous promise whereas with C#, the argument is the actual promise. So you need to call `.Result` if you want to get the return value out of it. 

Here's an example:

```C#
Console.WriteLine("Program running...");

Task<int> randomNumberTask = Task.Run(() =>
{
    Thread.Sleep(5000);
    return 2;
});

randomNumberTask.ContinueWith((randomNumber) =>
{
    Console.WriteLine(randomNumber.Result * 10);
});

while (true)
{
    var input = Console.ReadLine();
    Console.WriteLine($"Main thread is not blocked, user input: {input}");
}
```

A side note: one of the things I tried before getting to the code above was this:

```C#
Console.WriteLine("Program running...");

Task<int> randomNumberTask = Task.Run(() =>
{
    Task.Delay(5000)
    return 2;
});

randomNumberTask.ContinueWith((randomNumber) =>
{
    Console.WriteLine(randomNumber.Result * 10);
});

while (true)
{
    var input = Console.ReadLine();
    Console.WriteLine($"Main thread is not blocked, user input: {input}");
}
```

However, using `Task.Delay()` I immediately got the output of the `ContinueWith` promise (20). This is because `Task.Delay()` fires a task that will be resolved in X seconds on *a different thread* . So if you want this to hang there for X seconds, you need to use `async/await`.

### Unwrap

What happens if you chain more than one Task like this?

```C#
Task<int> randomNumberTask = Task.Run(() =>
{
    Thread.Sleep(1000);
    return 2;
});

Task<string> randomStringTask = Task.Run(() =>
{
    Thread.Sleep(1000);
    return "bananas";
});

Task<bool> boolTask = Task.Run(() =>
{
    Thread.Sleep(1000);
    return true;
});

var whatIsThis = randomNumberTask.ContinueWith((randomStringTask) => randomStringTask).ContinueWith(boolTask => boolTask);
```

Unlike Javascript where if you chain promises (like this)...

```javascript
var p1 = new Promise<string>(resolve => resolve("bananas"))

var p2 = new Promise<number>(resolve => resolve(1))

var p3 = new Promise<Boolean>(resolve => resolve(false))

var t = p1.then(() => p2).then(() => p3); //Promise<boolean>
```

...you get the last thing you return (in this case `Promise<boolean>`) in .NET you get nested task.

So this `var whatIsThis = randomNumberTask.ContinueWith((randomStringTask) => randomStringTask).ContinueWith(boolTask => boolTask);` will be a `Task<Task<Task<int>>>>`

This is not very useful because if you wanted to get the last value you'd still need to split into a variable and `await` 3 times.

However, there's a way around this using the `Unwrap` method. What `Unwrap` does is to take the return value of the task and pass it along rather than chaining the whole task. 

```C#

            Task<int> randomNumberTask = Task.Run(() =>
            {
                Thread.Sleep(1000);
                return 2;
            });

            Task<string> randomStringTask = Task.Run(() =>
            {
                Thread.Sleep(1000);
                return "bananas";
            });

            Task<bool> boolTask = Task.Run(() =>
            {
                Thread.Sleep(1000);
                return true;
            });

            var whatIsThis = randomNumberTask
                .ContinueWith((randomStringTask) => randomStringTask).Unwrap()
                .ContinueWith(boolTask => boolTask).Unwrap();
```
 
So, in this scenario, `whatIsThis` is going to be a `Task<int>`.