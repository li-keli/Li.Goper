---
title: "深入Go-Channel"
date: 2020-05-12T16:17:54+08:00
draft: false
featured_image: https://oss.likeli.top/uPic/20200513112628!smail_img_likeli
tags: ["Go", "Channel"]
categories: Channel
---

# Channel

Channel是Go核心的数据结构和Goroutine之间的通信方式。通过Channel的设计原理、数据结构和常见操作来理解Channel。

## 设计原理

Go最常见的、也经常被人提及的设计模式就是：**不要通过共享内存的方式进行通信，而是应该通过通信的方式共享内存。**

很多主流的语言中，多个线程传递数据的方式一般都是*共享内存*，为了解决线程冲突的问题，我们需要限制同一时间能够读写这些变量的线程数量（锁），这与Go鼓励的方式并不相同。

![shared-memory](https://oss.likeli.top/uPic/20200512162833!smail_img_likeli)

虽然我们在Go中也能使用共享内存加互斥锁进行通信，但是Go提供了一种不同的并发模型，也就是通信顺序进程（Communication sequential processes，CSP）。Goroutine和Channel分别对应CSP中的实体和传递信息的媒介，Go的Goroutine通过Channel传递数据。

![shared-memory](https://oss.likeli.top/uPic/20200512162841!smail_img_likeli)

上图中的两个Goroutine，一个回想Channel中发送数据，另一个会从Channel中接收数据，它们两者能够独立运行并不存在直接关联，但是能通过Channel间接完成通信。

### **先入先出**

目前的Channel收发操作均遵循了先入先出（FIFO）的设计，具体规则如下：

* 先从Channel读取数据的Goroutine会先接收到数据
* 先从Channel发送数据的Goroutine会得到先发送数据的权利

### 无锁管道

锁是一种常见的并发控制技术，我们一般会将锁分成**乐观锁**和**悲观锁**，即乐观并发控制和悲观并发控制，无锁（lock-free）队列更准确的描述是使用乐观并发控制的队列。乐观并发控制也叫乐观锁，但是它并不是真正的锁，很多人都会误以为乐观锁是一种真正的锁，然而它只是一种并发控制的思想。

乐观并发控制本质上是基于验证的协议，通过使用**原子指令CAS**（compare-and-swap或者compare-and-set）在多线程中同步数据，无锁队列的实现也依赖这一原子指令。

Channel在运行时的内部表示是`runtime.hchan`，该结构体中包含了一个用户保护成员变量的互斥锁，从某种程度上说，**Channel是一个用于同步和通信的有锁队列**。使用互斥锁解决程序中可能存在的线程竞争问题是很常见的，我们能很容易的实现有锁队列。

然后锁导致的休眠和唤醒会带来额外的上下文切换，如果临界区[^5]过小，加锁解锁导致的额外开销就会成为性能瓶颈。

Go语言社区在2014年提出了无锁Channel的实现方案，该方案将Channel分成了下面三种类型：

* 同步Channel - 不要缓冲区，发送方会直接将数据交给（Handoff）接收方
* 异步Channel - 基于环形缓存的传统生产着消费者模型
* `chan struct{}`类型的异步Channel - struct{}类型不占用内存空间，不需要实现缓冲区和直接发送的语义

> 这个提案的目的也不是实现完全无锁的队列，只是在一些关键路径上通过无锁提升Channel的性能。社区中已经有[无锁Channel的实现](https://github.com/OneOfOne/lfchan/blob/master/lfchan.go)，但是在实际的基准测试中，无锁队列在多核测试中的表现还需要进一步改进。
>
> 因为目前通过CAS实现的无锁Channel没有提供FIFO的特性，所以改提案暂时也被搁浅了。



## 数据结构

Go的Channel在运行时使用`runtime.hchan`[^1]结构体表示。如下：

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

`runtime.hchan`结构体中的五个字段`qcount`、`dataqsiz`、`buf`、`sendx`、`recv`构建底层的循环队列：

* qcount - Channel中的元素个数
* dataqsiz - Channel中的循环队列的长度
* buf - Channel的缓冲区数据指针
* sendx - Channel的发送操作处理到的位置
* recvx - Channel的接收操作处理的位置

除此之外，`elemsize`、`elemtype`分别表示当前Channel能够收发的元素类型和大小；`sendq`和`recvq`存储了当前Channel由于缓冲区空间不足而阻塞的Goroutine列表，这些等待队列使用双向链表`runtime.waiq`[^2]表示，链表中所有的元素都是`runtime.sudog`[^3]结构：

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

`runtime.sudog`表示一个在等候列表中的Goroutine，该结构体中存储了阻塞的相关信息以及两个分别指向前后的`runtime.sudog`的指针。



## 创建管道

Go语言中所有Channel的创建都会使用`make`关键字。

> 编译器会将`make(chan int, 10)`表达式被转换成`OMAKE`类型的节点，并在类型检查阶段将`OMAKE`类型的节点转换成`OMAKECHAN`类型：

```go
func typecheck1(n *Node, top int) (res *Node) {
	switch n.Op {
	case OMAKE:
		...
		switch t.Etype {
		case TCHAN:
			l = nil
			if i < len(args) { // 带缓冲区的异步 Channel
				...
				n.Left = l
			} else { // 不带缓冲区的同步 Channel
				n.Left = nodintconst(0)
			}
			n.Op = OMAKECHAN
		}
	}
}
```

这一阶段会对传入`make`关键字的缓冲区大小进行检查，如果我们不向`make`传递表示缓冲区大小的参数，那么就会设置一个默认值0，也就是当前Channel不存在缓冲区。

`OMAKECHAN`类型的节点最终都会在SSA中间代码生成阶段之前被转换成调用`runtime.makechan`[^6]或者`runtime.makechan64`的函数

```go
case OMAKECHAN:
		// When size fits into int, use makechan instead of
		// makechan64, which is faster and shorter on 32 bit platforms.
		size := n.Left
		fnname := "makechan64"
		argtype := types.Types[TINT64]

		// Type checking guarantees that TIDEAL size is positive and fits in an int.
		// The case of size overflow when converting TUINT or TUINTPTR to TINT
		// will be handled by the negative range checks in makechan during runtime.
		if size.Type.IsKind(TIDEAL) || maxintval[size.Type.Etype].Cmp(maxintval[TUINT]) <= 0 {
			fnname = "makechan"
			argtype = types.Types[TINT]
		}

		n = mkcall1(chanfn(fnname, 1, n.Type), n.Type, init, typename(n.Type), conv(size, argtype))
```

`runtime.makechan`和`runtime.makechan64`会根据传入的参数类型和缓冲区大小创建一个新的Channel结构，其中后者用于处理缓冲区大小大于$ 2^{32} $的情况，所以重点关注`runtime.makechan`函数：

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```

上述代码根据Channel中收发元素的类型和缓冲区带下初始化`runtime.hchan`结构体和缓冲区：

* 如果当前Channel中不存在缓冲区，那么就只会为`runtime.hchan`分配一段内存空间
* 如果当前Channel中存储的类型不是指针类型，就会为当前的Channel和底层的数组分配一块连续的内存空间
* 在默认情况下会单独为`runtime.hchan`和缓冲区分配内存

在函数的最后会统一更新`runtime.hchan`的`elemsize`、`elemtype`、`dataqsiz`几个字段。



## 发送数据

当给Channel发送数据时，就需要使用`ch <- i`语句。最终调用`runtime.chansend`[^4]。

`runtime.chansend`[^7]这个函数负责了发送数据的全部逻辑，如果在调用时将`block`参数设置为`true`。那么久表示当前发送的操作时一个阻塞操作：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
		// ...
  
  	lock(&c.lock)
  
  	if c.closed != 0 {
      unlock(&c.lock)
      panic(plainError("send on closed channel"))
    }
  	// ...
```

在发送数据的逻辑执行之前会为当前Channel加锁，防止发生竞争条件。如果Channel已经关闭。那么向该Channel发送数据时就会报`send on closed channel`错误并终止程序。

因为`runtime.chansend`函数的实现比较复杂，所以我们这里将该函数的执行过程分为三个部分：

* 当存在等待的接受者时，通过`runtime.send`直接将数据发送给阻塞的接受者
* 当缓冲区存在空余空间时，将发送的数据写入Channel的缓冲区
* 当不存在缓冲区或者缓冲区已满时，等待其他Goroutine从Channel接受数据

### 直接发送

如果目标Channel没有被关闭并且已经有出于等待的Goroutine，那么`runtime.chansend`函数会从接受队列`recvq`中取出最先陷入等待的Goroutine并直接向它发送数据：

```go
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```

![channel-direct-send](https://oss.likeli.top/uPic/20200513095441!smail_img_likeli)

发送数据时会调用`runtime.send`，该函数的执行可以分为两部分[^8]：

* 调用`runtime.sendDirect`函数将发送的数据直接拷贝到`x = <- c`表达式中变量`x`所在的内存地址上
* 调用`runtime.goready`将等待接收数据的Goroutine标记成可运行的状态`Grunnable`并把该Goroutine放到发送方所在的处理器的`runnext`上等待执行，该处理器在下一次调用时就会立刻唤醒数据的接收方

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}
```

> 需要注意的是，发送数据的过程只是将接收方的Goroutine放到了处理器的`runnext`中，并不是立刻执行该Goroutine。



### 缓冲区

如果创建的Channel包含缓冲区并且Channel中的数据没有装满，就会执行下面的代码[^9]：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
	...
}
```

在这里首先会使用`chanbuf`计算出下一个可以存储数据的位置，然后通过`runtime.typedmemmove`将发送的数据拷贝到缓冲区并增加`sendx`索引和`qcount`计数器。

![channel-buffer-send](https://oss.likeli.top/uPic/20200513100305!smail_img_likeli)

如果当前Channel的缓冲区未满，想Channel发送的数据会储存在Channel中`sendx`索引所在的位置，并将`sendx`索引加一，由于这里的`buf`是一个循环数据，所以当`sendx`等于`dataqsiz`时就会重新回到数组开始的位置。

### 阻塞发送

当Channel没有接受者能够处理数据时，想Channel发送数据就会被下游阻塞，当然是用`select`关键字可以向Channel非阻塞的发送消息。向Channel阻塞的发送数据会执行下面的代码，简化一下逻辑后如下：

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	...
	if !block {
		unlock(&c.lock)
		return false
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.elem = ep
	mysg.g = gp
	mysg.c = c
	gp.waiting = mysg
	c.sendq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)

	gp.waiting = nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

过程：

1. 调用`runtime.getg`获取发送数据使用的Goroutine；
2. 执行`runtime.acquireSudog`函数获取`runtime.sudog`结构体并设置这一次阻塞发送的相关信息，例如发送的Channel、是否在Select控制结构中和待发送数据的内存地址等；
3. 将刚刚创建并初始化的`runtime.sudog`加入待发送队列，并设置到当前Goroutine的`waiting`上，表示Goroutine正在等待该`sudog`准备就绪；
4. 调用`runtime.goparkunlock`函数将当前的Goroutine陷入睡眠等待唤醒；
5. 被调度器唤醒后会执行一些收尾工作，将一些属性置零并且释放`runtime.sudog`结构体；

在最后，函数会返回`true`表示这项Channel发送数据的过程结束。

### 小结

使用`ch <- 1`表达式想Channel发送数据时遇到的几种情况：

1. 如果当前Channel的`recvq`上存在已经被堵塞的Goroutine，那么会直接将数据发送给当前Goroutine并将其设置成下一个运行的Goroutine；
2. 如果Channel存在缓冲区并且其中还有空间的容量，我们就会直接将数据存储到当前缓冲区`sendx`所在的位置上；
3. 如果不满足上面的两种情况，就会创建一个`runtime.sudog`结构并将其加入Channel的`sendq`队列中，当前Goroutine也会陷入阻塞等待其他的协程从Channel接收数据；



发送数据的过程包含几个会触发Goroutine调度的时机：

1. 发送数据时发现Channel上存在等待接收数据的Goroutine，立即设置处理的`runnext`属性，但是并不会立刻触发调度；
2. 发送数据时没有找到接收方并且缓冲区已经满了，这是就会将自己加入Channel的`sendq`队列并调度`runtime.goparkunlock`触发Goroutine的调度让出处理器的使用权；



## 接收数据

Go语言有两种不同的方式去接收Channel中的数据：

* `i <- ch`
* `i, ok <- ch`

这两种不同的方法经过编译器的处理都会变成`ORECV`类型的节点，后者会在类型检查阶段被转换成`OAS2RECV`类型。数据的接收操作遵循以下的线路图：

![channel-receive-node](https://oss.likeli.top/uPic/20200513102431!smail_img_likeli)

虽然两种不同的接收方式会被转换成`runtime.chanrecv1`和`runtime.chanrecv2`两种不同函数的调用，但是最终都还是会调用`runtime.chanrecv`[^10]

当从一个空Channel接收数据时会直接调用`runtime.gopark`直接让出处理器的使用权。

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
  // ...
```

如果当前Channel已经被关闭并且缓冲区不存在任何的数据，那么就会清除`ep`指针中的数据并立刻返回。

除了上述两种特殊情况，使用`runtime.chanrecv`从Channel接收数据时还包含了三种不同情况：

* 当存在等待发送者时，通过`runtime.recv`直接从阻塞的发送者或者缓冲区中获取数据；
* 当缓冲区存在数据时，从Channel的缓冲区中接收数据；
* 当缓冲区中不存在数据时，等待其他Goroutine想Channel发送数据；

### 直接接收

当Channel的`sendq`队列中包含处于等待状态的Goroutine时，该函数会取出队列头等待的Goroutine，处理的逻辑和发送时相差无几，只是发送数据时调用的是`runtime.send`函数，而接收数据使用`runtime.recv`[^11]函数：

```go
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
```

`runtime.recv`函数的实现比较复杂：

```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if ep != nil {
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	gp := sg.g
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}
```

该函数会根据缓冲区的大小分别处理不同的情况：

* 如果Channel不存在缓冲区：
  * 调用`runtime.recvDirect`函数会将Channel发送队列中Goroutine存储的`elem`数据拷贝到目标内存地址中；
* 如果Channel存在缓冲区：
  * 将队列中的数据拷贝到接收方的内存地址；
  * 将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；

无论发生哪种情况，运行时都会调用`runtime.goready`函数将当前处理器的`runnext`设置成发送数据的Goroutine，在调度器下次调度时将阻塞的发送方唤醒。

![channel-receive-from-sendq](https://oss.likeli.top/uPic/20200513105845!smail_img_likeli)

收尾工作包括递增`recvx`，一单发现索引超过了Channel的容量时，就会将它归零（循环队列）；除此之外，这个函数还会减少`gcount`计数器并释放持有的Channel的锁。

### 阻塞接收

当Channel的发送队列中不存在等待的Goroutine并且缓冲区中也不存在任何数据时，从管道中接收数据的操作会变成阻塞操作，然后不是所有的接收操作都是阻塞的，与`select`关键字结合使用时就可能会使用到非阻塞的接收操作：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	...
	if !block {
		unlock(&c.lock)
		return false, false
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.elem = ep
	gp.waiting = mysg
	mysg.g = gp
	mysg.c = c
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	gp.waiting = nil
	closed := gp.param == nil
	gp.param = nil
	releaseSudog(mysg)
	return true, !closed
}
```

在正常的接收场景中，我们会使用`runtime.sudog`结构体将当前Goroutine包装成一个出于等待状态的Goroutine并将其加入到接收队列中。

完成入队之后，上述代码还会调用`runtime.goparkunlock`函数like出发Goroutine的调度，让出处理器的使用权并等待调度器的调度。

### 小结

从Channel接收数据时可能发生的五种情况：

1. 如果Channel为空，那么就会直接调用`runtime.gopark`挂起当前Goroutine；
2. 如果Channel已经关闭并且缓冲区没有任何数据，`runtime.chanrecv`函数会直接返回；
3. 如果Channel的`sendq`队列中存在挂起的Goroutine，就会将`recvx`索引所在的数据拷贝到接收变量所在的内存空间上，并将`sendq`队列中Goroutine的数据拷贝到缓冲区；
4. 如果Channel的缓冲区中包含数据就会直接读取`recvx`索引对应的数据；
5. 在默认情况下回挂起当前的Goroutine，并将`runtime.sudog`结构加入`recvq`队列并陷入休眠等待调度器的唤醒；

从Channel接收数据时，会触发的Goroutine调度的两个时机：

1. 当Channel为空时；
2. 当缓冲区中不存在数据并且也不存在数据的发送者时；



## 关闭管道

编译器会将用于关闭管道的关键字`close`转换成`OCLOSE`节点以及`runtime.closechan`的函数调用。

当Channel时一个空指针或者已经被关闭时，Go语言运行时都会直接`panic`并抛出异常：

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}
  // ...
```

处理完了这些异常的情况之后就可以开始执行关闭Channel的逻辑了，下面这段代码的主要工作就是将`recvq`和`sendq`连个队列中的数据加入到Goroutine列表`gList`中，与此同时该函数会清除所有`sudog`上未处理的元素：

```go
	c.closed = 1

	var glist gList
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		gp := sg.g
		gp.param = nil
		glist.push(gp)
	}

	for {
		sg := c.sendq.dequeue()
		...
	}
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

函数在最后会为所有被阻塞的Goroutine调用`runtime.goready`触发调度。





## 引用

本文来自：https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#64-channel



[^1]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L32-L51
[^2]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L53-L561 
[^3]: https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342-L368
[^4]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L142
[^5]: https://en.wikipedia.org/wiki/Critical_section
[^6]: https://github.com/golang/go/blob/6ed4661807b219781d1aa452b7f210e21ad1974b/src/cmd/compile/internal/gc/walk.go#L1185-L1200
[^7]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L142
[^8]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L290-L300
[^9]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L142
[^10]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L422
[^11]: https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L556