# 《用Go实现设计模式》三、建造者模式

# 建造者模式
建造者模式，是将一个复杂的对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。创建者模式隐藏了复杂对象的创建过程，它把复杂对象的创建过程加以抽象，通过子类继承或者重载的方式，动态的创建具有复合属性的对象。
## 应用场景
当一个类的构造函数参数个数超过4个，而且这些参数有些是可选的参数，考虑使用构造者模式。

## 代码实现
```go
package builder

import "fmt"

type ResourcePoolConfig struct {
	name string
	maxTotal int
	maxIdle int
	minIdle int
}

type ResourcePoolConfigOption struct {
	maxTotal int
	maxIdle int
	minIdle int
}

type ResourcePoolConfigOptFunc func(option *ResourcePoolConfigOption)

func NewResourcePoolConfig(name string,opts ...ResourcePoolConfigOptFunc) (*ResourcePoolConfig,error){
	if name == ""{
		return nil,fmt.Errorf("name can not be empty")
	}

	option := &ResourcePoolConfigOption{
		maxTotal: 10,
		maxIdle: 9,
		minIdle: 1,
	}

	fmt.Println(option)
	
	for _,opt := range opts{
		opt(option)
	}

	fmt.Println(option)
	if option.maxTotal < 0 || option.minIdle < 0 || option.maxIdle < 0{
		return nil,fmt.Errorf("args err,option : %v",option)
	}
	if option.maxTotal < option.maxIdle || option.minIdle > option.maxIdle{
		return nil,fmt.Errorf("args err,option : %v",option)
	}
	return &ResourcePoolConfig{
		name: name,
		maxTotal: option.maxTotal,
		minIdle: option.minIdle,
		maxIdle: option.maxIdle,
	},nil
}

```
单元测试：
```go
package builder

import (
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"testing"
)

func TestNewResourcePoolConfig(t *testing.T) {
	type args struct {
		name string
		opts []ResourcePoolConfigOptFunc
	}
	tests := []struct{
		name string
		args args
		want *ResourcePoolConfig
		wantErr bool
	}{
		{
			name: "name empty",
			args: args{
				name: "",
			},
			want:    nil,
			wantErr: true,
		},
		{
			name: "success",
			args: args{
				name: "test",
				opts: []ResourcePoolConfigOptFunc{
					func(option *ResourcePoolConfigOption) {
						option.minIdle = 2
					},
				},
			},
			want: &ResourcePoolConfig{
				name:     "test",
				maxTotal: 10,
				maxIdle:  9,
				minIdle:  2,
			},
			wantErr: false,
		},
	}

	for _,tt := range tests{
		t.Run(tt.name, func(t *testing.T) {
			got,err := NewResourcePoolConfig(tt.args.name,tt.args.opts...)
			require.Equalf(t,tt.wantErr,err != nil,"error=%v,wantErr %v",err,tt.wantErr)
			assert.Equal(t,tt.want,got)
		})
	}
}
```
