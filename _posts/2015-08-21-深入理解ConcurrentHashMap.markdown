---
layout: post
title:  "深入理解ConcurrentHashMap"
date:   2015-08-21 10:19:16 +0800
categories: java
keywords: java,hashmap,concurrenthashmap
description: 源码分析concurrenthashmap的实现原理
---
从名字就能看出来，ConcurrentHashMap是专门为并发场景而设计的，相比HashTable，ConcurrentHashMap采用了**分段锁**的设计，只有在同一个分段内才存在竞态关系，不同的分段锁之间没有锁竞争。相比于HashTable对整个Map加锁的设计，分段锁极大的提升了高并发环境下的处理能力。但同时，由于不是对整个Map加锁，也导致一些需要扫描整个Map的方法(如`size`)使用了特殊的实现，还有一些方法(比如`clear`)甚至放弃了对一致性的要求，ConcurrentHashMap是一弱一致性的。

![ConcurrentHashMap内部数据结构]({{site.baseurl}}/pic/concurrenthashmap/1.svg)

ConcurrentHashMap内部维护了一个segment数组。每个Segment称为一个分段，并且继承了ReentrantLock，所以ConcurrentHashMap的加锁粒度是基于每个段的。将数据分段存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问，实现了更好的并发性能。

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {
    final Segment<K,V>[] segments;

    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        transient volatile HashEntry<K,V>[] table;
    }

    static final class HashEntry<K,V> {
        final K key;
        final int hash;
        volatile V value;
        final HashEntry<K,V> next;
        ...
    }
}
```

<br/>

Segment类似于HashMap，内部拥有一个HashEntry数组，数组中的每个元素又是一个链表，但ConcurrentHashMap中的HashEntry相对于HashMap中的Entry有一定的差异性：HashEntry中的value被volatile修饰，这样在多线程读写过程中能够保持它的可见性；还有next被修饰为final的，不变(Immutable)和易变(Volatile)使得ConcurrentHashMap允许多个读操作并发进行，并且读操作并不需要加锁。如果使用传统的技术，如HashMap中的实现，如果允许可以在hash链的中间添加或删除元素，读操作不加锁将会是不一致的。

---

## ConcurrentHashMap的初始化

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    segmentShift = 32 - sshift;
    segmentMask = ssize - 1;
    this.segments = Segment.newArray(ssize);

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;

    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
}
```

<br/>

ConcurrentHashMap的初始化需要提供三个参数。容量initialCapacity与扩容因子loadFactor的含义与HashMap类似，这里比较重要的是第三个参数：并发度concurrencyLevel。**并发度**可以理解为程序运行时能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数，实际上就是ConcurrentHashMap中的分段的个数，即segment数组的长度。segmentMask的大小为Segment数组长度减1，segmentMask与segmentShift的作用是为了后续计算segment组数下标时使用的。

<br/>

ConcurrentHashMap默认的并发度为16。当用户设置并发度时，ConcurrentHashMap会使用大于等于该值的最小2幂指数作为实际并发度（假如用户设置并发度为17，实际并发度则为32），原因与HashMap设置容量大小为2的n次方一样，能够更高效的使用“与”操作来定位分段Segment。如果并发度设置的过小，会带来严重的锁竞争问题；如果并发度设置的过大，原本位于同一个Segment内的访问会扩散到不同的Segment中，CPU cache命中率会下降，从而引起程序性能下降。

<br/>

分段确定好以后，剩下的就是每个分段的初始化了。前面已经提到过，ConcurrentHashMap中的分段Segment其实与HashMap是非常相似的。首先根据initialCapacity与分段数算出每个分段的容量`c`，最终用于确认每个分段容量大小的变量`cap`也一样为不小于c的最小2幂指数：

```java
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;

    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
```

<br/>

最后两行循环创建每个分段，这部分基本与HashMap的初始化一样，这里做进一步介绍了。

---

## ConcurrentHashMap的put操作

```java
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    return segmentFor(hash).put(key, hash, value, false);
}

private static int hash(int h) {
    // Spread bits to regularize both segment and index locations,
    // using variant of single-word Wang/Jenkins hash.
    h += (h <<  15) ^ 0xffffcd7d;
    h ^= (h >>> 10);
    h += (h <<   3);
    h ^= (h >>>  6);
    h += (h <<   2) + (h << 14);
    return h ^ (h >>> 16);
}

final Segment<K,V> segmentFor(int hash) {
   return segments[(hash >>> segmentShift) & segmentMask];
}
```

首先根据key对象的hashCode重新计算hash值，根据hash值确定桶的位置(即segment组数的下标)。这里可能会产生一个疑问，为什么要对key对象的hashCode进行二次hash呢？这里其实是加入了高位计算，防止低位不变，高位变化时，造成的hash冲突，简单来说就是降低hash冲突的机率。

<br/>

