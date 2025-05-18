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

Semaphores are not necessarilly a way of synchronizing threads. Rather, they are a way of limiting how many threads are allowed to be executed at a given time.

A use case for this would be a database connection pool. Imagine you have a connection pool of 10 connections and you cannot go over that limit. A semaphore would be a good use case here because it would allow you to limit the number of threads that can actually get a connection from the connection pool.

Similarly to Mutex, Semaphores can be local (meaning they only span across one process) or they can be global, thus spanning across multiple processes.

note: again there are 2 classes for semaphores: `SemaphoreSlim` and `Semaphore`. Use `SemaphoreSlim` for local only semaphores and `Semaphore` for global (in fact only the `Semaphore` class has the "name" property)