---
title: 高效的Okio
categories: Android
top: 0
date: 2020-07-09 16:39:59
tags:
thumbnail:
---


> Okio is a library that complements java.io and java.nio to make it much easier to access, store, and process your data.  

正如[Okio](https://square.github.io/okio/)官网所说，它整合了java io 和nio让它们的更容易使用。此篇深入分析一下Okio高效的原因。

```groovy
implementation "com.squareup.okio:okio:2.7.0"
```

## 目录

- [目录](#目录)
  - [Okio UML](#okio-uml)
  - [基本用法](#基本用法)
  - [Buffer的角色](#buffer的角色)
  - [Segment结构与SegmentPool](#segment结构与segmentpool)
  - [Okio超时机制](#okio超时机制)
  - [结束语](#结束语)

<!--more-->

### Okio UML

![okio](https://dreamweaver.img.we1code.cn/Okio.png)

sink负责写入，source负责读取

### 基本用法

```kotlin
// 读取文件
val readFile = File("read")
val buffer = readFile.source().buffer()
buffer.readString(Charset.forName("utf-8"))

// 写入文件
val writableFile = File("writable")
if (!writableFile.exists()) {
    writableFile.createNewFile()
}
val sink = writableFile.sink().buffer()
sink.writeString("sink write in", Charset.defaultCharset())
sink.flush()
sink.close()
```

无论是读取还是写入，都是对java字节流做了一层包装，对于InputStream包装成了InputStreamSource，对于OutputStream包装成了OutputStreamSink。（对于output/input对于内存而言，output从内存写出，input从外部写入内存）对于Reader和Writer，没有做相关的扩展方法，可能是字符流已经使用了缓冲区吧。然后Source和Sink又被包装RealBufferedSource和RealBufferedSink，这些类的主要函数都以扩展函数的形式放在internal\RealBufferedSource.kt和internal\RealBufferedSink.kt文件中，为啥不放在对应的类中呢？个人觉得可能这些逻辑主要是跟Okio的Buffer相关，抽出来放在一起看起来更直观一些。

### Buffer的角色

字节流的读/写都会申请一遍内存空间用于存放读/写的数据，读写完之后再被回收，这样反复的申请内存显然是对系统性能有影响的。Buffer在Okio中起着重要的角色，它定义了一些读写的方法，并对申请的内存做了复用，节约了IO字节流反复申请内存的开销。使用Buffer之后，java字节流的读写逻辑变为先将数据放在可被回收复用的Segment中，然后等待后续处理。

### Segment结构与SegmentPool

Buffer内部使用*Segment*保存数据，*Segment*用于存储读/写的字节码，

```kotlin
internal class Segment {
  // 用于存储读/写数据 长度8192
  @JvmField val data: ByteArray

  /** The next byte of application data byte to read in this segment.  */
  // 当前byteArray的读取位置
  @JvmField var pos: Int = 0

  /** The first byte of available data ready to be written to.  */
  // 可以理解为data当前的容量
  @JvmField var limit: Int = 0

  /** True if other segments or byte strings use the same byte array.  */
  // 与下面的owner互斥表示这个segment是共享的还是独占的
  // 可能会影响到我们写入数据时是否需要创建新的segment
  @JvmField var shared: Boolean = false

  /** True if this segment owns the byte array and can append to it, extending `limit`.  */
  @JvmField var owner: Boolean = false

  /** Next segment in a linked or circularly-linked list.  */
  @JvmField var next: Segment? = null

  /** Previous segment in a circularly-linked list.  */
  @JvmField var prev: Segment? = null

 // 省略...
}
```

![segment](https://dreamweaver.img.we1code.cn/segment.png)

Segment是环状链表结构的节点。Buffer类中定义了许多读写相关的方法，简单的看下其中commonReadInt和commonWriteInt，看看有什么共通性。

```kotlin
internal inline fun Buffer.commonReadInt(): Int {
  if (size < 4L) throw EOFException()

  // 从Buffer中获取head
  val segment = head!!
  // pos 读取的起始位置
  var pos = segment.pos
  // segment中字节码的长度
  val limit = segment.limit

  // If the int is split across multiple segments, delegate to readByte().
  // 发现剩余的字节码的长度小于4，等于说剩余的长度不足一个int的长度啦，就直接去读byte吧
  if (limit - pos < 4L) {
    // shl 运算优先级要大于and
    return (readByte() and 0xff shl 24
      or (readByte() and 0xff shl 16)
      or (readByte() and 0xff shl 8) // ktlint-disable no-multi-spaces
      or (readByte() and 0xff))
  }

  val data = segment.data
  // 这里呢，在字节码数组中读取4位表示一个Int，第一个字节表示高8位，最后一个字节低8位，将这32位bit求和就得到读取的int值啦
  val i = (data[pos++] and 0xff shl 24
    or (data[pos++] and 0xff shl 16)
    or (data[pos++] and 0xff shl 8)
    or (data[pos++] and 0xff))
  // 读取了4个字节，长度减4咯
  size -= 4L

  // 当当前读取的位置与存储的上限位置相等表示读完啦
  if (pos == limit) {
    // 改变头部的指向
    head = segment.pop()
    // 将读取完的segment回收
    SegmentPool.recycle(segment)
  } else {
    // 改变当前读取到的位置
    segment.pos = pos
  }

  // 返回读取的int值
  return i
}

internal inline fun Buffer.commonWriteInt(i: Int): Buffer {
  // 拿到一个可写入的segment，4表示int的长度，影响后面是否要创建新的segment
  val tail = writableSegment(4)
  val data = tail.data
  // 下面就是常规操作啦，写入一个int值并更新字节数组的长度和size的大小
  var limit = tail.limit
  data[limit++] = (i ushr 24 and 0xff).toByte()
  data[limit++] = (i ushr 16 and 0xff).toByte()
  data[limit++] = (i ushr  8 and 0xff).toByte() // ktlint-disable no-multi-spaces
  data[limit++] = (i         and 0xff).toByte() // ktlint-disable no-multi-spaces
  tail.limit = limit
  size += 4L
  return this
}
```

从上面两个简单函数的实现上，我们可以看出，无论是读还是写，似乎都离不开segment这个结构体，并且Okio还贴心的创建了一个SegmentPool用于回收复用Segment。来看看SegmentPool的结构。

```kotlin
internal actual object SegmentPool {
  actual val MAX_SIZE = 64 * 1024

  private val LOCK = Segment(ByteArray(0), pos = 0, limit = 0, shared = false, owner = false)

  // hash_bucket_count与处理器ALU个数有关，4舍5入保证是2的指数。这样能保证线程之间抢占cpu的可能性更低，我们创建线程池是不是也可以这么设计呢?
  private val HASH_BUCKET_COUNT =
    Integer.highestOneBit(Runtime.getRuntime().availableProcessors() * 2 - 1)
  
  /**
   * Hash buckets each containing a singly-linked list of segments. We use multiple hash buckets so
   * different threads don't race each other. We use thread IDs as hash keys because they're handy,
   * and because it may increase locality.
   *
   * We don't use [ThreadLocal] because we don't know how many threads the host process has and we
   * don't want to leak memory for the duration of a thread's life.
   */
   // 对于io密集型场景，线程数 = cpu核心数 / (1 - 阻塞系数) （阻塞系数为该任务阻塞时间与（阻塞时间+计算时间）的比值）。可以简单设置为2倍cpu核心数
   // 对于计算密集型场景，线程数 = cpu核心数
   // hashBuckets的长度>=可用处理器长度，一般情况是一个线程对应一个Segment，但是也可能存在多个线程对应一个Segment的情况。
  private val hashBuckets: Array<AtomicReference<Segment?>> = Array(HASH_BUCKET_COUNT) {
    AtomicReference<Segment?>()
  }

  @JvmStatic
  actual fun take(): Segment {
    val firstRef = firstRef()

    // firstRef获取值并设置LOCK返回原值
    val first = firstRef.getAndSet(LOCK)
    when {
      // 如果first为LOCK，没有占有锁，不会从池中拿segment
      first === LOCK -> {
        // We didn't acquire the lock. Don't take a pooled segment.
        return Segment()
      }
      // first == null当前占有锁但是segment池是空的。释放锁返回一个segment
      first == null -> {
        // We acquired the lock but the pool was empty. Unlock and return a new segment.
        firstRef.set(null)
        return Segment()
      }
      // 其它情况，获取到了锁并且segment pool不是空的。复用当前的Segment。
      else -> {
        // We acquired the lock and the pool was not empty. Pop the first element and return it.
        firstRef.set(first.next)
        first.next = null
        first.limit = 0
        return first
      }
    }
  }

  @JvmStatic
  actual fun recycle(segment: Segment) {
    require(segment.next == null && segment.prev == null)
    // 如果Segment是共享的不能被回收
    if (segment.shared) return // This segment cannot be recycled.

    val firstRef = firstRef()

    val first = firstRef.get()
    // 若当前锁被占用，不能被回收
    if (first === LOCK) return // A take() is currently in progress.
    val firstLimit = first?.limit ?: 0
    if (firstLimit >= MAX_SIZE) return // Pool is full.

    segment.next = first
    segment.pos = 0
    // 更新缓存池的容量(头插法)
    segment.limit = firstLimit + Segment.SIZE

    // 如果当前值 == 预期值，则以原子方式将该值设置为给定的更新值。
    // 如果成功，则返回 true。返回 false 指示实际值与预期值不相等。
    if (!firstRef.compareAndSet(first, segment)) segment.next = null
    // If we raced another operation: Don't recycle this segment.
  }

  private fun firstRef(): AtomicReference<Segment?> {
    // Get a value in [0..HASH_BUCKET_COUNT).
    val hashBucket = (Thread.currentThread().id and (HASH_BUCKET_COUNT - 1L)).toInt()
    return hashBuckets[hashBucket]
  }
}
```

去掉些注释，代码量不过百来行，但确是整个框架的核心所在。早期版本的SegmentPool简单粗暴的使用synchronized将整个SegmentPool加了锁，访问效率比较低。改良后的SegmentPool是一个无锁的单链表结构，采用哨兵LOCK和CAS机制保证线程的安全性，内部的hashBuckets能保证对于每个线程都有一个独占的Segment单链表，不同的线程之间不会产生竞争。

### Okio超时机制

Okio种Timeout负责管理timeout和deadline，两者的使用场景略有不同，timeout主要用于socket通信超时，deadline用于读写操作是否在规定的期限内执行。接下来看看实际中是如何使用它们的。

```kotlin
fun main() {
    // 初始化一个文件输出流
    val ops = FileOutputStream("writable")
    val sink = ops.sink()
    val timeout = sink.timeout()
    // 设置deadline为5秒
    timeout.deadline(5, TimeUnit.SECONDS)
    // 当前线程睡眠6s
    Thread.sleep(6_000)
    val buffer = sink.buffer()
    // 执行写入操作
    buffer.writeString("after 10 second", Charset.defaultCharset())
    buffer.close()
}
```

执行上述代码后，发现抛出一下异常

```cmd
Exception in thread "main" java.io.InterruptedIOException: deadline reached
  at okio.Timeout.throwIfReached(Timeout.kt:102)
  at okio.OutputStreamSink.write(JvmOkio.kt:50)
  at okio.RealBufferedSink.close(RealBufferedSink.kt:260)
  at MainKt.main(Main.kt:21)
  at MainKt.main(Main.kt)
```

在设置timeout.deadline，会设置timeout中deadlineNanoTime，为当前系统时间加上deadline，调用buffer.close最终会调用到OutputStreamSink的write函数，其中的timeout.throwIfReached()判断是否到达deadline的时间点。

```kotlin
open fun throwIfReached() {
  if (Thread.interrupted()) {
    Thread.currentThread().interrupt() // Retain interrupted status.
    throw InterruptedIOException("interrupted")
  }
  // deadline time 与当前系统时间的差值小于0，抛出异常
  if (hasDeadline && deadlineNanoTime - System.nanoTime() <= 0) {
    throw InterruptedIOException("deadline reached")
  }
}
```

再看看Socket.sink()

```kotlin
// SocketAsyncTimeout是AsyncTimeout子类，实际上是用它控制timeout的
@Throws(IOException::class)
fun Socket.sink(): Sink {
  val timeout = SocketAsyncTimeout(this)
  val sink = OutputStreamSink(getOutputStream(), timeout)
  return timeout.sink(sink)
}

fun AsyncTimeout.source(source: Source): Source {
  return object : Source {
    override fun read(sink: Buffer, byteCount: Long): Long {
      // source.read被withTimeout包装了一层
      return withTimeout { source.read(sink, byteCount) }
    }

    override fun close() {
      withTimeout { source.close() }
    }

    override fun timeout() = this@AsyncTimeout

    override fun toString() = "AsyncTimeout.source($source)"
  }
}

inline fun <T> withTimeout(block: () -> T): T {
  var throwOnTimeout = false
  // 超时逻辑所在
  enter()
  // 省略。。。
}

fun enter() {
  check(!inQueue) { "Unbalanced enter/exit" }
  val timeoutNanos = timeoutNanos()
  val hasDeadline = hasDeadline()
  if (timeoutNanos == 0L && !hasDeadline) {
    return // No timeout and no deadline? Don't bother with the queue.
  }
  inQueue = true
  // 调度timeout，开启watchdog线程，watchdog会计算当前timeout节点是否超时若达到了超时时间将timeout从当前链表中移除并返回执行timeout.timeout()，当没有节点时watchdog自动挂起1分钟。
  scheduleTimeout(this, timeoutNanos, hasDeadline)
}
```

scheduleTimeout处理AysncTimeout单链表，并会通过timeout时间与deadline时间对节点排序，scheduleTimeout启动了Watchdog线程，比对头部节点超时时间与当前系统时间，若发现超时从链表中移除当前节点执行timeout.timeout()关闭socket。

### 结束语

Okio核心思想在于Segment与SegmentPool，相比用传统方法使用io流，减少反复申请内存对系统性能的开销。SegmentPool中对多线程编程的情况做了优化，减少了线程之间的竞争。通读一遍下来，你会发现Okio结构清晰，运用了大量的设计思想，在日常的开发过程中，如果可以合理地学习利用，相信我们也能开发出高性能的应用。
