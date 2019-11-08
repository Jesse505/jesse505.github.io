---
title: LruCache原理
date: 2019-01-01 15:11:15
tags: LRU
categories: [Android,基础]
---

## 一、LRU算法介绍

LRU是最近最少使用的算法，它的核心思想是当缓存满时，会优先淘汰那些最近最少使用的缓存对象。采用LRU算法的缓存有两种：LruCache和DisLruCache，分别用于实现内存缓存和硬盘缓存，其核心思想都是LRU缓存算法。

在复习算法和数据结构的时候，我通过双向链表实现了一个LRU缓存淘汰算法，[有兴趣的请点击](https://github.com/Jesse505/algo/blob/master/day04/LRUCache.java)

## 二、LruCache的介绍

LruCache是Android 3.1所提供的一个缓存类，所以在Android中可以直接使用LruCache实现内存缓存。而DisLruCache目前在Android 还不是Android SDK的一部分，但Android官方文档推荐使用该算法来实现硬盘缓存。

LruCache是个泛型类，主要算法原理是把最近使用的对象用强引用（即我们平常使用的对象引用方式）存储在 LinkedHashMap 中。当缓存满时，把最近最少使用的对象从内存中移除，并提供了get和put方法来完成缓存的获取和添加操作。

## 三、**LinkedHashMap原理**

我们先来分析下LinkedHashMap的原理，为后面分析LruCache做准备。
LinkedHashMap 继承 HashMap，在 HashMap 的基础上进行扩展，put 方法并没有重写，说明**LinkedHashMap遵循HashMap的数组加链表的结构**，LinkedHashMap重写了 **createEntry** **()**和**get()**方法

<!--more-->

看下LinkedHashMap 的 createEntry 方法

```java
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMapEntry<K,V> old = table[bucketIndex];
        LinkedHashMapEntry<K,V> e = new LinkedHashMapEntry<>(hash, key, value, old);
        table[bucketIndex] = e;
        e.addBefore(header);
        size++;
    }
```

可以看到LinkedHashMap的数组里面放的是**LinkedHashMapEntry**对象：

```java
    private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
        // 双向链表
        LinkedHashMapEntry<K,V> before, after;

        LinkedHashMapEntry(int hash, K key, V value, HashMapEntry<K,V> next) {
            super(hash, key, value, next);
        }
      
				//移除链表的节点
        private void remove() {
            before.after = after;
            after.before = before;
        }

        //添加到header节点的前面		
        private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        //移除节点并添加到header节点的前面
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }
				
      	//就是移除节点
        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }
```

**LinkedHashMapEntry继承 HashMapEntry，添加before和after变量，所以是一个双向链表结构，还添加了addBefore和remove 方法，用于新增和删除链表节点。并且重写了recordAccess方法和recordRemoval方法**

- addBefore是将一个数据添加到Header的前面，existingEntry 传的都是链表头header，将一个节点添加到header节点前面，只需要移动链表指针即可，添加新数据都是放在链表头header 的before位置，**链表头节点header的before是最新访问的数据，header的after则是最旧的数据。**
- recordAccess方法在LinkedHashMap重写的get方法里和HashMap的put方法里都有调用，作用是在访问模式下，每次访问了节点，就把该节点放到header节点的前面。

这里放一张图，有助于理解：

<img src="/img/201901/LinkedHashMap.png" alt="LinkedHashMap" style="width: 600px;">

## 四、LruCache的实现原理

**注意：文章分析的源码是V4包下的LruCache代码。**

LruCache的核心思想很好理解，就是要维护一个缓存对象列表，其中对象列表的排列方式是按照访问顺序实现的，即一直没访问的对象，将放在队尾，即将被淘汰。而最近访问的对象将放在队头，最后被淘汰。这样的存储规则LinkedHashMap正好可以提供。

通过下面构造函数来指定LinkedHashMap中双向链表的结构是访问顺序还是插入顺序。

```java
public LinkedHashMap(int initialCapacity,
	float loadFactor,
	boolean accessOrder) {
	super(initialCapacity, loadFactor);
	this.accessOrder = accessOrder;
}
```

其中accessOrder设置为true则为访问顺序，为false，则为插入顺序。

下面我们在LruCache源码中具体看看，怎么应用LinkedHashMap来实现缓存的添加，获得和删除的。首先来看构造方法：

```java
public LruCache(int maxSize) {
	if (maxSize <= 0) {
		throw new IllegalArgumentException("maxSize <= 0");
	}
	this.maxSize = maxSize;
	this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```

构造方法很简单，就是设置了缓存的最大值，新建了一个LinkedHashMap对象，并且设置为访问顺序。

### put()方法

```java
public final V put(K key, V value) {
         //不可为空，否则抛出异常
	if (key == null || value == null) {
		throw new NullPointerException("key == null || value == null");
	}
	V previous;
	synchronized (this) {
            //插入的缓存对象值加1
		putCount++;
            //增加已有缓存的大小
		size += safeSizeOf(key, value);
           //向map中加入缓存对象
		previous = map.put(key, value);
            //如果已有缓存对象，则缓存大小恢复到之前
		if (previous != null) {
			size -= safeSizeOf(key, previous);
		}
	}    
	if (previous != null) {
	   //是个空方法，可以自行实现
		entryRemoved(false, key, previous, value);
	}
        //调整缓存大小
	trimToSize(maxSize);
	return previous;
}
```

可以看到put方法就是调用Map的put方法，记录当前缓存大小，我们接着看trimToSize()方法

### trimToSize()方法

```java
public void trimToSize(int maxSize) {
    //死循环
	while (true) {
		K key;
		V value;
		synchronized (this) {
            //如果map为空并且缓存size不等于0或者缓存size小于0，抛出异常
			if (size < 0 || (map.isEmpty() && size != 0)) {
				throw new IllegalStateException(getClass().getName()
					+ ".sizeOf() is reporting inconsistent results!");
			}
            //如果缓存大小size小于最大缓存，或者map为空，不需要再删除缓存对象，跳出循环
			if (size <= maxSize || map.isEmpty()) {
				break;
			}
            //获取最旧的元素
			Map.Entry<K, V> toEvict = map.eldest();
      if (toEvict == null) {
          break;
      }
			key = toEvict.getKey();
			value = toEvict.getValue();
            //删除该对象，并更新缓存大小
			map.remove(key);
			size -= safeSizeOf(key, value);
			evictionCount++;
		}
		entryRemoved(true, key, value, null);
	}
}
```

可以看到trimToSize方法是在一个循环里调用，获取最旧的对象并删除，直到当前缓存大小小于等于最大缓存大小，才跳出循环。我们来看看map.eldest()是如何获取最旧的对象的：

```java
    public Map.Entry<K, V> eldest() {
        Entry<K, V> eldest = header.after;
        return eldest != header ? eldest : null;
    }
```

可以看到返回的就是LinkedHashMap中header的after节点，其实就是最旧的对象。

### get()方法

```java
public final V get(K key) {
        //key为空抛出异常
	if (key == null) {
		throw new NullPointerException("key == null");
	}

	V mapValue;
	synchronized (this) {
            //获取对应的缓存对象
		mapValue = map.get(key);
		if (mapValue != null) {
			hitCount++;
			return mapValue;
		}
		missCount++;
	}
```

get()方法也很简单，这里就不多说了，其实LruCache的中get()方法和put()方法只是简单的调用了LinkedHashMap里的方法，真正的实现是LinkedHashMap中recordAccess方法，recordAccess方法在put和get方法中都有被调用过，作用是在每次put或者get后都把该对象变放在队头，变为最新的数据，这个在上一节讲过。

## 四、总结

- **由此可见LruCache中维护了一个集合LinkedHashMap，该LinkedHashMap是以访问顺序排序的。当调用put()方法时，就会在集合中添加元素，同时会更新该元素到队头，并调用trimToSize()判断缓存是否已满，如果满了就获取map中最旧的元素并删除。当调用get()方法访问缓存对象时，就会调用LinkedHashMap的get()方法获得对应集合元素，同时会更新该元素到队头。**





