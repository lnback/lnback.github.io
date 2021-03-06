# Go并发编程(三)WaitGroup

# WaitGroup
> WaitGroup可以解决一个goroutine等待多个goroutine同时结束的场景，这个比较常见的场景就是后端启动了多个消费者干活，爬虫并发爬取数据，多线程下载等。

## 介绍
waitGroup提供了三个方法：
- Add：用来设置WaitGroup的计数值
- Done：用来将WaitGroup的计数值减1，其实就是调用了Add(-1)
- Wait：调用这个方法的goroutine会一直阻塞，直到WaitGroup的计数值变为0
  
## 源码
### 基本结构
```go
type WaitGroup struct {
    //避免复制使用的一个技巧，可以告诉vet违反了复制使用的规则
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}
```
WaitGroup的数据结构包含了一个noCopy和state1
- noCopy作为辅助字段，主要是辅助vet
- state1，一个12字节的具有复合含义的字段。包含了WaitGroup的计数、阻塞在检查点的waiter数和信号量
```go
// state returns pointers to the state and sema fields stored within wg.state1.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    //如果地址是64bit对齐的，数组前两个元素就做state，后一个元素做信号量
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
        //如果地址是32bit对齐，数组后两个元素用来做state，它可以用来做64bit的原子操作，第一个元素32bit用来做信号量
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```
类似于下图：

![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210108145408.svg)

在这里我想到一个问题：因为state的三个值都是32bit的，为什么不用32bit的原子操作呢？

因为counter和waiterCounter合起来是64bit的，如果用32bit的原子操作来修改的话，就不是64bit原子操作了。（好像有点绕）


### Add & Done
```go
func (wg *WaitGroup) Add(delta int) {
    // 先从 state 当中把数据和信号量取出来
	statep, semap := wg.state()

    // 在 waiter 上加上 delta 值
	state := atomic.AddUint64(statep, uint64(delta)<<32)
    // 取出当前的 counter
	v := int32(state >> 32)
    // 取出当前的 waiter，正在等待 goroutine 数量
	w := uint32(state)

    // counter 不能为负数
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}

    // 这里属于防御性编程
    // w != 0 说明现在已经有 waiter 在等待中，说明已经调用了 Wait() 方法
    // 这时候 delta > 0 && v == int32(delta) 说明在调用了 Wait() 方法之后又想加入新的等待者
    // 这种操作是不允许的
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
    // 如果当前没有人在等待就直接返回，并且 counter > 0
	if v > 0 || w == 0 {
		return
	}

    // 这里也是防御 主要避免并发调用 add 和 wait
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// 唤醒所有 waiter
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}

// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

### Wait
Wait方法的实现逻辑是：不断检查state的值，如果其中的计数值变为了0，那么说明所有的任务已经完成，调用者不必等待，直接返回。如果计数值大于0，说明此时还有任务没有完成。那么调用者就变成了等待者，需要假如waiter队列，并且阻塞自己。
```go
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state()
    
    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32) // 当前计数值
        w := uint32(state) // waiter的数量
        if v == 0 {
            // 如果计数值为0, 调用这个方法的goroutine不必再等待，继续执行它后面的逻辑即可
            return
        }
        // 否则把waiter数量加1。期间可能有并发调用Wait的情况，所以最外层使用了一个for循环
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            // 阻塞休眠等待
            runtime_Semacquire(semap)
            // 被唤醒，不再阻塞，返回
            return
        }
    }
}
```

## 总结

- `WaitGroup`可以用一个goroutine等待多个goroutine完成任务，也可以多个goroutine等待一个goroutine任务完成
- `Add(n>0)`方法应该在启动goroutine之前调用，然后再goroutine内部调用`Done`方法
- `WaitGroup`必须遭`Wait`方法返回之后才能再次使用
- `Done`只是`Add`的简单封装

![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210108170502.jpg)
