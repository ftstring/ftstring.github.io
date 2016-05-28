---
layout:	post
title: Hashtable源码实现简析
categories:	[jdk-source]
tags:	[collection, map]
fullview:	true
description: HashTable走起。
---

### Hashtable
Hashtable支持并发环境下使用，实现这一点的主要方式就是在对元素进行读取和操作时通过synchronized关键字进行加锁。

HashMap的源码分析可以参考我之前的博客[《HashMap源码实现简析》][1]。

Hashtable是JDK1.0加入的，而HashMap是JDK1.2版本加入的，看起来，HashMap有一些设计细节更值得肯定。

#### 继承结构

{% highlight java linenos %}
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
{ ... }
{% endhighlight %}

#### 构造函数

{% highlight java linenos %}
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }

    // 和HashMap的编程风格不同，Hashtable属性的默认值并没有在变量声明时给出，而是写在了程序里，
    // 从这一点来讲，个人感觉HashMap更合理一些
    public Hashtable() {
        this(11, 0.75f);
    }

    0.00public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }
{% endhighlight %}

#### 基本属性

同HashMap，HashTable定义了如下的属性：

{% highlight java linenos %}
	private transient Entry<?,?>[] table;

	private transient int count; // table中的元素数量

	private int threshold;

	private float loadFactor;

	// table结构变动时，会先更新modCount，然后在实际操作后更新count
	// 该值也用来判断iterator操作的快速失败
	private transient int modCount = 0;
{% endhighlight %}

#### 基本方法

{% highlight java linenos %}
    public synchronized int size() {
        return count;
    }

    public synchronized boolean isEmpty() {
        return count == 0;
    }

    ...
{% endhighlight %}

#### 核心方法

与[HashMap][1]不同，Hashtable不允许key和value为null，会抛出NullPointerException。

> 同时，index的计算方式也不同，看起来，Hashtable的设计者希望通过这种方式来减少冲突的可能（待验证）

{% highlight java linenos %}
	// 核心算法的实现也体现了设计者编程思想的不同
	// 跟其他基本方法一样，Hashtable的核心操作put、get等也用sychronized修饰
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) { // 与HashMap不同，Hashtable不允许value为null
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode(); // 同样的，如果key == null, 也会抛出NullPointerException
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) { // 这里的处理同HashMap
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }

    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        // 先判断是否超过threshold，需要rehash
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length; // index计算方式比较直接，对length取余
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }

    @SuppressWarnings("unchecked")
    protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
{% endhighlight %}

### 总结

这篇分析要比[《HashMap源码实现简析》][1]简洁得多，因为很多思想是一致的，不想赘述。在整理这篇笔记的同时，也发现了设计细节上一些不同，有意思的是，这两个类的实现作者基本一致。

言归正传，Hashtable和HashMap在核心设计上，都是把元素存储在Entry数组中，通过对key的两次hash（一次计算hashcode，一次取余）快速计算index，冲突的处理也都是基于链表。

最重要的不同在于，Hashtable用synchronized对函数进行了标记，使得Hashtale对于多线程操作时可兼容的。但是这种互斥锁，也必然造成了系统的同步开销的增加，之后，我会介绍ConcurrentHashMap，它以一种更细腻的方式实现了线程安全的Map。

[1]: {{ site.BASE_PATH }}/jdk-source/2016/05/27/hashmap.html "HashMap源码实现简析"