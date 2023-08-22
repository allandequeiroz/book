+++
title = "ThreadSafety"
description = ""
tags = [
    "java",
    "thread",
]
date = "2016-08-08"
categories = [
    "software",
    "engineering",
]
draft = false
+++

A few days ago I ended up into this [StackOverflow thread](https://stackoverflow.com/questions/7384875/what-is-wrong-with-this-thread-safe-byte-sequence-generator/7387782) from almost seven years ago, this guy was getting inconsistent results from a multithread code. Well, looked like an excellent excuse to recall some topics and do some new research.

To start with, let's just quickly remember the differences between Multiprogramming, Multiprocessing, Multitasking, and Multithreading.
  

### Multiprogramming
In a multiprogramming environment more than one __program__ can be loaded to the main memory, but only one will use the CPU to execute instructions, inside this sort of environment programs that are not using CPU  are waiting their turn, but off course this is not acceptable, imagine if the same program hold the CPU for 1h? To solve this problem, the OS will interrupt the current program as soon it starts to perform non CPU required operations, giving the control to one of the programs in memory `ready to execute` this process is a.k.a *context switch*. This way no CPU time is wasted waiting, e.g., I/O tasks to finish or to hope that the program will voluntarily release the CPU.
> Notice that at this point we're talking about whole programs

### Multiprocessing
To the end user multiprocessing may look similar to multiprogramming since we see multiple __programs__ executing at the same time, but different of multiprogramming that perform the trick through context switch and memory segregation only, multiprocessing is more about CPU units, if the hardware provides more than one processor, more than one program will execute at the same time. Keep in mind that multiprocessing and multiprogramming are not exclusive. A system can use both mechanisms.

### Multitasking
Multitasking is similar to what we have on multiprogramming, yet at this point we're not just talking about programs, we are now talking about programs, processes, tasks, and threads running at the same time if we have more than one CPU or the illusion of *"same time"* through *context switch* if not, although once again, they are not exclusive. The difference now is that we may have small processes or even sub-tasks using the CPU in a fair way, using __CPU time quantums__ defined by the scheduler instead allowing prosses to kidnap the CPU until it blocks. 

### Multithreading
Multithreading is a bit different from the previous models. The idea is that now we have multiple sub-segments *"threads"* running concurrently inside the context of another process, in other words, threads are child process sharing parent's resources executing independently. This concept is interesting because in the previous model each process had its resource quota to perform a task, in a multithreading environment, the primary process children share parent's resources so, a process can execute multiple computations in parallel through its children with the resources already available. But this advantage comes with some burden too. The programmer now needs to handle race conditions since now children are fighting for their parent's resources. Well, well, well.

So what can we do to manage race conditions and prevent the system from ending up inconsistently? There are some approaches.

- Stop sharing state across threads
- Make state immutable
- Use synchronization whenever the shared mutable data happen

To start with, I would like to do another recap, now about the Java thread memory model:
![Java thread memory model](https://i.imgur.com/IdBfUXs.png)

As you can see in the image above, Objects are kept in the Heap space and shared across all threads, do you remember the parent-child process idea? Java is the parent and threads the children, so Java threads have access to the parent resources, including the Heap space.
> JVMs can use a technique called escape analysis, by which they can tell that specific objects remain confined to a single thread for their entire lifetime, and that lifetime is bounded by the lifetime of a given stack frame. Such objects can be safely allocated on the stack instead of the heap.

But Java threads have its private area, the thread stack, where local variables and partial method results live their lives.

Another important aspect when talking about threads is the CPU cache. In a multithreaded environment for performance reasons each thread may keep a copy of the variables inside the CPU cache, in this case, if the thread operating over the CPU A change a variable, there's no guarantee that this change will be replicated back to the main memory so, another thread would be looking at an inconsistent value *visibility problem*.

![CPU Memory Cache](https://i.imgur.com/R9hYCkz.png)

The mechanism available in Java to address visibility problems is the __volatile__ keyword, declaring a variable volatile indicates that changes to that variable will be written back to the main memory immediately and read as well will happen directly from the main memory. But even volatile could not be enough, let's suppose you need to read first to figure out the next value before writing something, you could end up with race conditions problems inside this small time gap.

Enough recap let's take a look in four *(Thread confinement, Stack confinement, Object confinement and Immutability)* different approaches to design a thread safe class. The one giving birth to that code is responsible for addressing race conditions internally, do you remember encapsulation? You apply in this case too. If you encapsulate your class correctly, you can change your race condition control if you have to without penalizing the ones using it.

There are many different scenarios we could use to exemplify race conditions situations, but let's see one by __dr kabutz__. A basic util class used to format dates.

<pre class="line-numbers language-java">
<code>public class DateFormatter {
    private final DateFormat df = new SimpleDateFormat("yyyy-MM-dd");

    public String format(Date date) {
        return df.format(date);
    }

    public Date parse(String date) throws ParseException {
        return df.parse(date);
    }
}
</code></pre>

### Thread confinement

If we execute this code `1_000_000` against `4` threads, what do we have?

`java.util.concurrent.ExecutionException: java.lang.NumberFormatException: multiple points`

It looks like `DateFormat` is not thread-safe. We could use `ThreadLocal` to confine our `SimpleDateFormat` into the current running thread.

<pre class="line-numbers language-java">
<code>@ThreadSafe
public class DateFormatter {

   private static final ThreadLocal<DateFormat> threadLocal = ThreadLocal.withInitial(
         () -> new SimpleDateFormat("yyyy-MM-dd")
   );

   public String format(Date date) {
      return threadLocal.get().format(date);
   }

   public Date parse(String date) throws ParseException {
      return threadLocal.get().parse(date);
   }
}
</code></pre>

Now we are safe, and the GC log looks like below.

<pre class="line-numbers language-text">
<code>time = 2038
durationOfLog=PT2.161S
numberOfGCs=12
numberOfYoungGCs=12
numberOfOldGCs=0
memoryReclaimedDuringYoung=3.562GB
memoryReclaimedDuringOld=0.000B
maxHeapAfterGC=1.484MB
totalMemoryAllocated=3.563GB
averageCreationRate=1.65GB/s
timeInGCs=PT0.015351S
timeInYoungGCs=PT0.015351S
averageTimeInYoungGCs=DoubleSummaryStatistics{count=12, sum=0.015351, min=0.000543, average=0.001279, max=0.002282}
timeInOldGCs=PT0S
percentageOfTimeInGC=0.71%
</code></pre>

All right, I may be mistaken but internally `ThreadLocal` keeps a static class called `ThreadLocalMap` which keeps an `Entry[]` called table where `Entry` extends `WeakReference`. Again, I'm not sure of what I'm about to say, but I remember of posts where people were pointing this feature as source of problems like *if ThreadLocal maps are not cleaned up properly after a transaction, the next TransactionProcessingTask might inherit values from another previous, unrelated task* and there are even articles about of [how to clean ThreadLocals](https://www.javaspecialists.eu/archive/Issue229.html) so be careful.


### Stack confinement

Another solution would be stack confinement, do you remember the first image above? If not please take another look. What about this:
>The thread stack, it holds local variables and partial results, and plays a part in method invocation and return

<pre class="line-numbers language-java">
<code>@ThreadSafe
public class DateFormatter {

   public String format(Date date) {
      final DateFormat df = getDateFormat();
      return df.format(date);
   }

   public Date parse(String date) throws ParseException {
      final DateFormat df = getDateFormat();
      return df.parse(date);
   }

   private DateFormat getDateFormat() {
      return new SimpleDateFormat("yyyy-MM-dd");
   }
}
</code></pre>

Now we have local variables defined when method invocations happen, meaning that our `DateFormat` will live inside the thread stack and is not leaking to the heap. 

And this is how the GC log looks now.

<pre class="line-numbers language-text">
<code>time = 4599
durationOfLog=PT4.962S
numberOfGCs=63
numberOfYoungGCs=63
numberOfOldGCs=0
memoryReclaimedDuringYoung=15.573GB
memoryReclaimedDuringOld=0.000B
maxHeapAfterGC=1.508MB
totalMemoryAllocated=15.574GB
averageCreationRate=3.14GB/s
timeInGCs=PT0.0453345S
timeInYoungGCs=PT0.0453345S
averageTimeInYoungGCs=DoubleSummaryStatistics{count=63, sum=0.045335, min=0.000501, average=0.000720, max=0.002288}
timeInOldGCs=PT0S
percentageOfTimeInGC=0.91%
</code></pre>

Again, `1_000_000` rounds against `4` lovable threads. 

Easier but twice slower and the total memory allocated was quite higher too. But still, safe and easy.

### Object confinement  

Another option is the object confinement, here we're declaring `DateFormat` as a private final field and *synchronizing* the access to it avoiding two+ threads to have access to it at the same time.

<pre class="line-numbers language-java">
<code>@ThreadSafe
public class DateFormatter {
   @GuardedBy("this")
   private final DateFormat df = new SimpleDateFormat("yyyy-MM-dd");

   public synchronized String format(Date date) {
      return df.format(date);
   }

   public synchronized Date parse(String date) throws ParseException {
      return df.parse(date);
   }
}
</code></pre>

Let's see the GC log result:

<pre class="line-numbers language-text">
<code>time = 7307
durationOfLog=PT7.61S
numberOfGCs=98
numberOfYoungGCs=98
numberOfOldGCs=0
memoryReclaimedDuringYoung=4.124GB
memoryReclaimedDuringOld=0.000B
maxHeapAfterGC=1.492MB
totalMemoryAllocated=4.125GB
averageCreationRate=555.06MB/s
timeInGCs=PT0.0619821S
timeInYoungGCs=PT0.0619821S
averageTimeInYoungGCs=DoubleSummaryStatistics{count=98, sum=0.061982, min=0.000455, average=0.000632, max=0.002529}
timeInOldGCs=PT0S
percentageOfTimeInGC=0.81%
</code></pre>

All right, this approach spare some memory allocation, but the execution was even worse than *stack confinement*. It may be because we're introducing some contention *synchronizing* these methods.

### Immutability

This one will look a bit unfair because address the specific problem related to the `Date` subject but you can abstract it and think about immutability applied to any other scenario.

A new date API landed with Java 8, a useful class here would be `DateTimeFormatter` that by definition is thread-safe.
> A formatter created from a pattern can be used as many times as necessary, it is immutable and is thread-safe.

<pre class="line-numbers language-java">
<code>@ThreadSafe
public class DateFormatter {
   private static final DateTimeFormatter df = DateTimeFormatter.ISO_LOCAL_DATE;

   public String format(LocalDate date) {
      return df.format(date);
   }

   public LocalDate parse(String date) {
      return LocalDate.parse(date, df);
   }
}
</code></pre>

The GC log result once again, `1_000_000` executions and `4` threads.

<pre class="line-numbers language-text">
<code>time = 1371
durationOfLog=PT1.434S
numberOfGCs=10
numberOfYoungGCs=10
numberOfOldGCs=0
memoryReclaimedDuringYoung=1.999GB
memoryReclaimedDuringOld=0.000B
maxHeapAfterGC=1.391MB
totalMemoryAllocated=2.000GB
averageCreationRate=1.39GB/s
timeInGCs=PT0.0147572S
timeInYoungGCs=PT0.0147572S
averageTimeInYoungGCs=DoubleSummaryStatistics{count=10, sum=0.014757, min=0.000533, average=0.001476, max=0.002788}
timeInOldGCs=PT0S
percentageOfTimeInGC=1.03%
</code></pre>

What do we have here, 1.3 second and just 2G of memory allocated, not bad at all, apparently Scala people have a point. Thread-safe, quick and cheap.

### Conclusion

The conclusion here is inconclusive :) The approach of choice will probably depend on what is more important to the scenario in hand, are you short in memory, do you care more about performance or readability? Don't you care about any of these aspects and what matters to you is to deliver quick and let the problem blow in someone else's hands? __you know that there's a special place in hell for you, right?__

Depending on what you're planning to do about your afterlife or about the scenario you have in hands in this world, you should consider carefully the approach you're going to use to address thread-safe, you can make it concurrent proof but you may introduce other problems if you don't look to the options you have first.

Results summary:

```
Thread confinement
time = 2038
totalMemoryAllocated=3.563GB

Stack confinement
time = 4599
totalMemoryAllocated=15.574GB

Object confinement  
time = 7307
totalMemoryAllocated=4.125GB

Immutability
time = 1371
totalMemoryAllocated=2.000GB
```


### References
https://www.javaspecialists.eu/archive/Issue229.html
https://gabrieletolomei.wordpress.com/miscellanea/operating-systems/virtual-memory-paging-and-swapping/
https://www.geeksforgeeks.org/operating-system-difference-multitasking-multithreading-multiprocessing/
https://medium.com/@unmeshvjoshi/how-java-thread-maps-to-os-thread-e280a9fb2e06
https://dzone.com/articles/java-memory-and-cpu-monitoring-tools-and-technique
http://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html
https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.5.2
http://tutorials.jenkov.com/java-concurrency/volatile.html
