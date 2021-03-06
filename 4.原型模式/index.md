# 《用Go实现设计模式》四、原型模式


# 原型模式

## 定义
利用已有对象（原型）进行复制（拷贝）的方式来创建新对象，已达到节省创建时间的目的。
## 使用场景
- 对象的创建成本较大，并且同一个类的不同对象之间差别不大（大部分字段相同）
- 对象的创建成本比较大，对象数据需要经过复杂的计算，排序hash等。需要从rpc、网络、数据库等非常慢的io中获取。
## 实现方式
### 深拷贝
- 完全复制
- 实现方式：递归复制、序列化和反序列化
### 浅拷贝
- 仅复制对象的引用，不递归进行复制
- 如果字段是一个地址，那么复制的对象和源对象共享相同的数据
- 如果字段对应的是可变对象，那么复制的对象的改动会导致原来对象的改动
- 如果字段对应的是不可变对象，则没有什么问题
## 代码实现

- 这个模式在Java、C++这种面向对象的语言不太常用，但是如果大家使用过JS的话就会很熟悉，因为JS本身就是基于原型的面向对象语言，所以原型模式在JS应用中非常广泛。
- 有一个需求：假设现在数据库中有大量数据，包含了关键词，关键词被搜索的次数等信息，模块A为了业务需要
  - 会在启动时加载这部分数据到内存中
  - 并且需要定时更新里面的数据
  - 同时展现给用户的数据每次必须要是相同版本的数据，不能一部分来自版本1，一部分来自版本2

```go
package prototype

import (
	"encoding/json"
	"time"
)

type Keyword struct {
	word string
	visit int
	UpdatedAt *time.Time
}
//使用序列化和反序列的方式对对象进行深拷贝
func (k *Keyword) Clone() *Keyword {
	var newKeyword = &Keyword{}

	b , _ := json.Marshal(k)
	json.Unmarshal(b,&newKeyword)

	return newKeyword
}

type Keywords map[string]*Keyword
func (words Keywords) Clone(updateWords []*Keyword) Keywords {
	newKeywords := Keywords{}
	//使用了浅拷贝 后面改动不会修改原数据
	for k,v := range words{
		newKeywords[k] = v
	}
	//使用深拷贝
	for _,word := range updateWords{
		newKeywords[word.word] = word.Clone()
	}

	return newKeywords
}
```
单元测试
```go
package prototype

import (
	"fmt"
	"github.com/stretchr/testify/assert"
	"testing"
	"time"
)

func TestKeywords_Clone(t *testing.T) {
	updateAt, _ := time.Parse("2006", "2020")
	words := Keywords{
		"testA": &Keyword{
			word:      "testA",
			visit:     1,
			UpdatedAt: &updateAt,
		},
		"testB": &Keyword{
			word:      "testB",
			visit:     2,
			UpdatedAt: &updateAt,
		},
		"testC": &Keyword{
			word:      "testC",
			visit:     3,
			UpdatedAt: &updateAt,
		},
	}

	now := time.Now()
	updatedWords := []*Keyword{
		{
			word:      "testB",
			visit:     10,
			UpdatedAt: &now,
		},
	
	}

	got := words.Clone(updatedWords)
	fmt.Println(assert.Equal(t, words["testA"], got["testA"]))
	fmt.Println(assert.NotEqual(t, words["testB"], got["testB"]))
	fmt.Println(assert.NotEqual(t, updatedWords[0], got["testB"]))
	fmt.Println(assert.Equal(t, words["testC"], got["testC"]))
}
```

