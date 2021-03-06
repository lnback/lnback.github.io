# Go并发编程(四)Once


# Once
> Once是一个很简单的并发原语，用来保证仅仅执行一次动作，常用于单例对象的初始化场景

## 介绍
### 案例
用单例对象的创建来介绍一下Once的
在Java中有一种单例模式的创建模式叫双重检测锁，在go中可以用once来实现。
```go
type Singleton struct{}

var (
    singleton *Singleton
    once sync.Once
)

func GetInstance() *Singleton{
    if singleton == nil{
        once.Do(func(){
            singleton = &Singleton{}
        })
    }

    return singleton
}
```
## 源码
```go
func (o *Once) Do(f func()) {
        // 在这里有一段note 讲了一下为什么不直接使用CAS来保证Once只执行一次

	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
    }
}
```
把note中的那段代码截下来
```go
if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	f()
}
```
大家都有一个问题，为什么Once不直接使用CAS获取锁？如果并发调用的话，一个goroutine执行，另外一个goroutine不会等正在这个执行的goroutine成功之后返回，而是直接就返回了，这就不能保证传进来的方法只执行一次了。

所以官方的实现是这样的：
```go
if atomic.LoadUint32(&o.done) == 0 {
    // Outlined slow-path to allow inlining of the fast-path.
    o.doSlow(f)
}
```
会先判断done是否为0，如果不为0说明还没执行过，就进入`doSlow`
```go
func (o *Once) doSlow(f func()){
    o.m.Lock()
    defer o.m.Unlock()

    if o.done == 0{
        defer atomic.StoreUint32(&o.done,1)
        f()
    }
}
```
在`doSlow`中使用了互斥锁来保证只会执行一次
## 总结
- Once 保证了传入的函数只会执行一次，这常用在单例模式，配置文件加载，初始化这些场景下
- 但是需要注意。Once 是不能复用的，只要执行过了，再传入其他的方法也不会再执行了
- 并且 Once.Do 在执行的过程中如果 f 出现 panic，后面也不会再执行了

![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210108181639.jpg)
