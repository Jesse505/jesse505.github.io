---
title: Android高性能日志写入方案-mmap
date: 2019-06-18 22:11:15
tags: mmap
categories: [Android,日志]
---

## 前言

最近在做一个新零售的收银app，对于app稳定性要求比较高，但是难免会出现一些难以复现的问题，针对这些问题，分析日志有时候是解决问题的必要手段。下面我们主要分析下日志写入方案的实现。

## 常规方案的缺陷

- **性能问题：**一开始日志的写入就是通过标准I/O直接写文件，当有一条日志要写入的时候，首先，打开文件，然后写入日志，最后关闭文件。但是写文件是 IO 操作，随着日志量的增加，更多的IO操作，一定会造成性能瓶颈。为什么这么说呢？**因为数据从程序写入到磁盘的过程中，其实牵涉到两次数据拷贝：一次是用户空间内存拷贝到内核空间的缓存，一次是回写时内核空间的缓存到硬盘的拷贝。当发生回写时也涉及到了内核空间和用户空间频繁切换**。
- **丢日志：**为了解决性能问题，直接想到就是减少I/O操作，我们可以先把日志缓存到内存中，当达到一定数量或者在合适的时机将内存里的日志写入磁盘中。这样似乎可以减少I/O操作，但是在将内存里的日志写入磁盘的过程中，app被强杀了或者Crash了的话，这样会造成更严重的问题，日志丢失。

看到这里，难道真的就没有高性能又能保证日志完整性的方案了吗？答案是mmap。mmap是个什么鬼？我们接着往下看。

<!--more-->

## 什么是mmap

> mmap是一种内存映射文件的方法，即将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对应关系；实现这样的映射关系后，进程就可以采用指针的方式读写操作这一块内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必调用read，write等系统调用函数，相反，内核空间堆这段区域的修改也直接反应到用户空间，从而可以实现不同进程间的文件共享。

网上很多文章都说 mmap 完全绕开了页缓存机制，其实这并不正确。我们最终映射的物理内存依然在页缓存中，它可以带来的好处有：

- **减少系统调用**。我们只需要一次 mmap() 系统调用，后续所有的调用像操作内存一样，而不会出现大量的 read/write 系统调用。
- **减少数据拷贝**。普通的 read() 调用，数据需要经过两次拷贝；而 mmap 只需要从磁盘拷贝一次就可以了，并且由于做过内存映射，也不需要再拷贝回用户空间。
- **可靠性高**。mmap 把数据写入页缓存后，跟缓存 I/O 的延迟写机制一样，可以依靠内核线程定期写回磁盘。

<img src="/img/201906/mmap.jpg" alt="mmap" style="width: 600px;">

从上面的图看来，我们使用 mmap 仅仅只需要一次数据拷贝。

## mmap使用场景

mmap 比较适合于对**同一块区域频繁读写**的情况，推荐也使用线程来操作。

- 用户日志、数据上报都满足这种场景，微信开源的 mars 框架中的 [xlog](https://mp.weixin.qq.com/s/cnhuEodJGIbdodh0IxNeXQ)模块也是基于 mmap 特性实现的。
- 需要跨进程同步的时候，mmap 也是一个不错的选择，Android 跨进程通信有自己独有的 Binder 机制，它内部也是使用 mmap 实现。

## 具体实现

在Android中可以将文件通过Java提供的`MappedByteBuffer`映射到内存，然后进行读写。（微信的[xlog](https://mp.weixin.qq.com/s/cnhuEodJGIbdodh0IxNeXQ)模块mmap实现是基于C++代码实现)

`MappedByteBuffer` 位于 Java NIO 包下，用于将文件内容映射到缓冲区，使用的即是 mmap 技术。通过 FileChannel 的 `map` 方法可以创建缓冲区。

```java
RandomAccessFile raf = new RandomAccessFile(file, "rw");
//position映射文件的起始位置，size映射文件的大小
MappedByteBuffer buffer = raf.getChannel().map(FileChannel.MapMode.READ_WRITE, position, size);
//往缓冲区里写入字节数据
buffer.put(log);
```

有一点比较坑，Java 虽然提供了 map 方法，但是并没有提供 `unmap` 方法，通过 Google 得知 `unmap` 方法是有的，不过是私有的，我们可以通过反射调用获取`unmap`方法（Android 9.0以上对反射做了限制，可以参考这篇[博文](http://weishu.me/2019/03/16/another-free-reflection-above-android-p/)绕过限制）

```java
    /**
     * 解除内存与文件的映射
     * */
    private void unmap(MappedByteBuffer mbbi) {
        if (mbbi == null) {
            return;
        }
        try {
            Class<?> clazz = Class.forName("sun.nio.ch.FileChannelImpl");
            Method m = clazz.getDeclaredMethod("unmap", MappedByteBuffer.class);
            m.setAccessible(true);
            m.invoke(null, mbbi);
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }
```

在这里记录一个点，刚开始在写入文件的时候，我是使用mmap将日志直接写入文件，这样的话需要通过代码去实现动态扩容，挺麻烦的，后来发现xlog是将日志写入高速缓冲区，这块高速缓冲区是使用mmap映射出来的内存区，被映射的磁盘文件是它新建的一个缓存文件，当高速缓冲区内容写到一定阈值时，通知后台线程将缓冲区的内容写入文件。借鉴xlog的方案，后面我们也改为**先写入缓存中，当写满了之后，再flush到目标文件，也可以手动调用flush，将缓存刷新到目标文件**。

至于微信为什么这么做，肯定也是出于性能的原因啦，具体的可参考[微信跨平台组件mars-xlog架构分析及迁移思路](https://zhuanlan.zhihu.com/p/25011775)

## 总结

这篇文章主要讲了日志写入常规方案存在的一些缺陷以及原因，进而引出mmap的定义，优势和使用场景。最后主要讲了mmap的具体实现以及如何应用到日志写入当中。



参考文档:《Android开发高手课》、微信[xlog](https://mp.weixin.qq.com/s/cnhuEodJGIbdodh0IxNeXQ)