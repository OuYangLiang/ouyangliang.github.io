---
layout: post
title:  "深入理解ThreadLocal"
date:   2016-09-13 20:19:16 +0800
categories: java
keywords: java,threadlocal,线程
description: 介绍Java ThreadLocal的实现
---
ThreadLocal类，字面含义是线程局部变量，它为多线程的并发问题提供了一种新的思路：当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。从线程的角度看，目标变量就象是线程的本地变量，这也是类名中Local所要表达的意思。

<br/>

在第一次接触ThreadLocal类，了解它的作用后，可能会认为到它的内部可能维护了一个key为Thread的Map，但ThreadLocal的实现却比我们想象中要稍微复杂一些，让我们来一步步解开它神秘的面纱。

<br/>

ThreadLocal内部有一个静态的内部类：ThreadLocalMap，这与我们开始的猜想是一样的，只不过这个Map的key不是Thread，而是ThreadLocal。ThreadLocalMap.Entry又是ThreadLocalMap的一个静态内部类，并且继承了WeakReference类，说明ThreadLocalMap.Entry是一个弱引用，如下所示：

```java
public class ThreadLocal<T> {
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }

        private Entry[] table;
    }
}
```

<br/>

如果你熟悉HashMap的实现，那不难理解这里的Entry其实就是用来存放ThreadLocalMap中的键值对的。与HashMap的Entry不同的是ThreadLocalMap.Entry只接受类型为ThreadLocal的key，并且它继承了WeakReference，是一个弱引用，至于为什么，后文会详细说明。另一个不同点是HashMap的Entry是一个链表结构，而ThreadLocalMap.Entry却不是，难道ThreadLocalMap不会发生Hash冲突吗？不是的，ThreadLocal采用了另外的方法来解决冲突的，后文会详细说明。

<br/>

与我们猜想不同的另一点是ThreadLocalMap并没有存储在ThreadLocal中，它只是在ThreadLocal类中定义。而事际上，Thread类的内部有一个类型为ThreadLocalMap的成员变量`threadLocals`，如下所示：

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

<br/>

所以，ThreadLocal类并没有与Thread、ThreadLocalMap等类有直接的聚合关系。相关的类图如下：

![ThreadLocal相关类图]({{site.baseurl}}/pic/threadlocal/1.svg)

---

## ThreadLocal.set方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

<br/>

前两行代码很简单，得到当前线程的ThreadLocalMap。根据ThreadLocalMap是否为空来决定进行修改还是创建一个带有初始值的ThreadLocalMap。我们先看一下createMap方法：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

// ThreadLocalMap的构造方法
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

<br/>

如果你熟悉HashMap，那上面的代码很容易理解。这里的重点是ThreadLocal类的threadLocalHashCode，它在这里用作计算table的下标位置。ThreadLocalMap不像Map那样支持任意类型的key，它只接受ThreadLocal作为key，所以没有必须要像普通Map那样通过hashCode来计算。ThreadLocal采用了很简单的办法来保证每一个ThreadLocal的实例都有一个不同的threadLocalHashCode值，在初始化阶段就已经准备好，不需要每次调用hashcode方法来计算。

```java
public class ThreadLocal<T> {
    private static final int HASH_INCREMENT = 0x61c88647;
    private static AtomicInteger nextHashCode = new AtomicInteger();

    private final int threadLocalHashCode = nextHashCode();
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
}
```

<br/>

我们再接着研究ThreadLocalMap的set方法：

