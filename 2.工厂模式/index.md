# 《用Go实现设计模式》二、工厂模式

# 工厂模式

## 如何判断使用工厂模式
### 封装变化
创建逻辑有可能变化，封装成工厂类之后，创建逻辑的变更对调用者透明。
### 代码复用
创建代码抽离到独立的工厂类之后可以复用
### 隔离复杂性
封装复杂的创建逻辑，调用者无需了解如何创建对象
### 控制复杂度
将创建代码抽离出来，让原本的函数或类职责更单一，代码更简洁

## 简单工厂
因为Go本身没有构造函数，所以一般使用`NewName`的方式创建接口/对象，当他返回的是接口的时候其实就是简单工厂模式
```go
package factory

type IRuleConfigInterface interface {
	Parse(data []byte)
}

type JsonConfigInterface struct {
	
}

func (config JsonConfigInterface) Parse(data []byte)  {
	panic("implement me")
}

type YamlConfigInterface struct {
}

func (yaml YamlConfigInterface) Parse(data []byte)  {
	panic("implement me")
}

func NewIRuleConfigParse(t string) IRuleConfigInterface  {
	switch t {
	case "json":
		return JsonConfigInterface{}
	case "yaml":
		return YamlConfigInterface{}
	}
	return nil
}

```
单元测试：通过得到的got和want对比，测试是否equal。
```go
package factory
import (
	"reflect"
	"testing"
)

func TestNewIRuleConfigParser(t *testing.T) {
	type args struct {
		t string
	}
	tests := []struct {
		name string
		args args
		want IRuleConfigInterface
	}{
		{
			name: "json",
			args: args{t: "json"},
			want: JsonConfigInterface{},
		},
		{
			name: "yaml",
			args: args{t: "yaml"},
			want: YamlConfigInterface{},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := NewIRuleConfigParse(tt.args.t); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("NewIRuleConfigParser() = %v, want %v", got, tt.want)
			}
		})
	}
}
```
## 工厂方法
当对象的创建逻辑比较复杂，不只是简单的 new 一下就可以，而是要组合其他类对象，做各种初始化操作的时候，推荐使用工厂方法模式，将复杂的创建逻辑拆分到多个工厂类中，让每个工厂类都不至于过于复杂
```go
type IRuleConfigParserFactory interface {
	CreateParser() IRuleConfigParserFactory
}

type yamlRuleConfigParserFactory struct {
}

func(yaml yamlRuleConfigParserFactory) CreateParser() IRuleConfigParserFactory {
	return yamlRuleConfigParserFactory{}
}

type jsonRuleConfigParserFactory struct {

}

func (json jsonRuleConfigParserFactory) CreateParser() IRuleConfigParserFactory {
	return jsonRuleConfigParserFactory{}
}

func NewIRuleConfigParserFactory(t string) IRuleConfigParserFactory {
	switch t {
	case "json":
		return jsonRuleConfigParserFactory{}
	case "yaml":
		return yamlRuleConfigParserFactory{}
	}
	return nil
}
```
单元测试
```go
func TestNewIRuleConfigParserFactory(t *testing.T) {
	type args struct {
		t string
	}
	tests := []struct {
		name string
		args args
		want IRuleConfigParserFactory
	}{
		{
			name: "json",
			args: args{t: "json"},
			want: jsonRuleConfigParserFactory{},
		},
		{
			name: "yaml",
			args: args{t: "yaml"},
			want: yamlRuleConfigParserFactory{},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := NewIRuleConfigParserFactory(tt.args.t); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("NewIRuleConfigParserFactory() = %v, want %v", got, tt.want)
			}
		})
	}
}
```
## 抽象工厂
```go
type IRuleConfigParser interface {
	Parse(data []byte)
}

// jsonRuleConfigParser jsonRuleConfigParser
type jsonRuleConfigParser struct{}

// Parse Parse
func (j jsonRuleConfigParser) Parse(data []byte) {
	panic("implement me")
}

// ISystemConfigParser ISystemConfigParser
type ISystemConfigParser interface {
	ParseSystem(data []byte)
}

// jsonSystemConfigParser jsonSystemConfigParser
type jsonSystemConfigParser struct{}

// Parse Parse
func (j jsonSystemConfigParser) ParseSystem(data []byte) {
	panic("implement me")
}

// IConfigParserFactory 工厂方法接口
type IConfigParserFactory interface {
	CreateRuleParser() IRuleConfigParser
	CreateSystemParser() ISystemConfigParser
}

type jsonConfigParserFactory struct{}

func (j jsonConfigParserFactory) CreateRuleParser() IRuleConfigParser {
	return jsonRuleConfigParser{}
}

func (j jsonConfigParserFactory) CreateSystemParser() ISystemConfigParser {
	return jsonSystemConfigParser{}
}
```
单元测试
```go
package factory

import (
	"reflect"
	"testing"
)

func Test_jsonConfigParserFactory_CreateRuleParser(t *testing.T) {
	tests := []struct {
		name string
		want IRuleConfigParser
	}{
		{
			name: "json",
			want: jsonRuleConfigParser{},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			j := jsonConfigParserFactory{}
			if got := j.CreateRuleParser(); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("CreateRuleParser() = %v, want %v", got, tt.want)
			}
		})
	}
}

func Test_jsonConfigParserFactory_CreateSystemParser(t *testing.T) {
	tests := []struct {
		name string
		want ISystemConfigParser
	}{
		{
			name: "json",
			want: jsonSystemConfigParser{},
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			j := jsonConfigParserFactory{}
			if got := j.CreateSystemParser(); !reflect.DeepEqual(got, tt.want) {
				t.Errorf("CreateSystemParser() = %v, want %v", got, tt.want)
			}
		})
	}
}
```
## DI容器
DI容器就相当于Spring的Bean容器，首先通过配置将Bean注册到容器中，以后取的时候就从容器中拿。

## 工厂方法和抽象工厂的区别
- 抽象工厂的关键在于产品之间的抽象关系，所以至少需要两种产品。工厂方法在于生产产品，不关注产品之间的关系，所以可以只生产一个产品。
- 抽象工厂中客户端把产品的抽象关系理清楚，在最终使用的时候，一般使用客户端（和其接口），产品之间的关系是被封装固定的；而工厂方法是在最终使用的时候，使用产品本身（和其接口）。

也可以简单归纳：
- 实例 -> 类 -> 类工厂
- 实例 -> 类 -> 类工厂 -> 抽象工厂


