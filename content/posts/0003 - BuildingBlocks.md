+++
title = "BuildingBlocks"
description = ""
tags = [
    "java",
    "core java",
]
date = "2015-02-02"
categories = [
    "software",
    "engineering",
]
draft = false
+++

Today I would like to write a bit about Java Collections. I believe most of us that already played with Java know about the primary and most used classes in the [Java Collections framework](https://docs.oracle.com/javase/tutorial/collections/), at least ones that ever had an interview are familiar with, they are part of the default pack of questions that interviewers ask to check if you at least read about Java.

But this post does not intend to talk about the different aspects of data structures, or when to use A or B, the idea is to speak briefly about collections in a concurrent context.

There are some aspects of Java that were real reasons for concern in the past and still causing *skepticism* nowadays, example EJBs, reflection or concurrency in general. Who does not have that strange feeling when someone mention Vector, HashTable or StringBuffer? And who do not transfer those mixed feelings to `Concurrent<Something>Set,Map,Queue,Deque...` for example?

Something got lost in the way about concurrency. `Vector, HashTable and StringBuffer` are __`synchronized`__ not __`concurrent`__. There are differences.

### Regular collections
I'll assume that we don't need to spend much time over here, you probably have been playing with `HashSet, ArrayList, LinkedList and HashMap` at some point. These classes are just great and probably another way to verify Pareto's principle in the Java context.

So if you know you're not sharing your collection at any moment, use them and be happy. By sharing, I mean something around [this](https://allandequeiroz.com/thread-safety/).

### Synchronized collections

Now we got to the point that is probably one of the causes behind all the mysticism around race condition control slowness, __contention__. But what could cause contention to classes such as `Vector or HashTable`? To answer that let's take a look into `Collections.synchronizedList` and see what happen when we use synchronized collections.

<pre class="line-numbers language-java">
<code>// The first step is a simple check to verify if the provided List
// is an instance of RandomAccess and decide who is wrapping it.
public static <T > List < T > synchronizedList(List < T > list) {
   return (list instanceof RandomAccess ?
         new SynchronizedRandomAccessList<>(list) :
         new SynchronizedList<>(list));
}

// Now the SynchronizedList constructor receives our List and keeps 
// an internal reference to it, just notice that SynchronizedList is 
// passing this list to it's parent SynchronizedCollection, another 
// static inner class inside Collections
SynchronizedList(List < E > list) {
   super(list);
   this.list = list;
}

// This is how SynchronizedCollection's constructor looks like, once
// again it keeps an internal reference, but the important part is the
// mutex that will be used to synchronize almost all methods around the
// original List.
SynchronizedCollection(Collection < E > c) {
   this.c = Objects.requireNonNull(c);
   mutex = this;
}

// What's the mutex for

public boolean add (E e){
   synchronized (mutex) {
      return c.add(e);
   }
}

public E get (int index){
   synchronized (mutex) {
      return list.get(index);
   }
}

public boolean removeAll (Collection < ? > coll){
   synchronized (mutex) {
      return c.removeAll(coll);
   }
}
</code></pre>

As you can see, except for `spliterators` and `streams` related methods, all the other ones are synchronized this way, blocking entire methods from outside using the same object to lock everything, this is thread safe but when a thread is holding this lock, doesn't matter what's going on, no one else will perform anything over this Collection. And the mechanism is similar for the other synchronized collections too. If you use one of the methods below, this is how you're protecting your collection against race conditions.

<pre class="line-numbers language-java">
<code>Collections.synchronizedList(List<T> list)
Collections.synchronizedCollection(Collection<T> c)
Collections.synchronizedMap(Map<K,V> m)
Collections.synchronizedNavigableMap(NavigableMap<K,V> m)
Collections.synchronizedNavigableSet(NavigableSet<T> s)
Collections.synchronizedSet(Set<T> s)
Collections.synchronizedSortedMap(SortedMap<K,V> m)
Collections.synchronizedSortedSet(SortedSet<T> s)
</code></pre>

But that's all right. It doesn't mean you'll have an awful performance just because you're using it. Then when should we use a synchronized collection? 

- If the collection is shared, but access is not too frequent, we're safe to use it.

Just remember if you're using large collections it won't scale that well and even using a synchronized collection still necessary to provide additional locking to guard compound actions, e.g.:

- Iteration
- Navigation
- Conditional operations - putIfAbsent


### Concurrent collections

Concurrent collections are specific versions designed with concurrency in mind by design, instead of using a single collection-wide lock it uses the concept of segmentation supported by *(internal data structures)*.

Let's use `ConcurrentHashMap` as an example, segments *(see upfront)* will be locked just during updating operations and even during these operations, synchronization will happen in specific moments, is almost surgical. *Take a look for example into __ConcurrentHashMap.putVal__*.

One of `ConcurrentHashMap's` inner classes is `Segment` kept in a bucket table that holds chunks of this HashMap, this way different threads can operate in different segments reducing contention.

<pre class="line-numbers language-java">
<code>Segment&#x3C;K,V&#x3E;[] segments = (Segment&#x3C;K,V&#x3E;[])
            new Segment&#x3C;?,?&#x3E;[DEFAULT_CONCURRENCY_LEVEL];
</code></pre>

`DEFAULT_CONCURRENCY_LEVEL` is the "expected" number of concurrently updating threads operating over this Map, by default setted to __`16`__ meaning that during high concurrency moments at least `16` threads can operate at the same time over this Map *(We'll have 16 locks, each one guarding each segment or bucket if you prefer)*.

If you take a look at the [ConcurrentHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html#ConcurrentHashMap-int-float-int-) documentation you'll see that we can change the default values to something more realistic for the scenario in hand, apart of `concurrencyLevel` is possible to specify `loadFactor` that tells how much the map should grow in case it has to, by default the growth factor is __`0.75%`__. The `initialCapacity`, if we have an idea of how big will be the Map, we can avoid the resizing overhead using this parameter, the default again is __`16`__.

At this point, we know that internally the `ConcurrentHashMap` breaks it's content in segments so threads can work in different parts of it simultaneously without interference. 

All right then, but how does it know where to go *which segment* when reading or writing content after breaking it up? 

Based on the `key` hash calc `int hash = spread(key.hashCode())` the bucket is identified, created or resized than the new `Node<k,v>` is inserted. At this point *the insertion*, we finally have a `synchronized` block.

With this notion about the internal complexity difference between a `synchronized` and a `concurrent` collection, you may are wondering when to use the concurrent option.

The answer is if the collection will be shared frequently and accessed by multiple threads, for sure a concurrent collection would be handy. Just remember, it might use more memory, especially `ConcurrentHashMap` why? To support all this mechanism, each `Segment` is a `ReentrantLock`, internally `ReentrantLock` maintain an inner class called `Sync` that extends `AbstractQueuedSynchronizer` that holds a queue of nodes to maintain the state of the threads, ending up in few more memory usage. At this point, you can see things such as.

<pre class="line-numbers language-java">
<code>/**
 * The thread that enqueued this node.  Initialized on
 * construction and nulled out after use.
 */
volatile Thread thread;

volatile int waitStatus;

So on...
</code></pre>

Ok, I got it! I won't use it otherwise memory will blow up!!

No. Not necessarily, I've performed some silly tests between HashMap, synchronized HashMap, and ConcurrentHashMap to see the difference.

The __first round__ `1_000_000` writes and reads with a single Thread.

```
# HashMap
Time = 3.8s
totalMemoryAllocated=538.500MB

# Synchronized HashMap
Time = 3.9s
totalMemoryAllocated=540.500MB

# Concurrent HashMap
Time = 5.5s
totalMemoryAllocated=541.500MB
```

The __second round__ `1_000_000` writes and reads against four threads.

```
# HashMap
Time = 4.1s
totalMemoryAllocated=538.500MB

# Synchronized HashMap
Time = 3.6s
totalMemoryAllocated=539.500MB

# Concurrent HashMap
Time = 4.5s
totalMemoryAllocated=540.000MB
```

As you can see, not bad at all so unless you're fighting for each nanosecond you better think first about preventing wrong results and/or race condition problems.


### Bonus

If you're not happy with the alternatives provided by the Java language, there are options out there, some already well known, e.g., Google Guava or more specific one's example:

[Eclipse Collections](https://www.eclipse.org/collections/)

> Eclipse Collections is the best Java collections framework ever that brings happiness to your Java development. 

- Minimize object creation
- Work with large data sets
- Support mutable and immutable collections

[JCTools](https://github.com/JCTools/JCTools)

Supports various specialized queues, such as

- SPSC - Single Producer Single Consumer (Wait-Free, bounded and unbounded)
- MPSC - Multi-Producer Single Consumer (Lockless, bounded and unbounded)
- SPMC - Single Producer Multi-Consumer (Lockless, bounded)
- MPMC - Multi-Producer Multi-Consumer (Lockless, bounded)

[Cliff Click's High Scale Library](https://github.com/boundary/high-scale-lib)

> A collection of Concurrent and Highly Scalable Utilities. These are intended as direct replacements for the java.util.* or java.util.concurrent.* collections but with better performance when many CPUs are using the collection concurrently. Single-threaded performance may be slightly lower.

### Conclusion

That fear of certain Java mechanisms performance are gone or at least minimized, today they are real alternatives to solve practical problems and knowing them is important to avoid coding mistakes or debates using arguments from 10+ years ago. 

Knowing a bit more about the nuances of the available options is useful not to banish a tool but to choose it correctly at the proper moment.

### References


[ConcurrentHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html)

[CopyOnWriteArrayList](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CopyOnWriteArrayList.html)

[CopyOnWriteArraySet](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CopyOnWriteArraySet.html)

[ConcurrentHashMap.KeySetView](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.KeySetView.html)

[ConcurrentLinkedDeque.html](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedDeque.html)

[ConcurrentLinkedQueue.html](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedQueue.html)

[ConcurrentMap.html](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentMap.html)

[ConcurrentNavigableMap.html](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentNavigableMap.html)

[ConcurrentSkipListMap.html](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentSkipListMap.html)

[ConcurrentSkipListSet.html](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentSkipListSet.html)