```java
private void set(ThreadLocal key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);//标识 1

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal k = e.get();

        if (k == key) { // 标识 2
            e.value = value;
            return;
        }

        if (k == null) { //标识 3
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value); //标识 4
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

<br/>

从标识1处可知，ThreadLocal.threadLocalHashCode同样用来确定table数组的下标位置。set方法中最难理解的部分是在for循环中。根据下标`i`得到对应的Entry对象，如果这个Entry对象为空，则不会进行for循环，直接在对应的下标位置创建一个新的Entry对象（标识4）。最后两行是判断是否需要对table数组进行扩容的。

<br/>

根据下标`i`得到对应的Entry对象如果不为空，则需要进一步比较该Entry对象与入参ThreadLocal是否为同一对象，如果是同一对象，则直接进行覆盖（标识2)。这种情况由于没有增加Entry元素，因而不需要扩容，直接reture返回。

<br/>

根据下标`i`得到对应的Entry对象如果不为空，并且与入参ThreadLocal不是同一对象，并且Entry中的key非空的话，则将下标`i`的值加1（说加1其实不准确，其实是在下标未超出table数组长度下加1，否则置为0）再次进行判断。这种情况相当于产生了hash冲突，ThreadLocalMap并没有采用像HashMap那样的链表结构来解决冲突问题，而是简单的遍历空位，找到一个最近的、可用的下标位置来存放内容。

<br/>

根据下标`i`得到对应的Entry对象如果不为空，并且与入参ThreadLocal不是同一对象，并且Entry中的key为空的，说明某个ThreadLocal对象不可达，JVM已经回收了它，这时可以重新使用这个下标位置来存放当前的值（标识3）。还记得ThreadLocalMap.Entry继承了WeakReference吗？

![ThreadLocal JVM视图]({{site.baseurl}}/pic/threadlocal/2.svg)

上面是关于ThreadLocal在JVM栈和堆空间的一个视图。ThreadLocal的设计涉及两个很重要的因素：

1. 要求线程执行结束后，通过ThreadLocal存储的内容可以被正常的GC回收。从上图可知，当线程结束后，ThreadLocalMap不可达，ThreadLocalMap中的Entry也不可达，从而Entry中的value是可以正常被GC的。这点与Entry是弱引用其实没有关系。

2. 要求当ThreadLocal不再使用时（即上图中ThreadLocalRef对ThreadLocal的引用失效后），即便引用它的线程还在继续执行，它也可以被正常的GC回收。试想一下，如果Entry使用强引用，那么当ThreadLocalRef对ThreadLocal的引用不可达时，ThreadLocalMap还持有ThreadLocal的强引用，在线程未结束之前，ThreadLocal将无法被正常的回收。所以ThreadLocalMap.Entry实现采用了弱引用的方式。当ThreadLocalRef对ThreadLocal的引用不可达时，JVM可以安全的回收ThreadLocal对象，之后ThreadLocalMap.Entry的get方法会直接返回null，标识3所对应的场景就是这种情况。

---

## ThreadLocal.get方法

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}
```

<br/>

我们再来看看ThreadLocal的set方法，首先获得当前线程的ThreadLocalMap，如果map为空的话，则调用setInitialValue设计初始值。我们来看看setInitialValue的实现：

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

<br/>

在使用ThreadLocal时，我们可以提供一个子类去这实现initialValue方法，通过该方法可以指定一个初始值，从源码可以看到，默认的初始值是null。

<br/>

我们再接着看看ThreadLocalMap的getEntry方法：

```java
private Entry getEntry(ThreadLocal key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal k = e.get();
        if (k == key)
            return e;
        if (k == null) // 标识 5
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

<br/>

如果你已经掌握了ThreadLocalMap的set方法，那理解getEntry是很容易的。当下标i对应的位置不为空，且与入参ThreadLocal是同一个对象时，直接返回该位置的Entry。否则跳转到getEntryAfterMiss方法，采用遍历table数组的方式继续查询是否存在满足条件的Entry。标识5表示某个ThreadLocal对象已经被GC回收了，这里将清除对应的Entry所占有的位置。

---

## ThreadLocal.remove方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

// ThreadLocalMap的remove方法
private void remove(ThreadLocal key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

<br/>

至此，你应该已经掌握了ThreadLocalMap解决冲突的方式：根据ThreadLocal.threadLocalHashCode与table数组长度减1的值进行“与”操作获得起始下标位置，然后从起始位置开始遍历寻找一个可用的空位。remove方法也是采用遍历的方式定位入参ThreadLocal在当前线程ThreadLocalMap中的Entry，并调用Reference.clear方法及expungeStaleEntry方法进行空位清除。

---

## ThreadLocal内存泄漏的问题

通过ThreadLocal在JVM栈和堆空间的视图中，我们很容易明白保存在ThreadLocalMap中的对象（someObj）与ThreadLocal本身并没有直接联系。JVM能够正常的回收someObj的途径只有两个：

1. 调用ThreadLocal.remove方法。
2. 对应的线程执行结束，从而ThreadLocalMap至Entry、Entry至someObj的引用链失效。

<br/>

但是在一些Web环境中，线程可能维护在一个容器内部的线程池，它们的生命周期可能大于具体的Web应用。保存在线程ThreadLocalMap中的someObj对象，如果是第三方类加载器（比如tomcat的WebClassLoader）加载的，而我们没有显式的通过ThreadLocal.remove方法移的话，这些someObj对象将无法被GC回收。如果你熟悉java类加载机制的话，就会发现加载这些someObj的类加载器也将无法被回收，从而会导致永久代的溢出，即java.lang.OutOfMemoryError: PermGen space异常。所以很重要的一点，如果我们使用了ThreadLocal，务必使用remove方法进行清除。

<br/>

我曾经看到阿里有人在工具类中使用ThreadLocal来解决SimpleDateFormat线程不安全的问题，当时由于没有发现显示的使用remove方法，错误的认为这会导致内存泄漏的问题。而事实上，这么使用是正确的，并不会出现内存泄漏的问题，因为其实也很简单，因为SimpleDateFormat是jdk的类，它是由AppClassLoader加载的，而非第三方实现的类加载器加载的，所以不会导致第三方类加载器无法被回收的问题。