将得到的hash值向右按位移动segmentShift位，然后再与segmentMask做“与”运算得到segment组数的下标。
在初始化的时候我们说过segmentShift的值等于32-sshift，例如concurrencyLevel等于16，则sshift等于4，则segmentShift为28。hash值是一个32位的整数，将其向右移动28位就变成这个样子：
0000 0000 0000 0000 0000 0000 0000 xxxx，然后再用这个值与segmentMask做“与”运算，也就是取最后四位的值，这个值确定Segment的索引，即segment组数的下标。

<br/>

最后是Segment的put方法：

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        int c = count;
        if (c++ > threshold) // ensure capacity
            rehash();
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue;
        if (e != null) {
            oldValue = e.value;
            if (!onlyIfAbsent)
                e.value = value;
        }
        else {
            oldValue = null;
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

除了对当前的桶进行加锁之外，Segment的put方法与HashMap的put方法基本是一样的，还有就是在扩容的时候有一点区别：Segment类的rehash方法在新的table初始化完成之前，并不会修改老的table，以保证其它线程能够进行读访问。

---

## ConcurrentHashMap的get操作

```java
public V get(Object key) {
    int hash = hash(key.hashCode());
    return segmentFor(hash).get(key, hash);
}

// Segment类的get方法
V get(Object key, int hash) {
    if (count != 0) { // read-volatile
        HashEntry<K,V> e = getFirst(hash);
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                V v = e.value;
                if (v != null)
                    return v;
                return readValueUnderLock(e); // recheck
            }
            e = e.next;
        }
    }
    return null;
}
```

同样的，get方法先根据key对象的hashCode重新计算hash值，根据hash值确定桶的位置，再把真正的get操作委托给Segment类的get方法。从上面可以看出，ConcurrentHashMap的get操作是不需要加锁的（如果value为null，会调用readValueUnderLock，只有这个步骤会加锁），通过前面提到的volatile和final来确保数据安全。

---

## ConcurrentHashMap的size方法

```java
public int size() {
        final Segment<K,V>[] segments = this.segments;
        long sum = 0;
        long check = 0;
        int[] mc = new int[segments.length];
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        for (int k = 0; k < RETRIES_BEFORE_LOCK; ++k) {
            check = 0;
            sum = 0;
            int mcsum = 0;
            for (int i = 0; i < segments.length; ++i) {
                sum += segments[i].count;
                mcsum += mc[i] = segments[i].modCount;
            }
            if (mcsum != 0) {
                for (int i = 0; i < segments.length; ++i) {
                    check += segments[i].count;
                    if (mc[i] != segments[i].modCount) {
                        check = -1; // force retry
                        break;
                    }
                }
            }
            if (check == sum)
                break;
        }
        if (check != sum) { // Resort to locking all segments
            sum = 0;
            for (int i = 0; i < segments.length; ++i)
                segments[i].lock();
            for (int i = 0; i < segments.length; ++i)
                sum += segments[i].count;
            for (int i = 0; i < segments.length; ++i)
                segments[i].unlock();
        }
        if (sum > Integer.MAX_VALUE)
            return Integer.MAX_VALUE;
        else
            return (int)sum;
    }
```

size操作与put和get操作最大的区别在于，size操作需要遍历所有的Segments才能算出整个Map的大小，而put和get都只关心一个Segment。假设我们在遍历一个Segment时，别的线程有可能会修改其它的Segment，于是这一次计算出来的size值可能并不是Map当前的真正大小。

<br/>

一个比较简单的办法就是计算Map大小的时候将所有的Segment都Lock住，计算完之后再Unlock。这是普通人能够想到的方案，但是ConcurrentHashMap用了一个很好的点子：先在不加锁的情况下进行两次测试，遍历所有的Segments，累加各个Segment的大小得到整个Map的大小，如果两次测试获取的所有Segment的更新的次数（modCount，每个Segment都有一个modCount变量，这个变量在Segment中的Entry被修改时会加一，通过这个值可以得到每个Segment的更新操作的次数）是一样的，说明计算过程中没有更新操作，则直接返回这个值。如果这两次不加锁的计算过程中Map的更新次数有变化，则之后的计算先对所有的Segments加锁，再遍历所有Segments计算Map大小，最后再解锁所有Segments。


---

## ConcurrentHashMap的clear操作

```java
public void clear() {
    for (int i = 0; i < segments.length; ++i)
        segments[i].clear();
}

// Segment类的clear方法
void clear() {
    if (count != 0) {
        lock();
        try {
            HashEntry<K,V>[] tab = table;
            for (int i = 0; i < tab.length ; i++)
                tab[i] = null;
            ++modCount;
            count = 0; // write-volatile
        } finally {
            unlock();
        }
    }
}
```

<br/>

clear方法并没有采用特别的处理，循环处理每个桶，因为没有全局的锁，在清除完一个Segment之后，正在清理下一个Segment的时候，已经清理Segment可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。因此，clear方法是弱一致的。
