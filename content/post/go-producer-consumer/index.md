---
title: "Go 生产者消费者模型：有界队列与优雅退出"
description: "通过 channel 和 select 构建支持退出信号的有界阻塞队列，并联系 Kubernetes Controller 的事件消费流程。"
slug: go-producer-consumer
date: 2026-10-19 16:36:32+0800
image: cover.png
categories:
    - Go
    - Kubernetes
---

### 研究背景
对于常见的一个Bounded Blocking Queue:
```go
// 定义队列结构
type BoundedBlockingQueue struct {
    ch chan int
}

// 构造函数
func Constructor(capacity int) BoundedBlockingQueue {
    return BoundedBlockingQueue{
        ch : make(chan int, capacity),
    }
}

// 入队方法（如果是别人调用，且队列满了，底层的 channel 会自动让调用方的 goroutine 阻塞）
func (q *BoundedBlockingQueue) Enqueue(element int) {
    q.ch <- element
}

// 出队方法
func (q *BoundedBlockingQueue) Dequeue() int {
    return <- q.ch 
}

func (q *BoundedBlockingQueue) Size() int {
    return len(q.ch)
}
```
我们假设这样一个场景：现代云原生架构中，我们经常需要处理节点的平滑重启（Graceful Shutdown）。如果系统要关闭，我们不能让试图入队的线程永远卡死在那里。那么在Enqueue阻塞的生产者怎么给干掉啊，而不是我想要把这个生产消费工作停了，但是channel旁边依然伺服着一群go routine在ch有一有空闲的时候随时再enqueue继续生产

### go并发编程基础
#### 多路复用select机制：
select专门用来同时监听多个通道。谁先准备好，就执行谁；如果多个同时准备好，随机挑一个。
```go
func (q *BoundedBlockingQueue) Enqueue(element int) {
    select {
    case <- q.done:
        // 一旦外部调用了 close(q.done)，这行代码就会立刻被触发
        // 直接 return 退出，不再傻等
        return
    case q.ch <- element:
        // 如果 q.done 没被关闭，且队列有空位，就会正常入队
    }
}
```
说白了，其实select下阻塞的go routine可以有两个出口：眼里同时盯着两辆船(“阻塞”)，一看哪一辆有空位就直接上去了（“唤醒”）

### iterate: BoundedBlockingQueue 带有优雅退出机制的版本
```go 
type BoundedBlockingQueue struct {
	ch   chan int           // 数据平面
	done chan struct{}      // 控制平面
}

func Constructor(capacity int) BoundedBlockingQueue {
	return BoundedBlockingQueue{
		ch:   make(chan int, capacity),
		done: make(chan struct{}), 
	}
}

func (q *BoundedBlockingQueue) Enqueue(element int) bool {
	select {
	case <-q.done: //一旦外部 close(q.done)，就会触发这个，直接拒绝新的入队请求，防止变僵尸协程
		return false
	case q.ch <- element:
		return true
	}
}

func (q *BoundedBlockingQueue) Dequeue() (int, bool) { //同理dequeue这边也要做优雅退出，不能直接写阻塞等待queue中有数据，不然也成僵尸go routine了
	select {
	case <-q.done: //收到shutdown信号后执行清空当前队列
		select {
		case val := <-q.ch:
			return val, true 
		default:
			return 0, false  
		}
	case val := <-q.ch:
		return val, true
	}
}

func (q *BoundedBlockingQueue) Size() int {
	return len(q.ch)
}

func (q *BoundedBlockingQueue) Shutdown() {
	// 所有监听 <-q.done 的 select 都会立刻收到信号
	close(q.done)
}
```

### 应用
Kubernetes的代码中，几乎所有的Controller(Deployment Controller Node Controller 还有你自己写的kubernetes operator)核心都是一个生产者消费者模型：
如图所示，reflector通过list&watch(长连接看metadata.ResourceVersion变化时APIserver的事件)机制，先打包进一个DeltaFIFO队列中，Informer中的一个常驻go routine则从中不断pop出对象，Indexer看到弹出对象时则直接存在Thread Safe Store中（这个存储存在的意义是减轻查APIserver的压力，后续的controller worker直接在读这个**内存中**的Indexer即可不需要问APIserver）

在informer更新完Indexer本地缓存之后，同时会把这个事件分发给用户注册的`ResourceEventHandler`，这里为维护一个WorkQueue，里面存的是Key(注意仅仅是Key字符串，不是整个对象)，此时Controller启动时起的多个goroutine执行Worker()，就是从WorkQueue取出Key，拿着这个Key去我们刚才保存的Thread Safe Store中去获得完整的对象状态
![Screenshot 2026-05-10 at 09.20.36.png](https://cdn.jsdelivr.net/gh/XiDianJiale/picgo-repo-img/img/Screenshot%202026-05-10%20at%2009.20.36.png)

在上下游处理速率不匹配时，它能起到削峰填谷的作用。更重要的是它的有界性，在突发洪峰流量或下游服务宕机时，无界队列会导致自身内存雪崩OOM

### 思考题
在数字电路设计中，假设现在有一个100MHz 的数据采集模块，要把数据传给 50MHz 的处理模块，这里同样是一个“快慢两端速率不匹配”的问题，硬件上如何实现一个生产者消费者模型？