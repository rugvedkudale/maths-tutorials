---
layout: post
title:  "Java Spliterator"
date:   2021-08-26
categories: java
tags: core-java
author: pradale
description: The Spliterator interface was introduce in java 8. This can be used for traversing and partitioning elements of source. The source could be anything like an array, a Collection, an IO channel, or a generator function.
---

The Spliterator interface was introduce in java 8. This can be used for traversing and partitioning elements of source. The source could be anything like an array, a Collection, an IO channel, or a generator function.

### Why it was introduced in java 8
Spliterator was added to support parallel processing. As its name suggests, Spliterator combines two behaviors: iterating the elements of the source, and splitting the input source for parallel execution.

#### Important methods in Spliterator

##### tryAdvance(Consumer<? super T> action)
This method takes **Consumer** as argument and performs the given action on element. If no elements are present, tryAdvance() will return false.

##### trySplit()
The trySplit() method will split the elements into two sections of similar size (in most cases). 
The two new sections will be new instance of Spliterator. if the source can't be split it will return null.  

#### forEachRemaining(Consumer<? super T> action)
This method takes **Consumer** as argument and performs the given action on all the elements.

#### characteristics()
Returns a set of characteristics of this Spliterator and its elements. The result is represented as values from **ORDERED**, **DISTINCT**, **SORTED**, **SIZED**, **NONNULL**, **IMMUTABLE**, **CONCURRENT**, **SUBSIZED**.

* SIZED Capable of returning the exact number of elements in the source when we invoke estimateSize() method
* SUBSIZED When we split the instance using trySplit() and obtain SIZED SplitIterators as well
* ORDERED Iterating over an ordered sequence
* SORTED Iterating over a sorted sequence
* NONNULL Source guarantees to have not null values
* DISTINCT No duplicates exist in our source sequence
* IMMUTABLE If we can’t structurally modify the element source
* CONCURRENT The element source can be safely concurrently modified

### Why use Spliterator?
In most cases you will not directly deal with Spliterator unless you are writing some custom collection and want to optimize parallelized operations on all elements. 

Some advantages are:

* It Supports parallel processing of elements.
* tryAdvance() method combines both hasNext/next operations a single method.
* Spliterator also is a smarter Iterator because of its internal properties like DISTINCT or SORTED, etc


### Examples

#### Splitting the list
Here is the example how the list is divided into two sections which can be iterated dependently.

```java
List<String> list = new ArrayList<>();

for (int i = 0; i < 10; i++) {
    list.add("String-" + i);
}

Spliterator<String> spliterator1 = list.spliterator();
Spliterator<String> spliterator2 = spliterator1.trySplit();

System.out.println("===1st List===");
spliterator1.forEachRemaining(System.out::println);

System.out.println("===2nd List===");
spliterator2.forEachRemaining(System.out::println);
```

```output
===1st List===
String-5
String-6
String-7
String-8
String-9
===2nd List===
String-0
String-1
String-2
String-3
String-4
```

#### Custom Implementation for Spliterator
Below is the sample implementation just for understanding the concept. obvious it will not be this simple.

```java
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.Spliterator;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Consumer;

@Slf4j
public class SpliteratorSample implements Spliterator<String> {

    private final List<String> list;
    AtomicInteger current = new AtomicInteger();

    public SpliteratorSample(List<String> list) {
        this.list = list;
    }

    @Override
    public boolean tryAdvance(Consumer<? super String> action) {
        log.info("Executing - " + Thread.currentThread().getName());
        action.accept(list.get(current.getAndIncrement()));
        return current.get() < list.size();
    }

    @Override
    public Spliterator<String> trySplit() {
        log.info("Splitting - " + Thread.currentThread().getName());

        // Current size of list
        int size = list.size() - current.get();

        if(size < 2) {
            // No need split if the list is small
            return null;
        }else {
            // Try to divide list in half
            for (int position = size / 2 + current.intValue(); position < list.size(); position++) {
                if (list.get(position).length() > 0) {
                    Spliterator spliterator = new SpliteratorSample(list.subList(current.get(), position));
                    current.set(position);
                    return spliterator;
                }
            }
        }
        return null;
    }

    @Override
    public long estimateSize() {
        return list.size() - current.get();
    }

    @Override
    public int characteristics() {
        return CONCURRENT;
    }
}
```

Sequential execution

```java
List<String> sList = new ArrayList<>();

for (int i = 5; i > 0; i--) {
sList.add("Test-String" + i);
}

SpliteratorSample spliteratorSample = new SpliteratorSample(sList);

Stream<String> stream = StreamSupport.stream(spliteratorSample, false);
stream.forEach(System.out::println);
```

Output

```output
20:56:23.108 [main] INFO org.pradale.SpliteratorSample - Executing - main
Test-String5
20:56:23.111 [main] INFO org.pradale.SpliteratorSample - Executing - main
Test-String4
20:56:23.111 [main] INFO org.pradale.SpliteratorSample - Executing - main
Test-String3
20:56:23.111 [main] INFO org.pradale.SpliteratorSample - Executing - main
Test-String2
20:56:23.111 [main] INFO org.pradale.SpliteratorSample - Executing - main
Test-String1
```

Parallel execution
```java
List<String> sList = new ArrayList<>();

for (int i = 5; i > 0; i--) {
sList.add("Test-String" + i);
}

SpliteratorSample spliteratorSample = new SpliteratorSample(sList);

Stream<String> stream = StreamSupport.stream(spliteratorSample, true);
stream.forEach(System.out::println);
```

Output: You can see the logs from different threads. which shows it's running in parallel.

```output
20:54:43.019 [main] INFO org.pradale.SpliteratorSample - Splitting - main
20:54:43.022 [main] INFO org.pradale.SpliteratorSample - Splitting - main
20:54:43.022 [main] INFO org.pradale.SpliteratorSample - Executing - main
Test-String3
20:54:43.022 [ForkJoinPool.commonPool-worker-1] INFO org.pradale.SpliteratorSample - Splitting - ForkJoinPool.commonPool-worker-1
20:54:43.022 [ForkJoinPool.commonPool-worker-2] INFO org.pradale.SpliteratorSample - Splitting - ForkJoinPool.commonPool-worker-2
20:54:43.023 [ForkJoinPool.commonPool-worker-2] INFO org.pradale.SpliteratorSample - Executing - ForkJoinPool.commonPool-worker-2
20:54:43.023 [ForkJoinPool.commonPool-worker-3] INFO org.pradale.SpliteratorSample - Executing - ForkJoinPool.commonPool-worker-3
Test-String1
20:54:43.023 [ForkJoinPool.commonPool-worker-1] INFO org.pradale.SpliteratorSample - Executing - ForkJoinPool.commonPool-worker-1
Test-String5
Test-String4
20:54:43.023 [ForkJoinPool.commonPool-worker-2] INFO org.pradale.SpliteratorSample - Executing - ForkJoinPool.commonPool-worker-2
Test-String2
```
