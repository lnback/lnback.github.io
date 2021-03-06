# 《用Go实现设计模式》一、单例模式

# 单例模式
## 定义
**一个类只允许创建一个对象（或者叫实例）**，那这个类就是一个单例类，这种设计模式就叫作单例设计模式，简称单例模式。
## 用处
从业务概念上，有些数据在系统中只应该保存一份，就比较适合设计为单例类。
## 单例模式的唯一性
### 进程间唯一
### 如何实现线程间唯一
- 通过获取线程ID
- 在golang中无法使用，因为golang中使用的是协程，并且协程的id并没有暴露出来。

### 如何实现集群环境之间唯一
通过外部共享存储的锁进行，例如文件。
## 如何实现
- 构造函数是private访问权限
- 考虑对象创建时的线程安全问题
- 考虑是否支持延迟加载
- 考虑getInstance的性能问题（是否有加锁等）
## 实现方式
### 懒汉式
- 类加载的时候instance实例已经创建好了
- instance的创建过程是线程安全的
- 不支持延迟加载
- 初始化时间可能会比较长
### 饿汉式
- 在getInstance的时候再去创建
- instance创建过程中需要加锁
- 支持延迟加载
- 不支持高并发
### 双重检测
- 在懒汉式的基础上，将方法的锁改成类级别的锁
- 相对于懒汉式锁的粒度更小，不会每次都去获取锁
### 静态内部类
- 在Java的类中创建一个静态的内部类
- 线程安全
- 延迟加载
### 枚举
- 利用了Java的枚举特性
## 存在的问题
- OOP的特性支持不友好
- 会隐藏类之间的依赖关系
  - 减低可读性
  - 通过构造函数、参数传递等方式声明的类之间的依赖关系，我们通过查看函数的定义，就能很容易识别出来

- 代码扩展性不好
如果有一天需要创建多个实例就会比较麻烦
- 代码的可测性不好
  - 单例类是硬编码的模式，不利于mock
  - 如果是可以修改的全局变量，测试的时候还要注意不同测试用例对它的修改可能会有问题
- 不支持有参数的构造函数

## 代码实现

### 饿汉式
```go
package singleton

type Singleton struct {

}

var singleton *Singleton

func GetInstance()  *Singleton {
	return singleton
}

func init(){
	singleton = &Singleton{}
}
```
单元测试
```go
package singleton_test

import (
	"github.com/stretchr/testify/assert"
	"go-demo/singleton"
	"testing"
)

func TestGetInstance(t *testing.T)  {
	assert.Equal(t,singleton.GetInstance(),singleton.GetInstance())
}

func BenchmarkGetInstanceParallel(b *testing.B)  {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next(){
			if singleton.GetInstance() != singleton.GetInstance(){
				b.Errorf("test fail")
			}
		}
	})
}
```
### 懒汉式（双重检测）
```go
package singleton

import "sync"

var (
	lazySingleton *Singleton
	once = &sync.Once{}

)

func GetLazyInstance() *Singleton {
	if lazySingleton == nil{
		once.Do(func() {
			lazySingleton = &Singleton{}
		})
	}
	return lazySingleton

}
```
单元测试
```go
package singleton_test

import (
	"github.com/stretchr/testify/assert"
	"go-demo/singleton"
	"testing"
)

func TestGetLazyInstance(t *testing.T)  {
	assert.Equal(t,singleton.GetLazyInstance(),singleton.GetLazyInstance())
}

func BenchmarkGetLazyInstanceParallel(b *testing.B){
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			if singleton.GetLazyInstance() != singleton.GetLazyInstance(){
				b.Errorf("test fail")
			}
		}
	})
}
```
### 测试结果
```bash

goos: windows
goarch: amd64
pkg: go-demo/singleton_test
BenchmarkGetInstanceParallel
BenchmarkGetInstanceParallel-6          1000000000               0.242 ns/op           0 B/op          0 allocs/op
BenchmarkGetLazyInstanceParallel
BenchmarkGetLazyInstanceParallel-6      1000000000               0.578 ns/op           0 B/op          0 allocs/op
PASS
ok      go-demo/singleton_test  0.943s
```

通过测试结果可以看出直接init的获取方式性能要稍微好一些

