# Go并发编程(五)atomic


# atomic
> sync/atomic实现了同步算法底层的原子的内存操作原语。Go提供了一个通用的原子操作API，将更底层的不同的架构下的实现封装成atomic包，提供了修改类型的原子操作（RWW）和加载存储类型的原子操作（Load和Store）的API。

## 应用场景
假设在一个程序中使用一个标志（flag，比如一个bool类型的变量），来标识一个定时任务是否已经启动执行了。如果使用Mutex和RWMutex，在读取和设置这个标志的时候，是可以做到互斥的、保证一个时刻只有一个定时任务在执行，所以使用读写锁和互斥锁是可以的。

但是这个场景没有资源复杂的竞争逻辑，只是并发地读写这个标志，这个场景就可以使用atomic的原子操作了。

你可以使用一个 uint32 类型的变量，如果这个变量的值是 0，就标识没有任务在执行，如果它的值是 1，就标识已经有任务完成了。
## atomic提供的方法
atomic为了支持int32、int64、uint32、uint64、uintptr、Pointer(Add方法不支持)类型，分别提供了Add、CompareAndSwap、Swap、Load、Store等方法。

atomic操作的对象是一个地址，需要把可寻址的变量的地址作为参数传递给方法，而不是把变量的值传递给方法。

## 源码

### 方法一览
atomic的函数签名有很多，但是大部分都是重复的为了不同的数据类型创建了不同的签名，这就是没有泛型的坏处了，基础库会比较麻烦。

1.  第一类`AddXXX`当需要添加的值为负数的时候，做减法，正数做加法。
```go
func AddInt32(addr *int32, delta int32) (new int32)
func AddInt64(addr *int64, delta int64) (new int64)
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddUint64(addr *uint64, delta uint64) (new uint64)
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)
```
2.  第二类`CompareAndSwapXXX`CAS操作，会先比较传入的地址的值是否是old，如果是的话就尝试赋新值，如果不是的话就直接返回false，返回true时表示赋值成功。
```go
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)
```
3.  第三类`LoadXXX`，从某个地址中取值
```go
func LoadInt32(addr *int32) (val int32)
func LoadInt64(addr *int64) (val int64)
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
func LoadUint32(addr *uint32) (val uint32)
func LoadUint64(addr *uint64) (val uint64)
func LoadUintptr(addr *uintptr) (val uintptr)
```
4.  第四类`StoreXXX`，给某个地址赋值
```go
func StoreInt32(addr *int32, val int32)
func StoreInt64(addr *int64, val int64)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
func StoreUint32(addr *uint32, val uint32)
func StoreUint64(addr *uint64, val uint64)
func StoreUintptr(addr *uintptr, val uintptr)
```
5.  第五类`SwapXXX`，交换两个值，并且返回老的值
```go
func SwapInt32(addr *int32, new int32) (old int32)
func SwapInt64(addr *int64, new int64) (old int64)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
func SwapUint32(addr *uint32, new uint32) (old uint32)
func SwapUint64(addr *uint64, new uint64) (old uint64)
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)
```

6.  第六类`Value`用于任意类型的值的Store、Load。签名的方法都只能作用于特定的类型，引入这个方法过后就可以用于任意类型了
```go
type Value
func (v *Value) Load() (x interface{})
func (v *Value) Store(x interface{})
```

## 总结

![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210113173525.jpg)


