理解interface核心的两句话


理解interface，只需要理解两句话：
```
1. interface是一种变量类型，interface类型只是interface类型，而不是任意的变量类型

2. interface和其他变量类型在满足某种条件下，可以相互转换
```

第一句话纠正我之前的一个理解误区，认为`interface`可以表示任意类型，实际上不是。

第二句话说明了`interface`怎么使用: 把`interface`转换成其他类型的过程叫做`类型断言`; 把其他类型转换成`interface`类型的过程叫做`interface类型初始化`

我们先看看项目中具体是怎么使用interface

## interface使用场景

场景1: 不确定输入或者返回数据的类型，例如：

```
package main

import "fmt"

// 函数输入参数和返回值都不确定
func appendEverything(v interface{}) (nv interface{}) {
    // 类型断言，断言出v的原始类型
	switch t := v.(type) {
	case []int:
		newSlice := make([]int, 0)
		for _, i := range t {
			newSlice = append(newSlice, i+1)
		}
		nv = newSlice
	case []string:
		newSlice := make([]string, 0)
		for _, i := range t {
			newSlice = append(newSlice, i+"!")
		}
		nv = newSlice
	}

	return
}

func main() {
    // 其他类型作为参数传递给函数，会初始化函数的interface参类型
	fmt.Println(appendEverything([]int{1, 2, 3}))
	fmt.Println(appendEverything([]string{"hello", "world"}))
}

```

场景2：需要把json转为struct，如果json数据是如下:
```
{
    "data": ["hello world", 1234566]
}
```
这种情况，`data`是一个数组，但是数组内的元素类型不是确定的，这种情况定义`struct`结构体如下：
```
type Response struct {
    // 定义结构体，注意元素是interface表示类型不确定
	Data []interface{} `json:"data"`
}

// 假设json数据已经序列化为Response结构体，通过类型断言获取原始数据类型
val := Reponse.Data

// 通过类型断言获取原始数据的值
v0, ok := val[0].(string)
v1, ok := val[0].(float64)
```

场景3: 业务抽象。把一个业务场景的核心逻辑抽象出来，定义成一个特定的interface, 我们实现一个稍微复杂但会经常用到的模式-观察者模式:

```
 // 定义一个被处理的对象
type Collector interface {
    Collect()
}

// 定义注册模型
type Registerer interface {
    // 注册方法
    Register(Collector) error

    // 取消注册
    Unregister(Collector) bool
}

// 定义通知模型
type Notifier interface {
    Notify(Collector) bool
}

// 定义观察者：注册和通知
type Observer interface {
    // interface可以组合
    Registerer
    Notifier
}

// 使用方式
func test(o Observer, c Collector) {
    // 注册
    o.Register(c)
    // 取消注册
    o.Unregister(c)
    // 通知
    o.notify(c)
}
```

如果一个`observer`变量类型，同时实现了`Register`, `Unregister`, `Notify`
这三个方法，那就代表他实现了`interface`， 【那么`observer`变量就可以作为参数传递给`test`函数作为其参数】

> 我这里使用【变量类型实现了某种方法】的说法，而不是我们在面向对象中理解的【类实现了某种方法】，go语言可以通过扩展一个变量类型的方法来模拟面向对象，所以这里我说是【变量类型】。

我们可以试着自定义一个变量类型来实现observer需要的3个方法

```
type Registry struct {
    Collectors  []Collector
}

func (r *Registry) Register(c Collector) error {
    r.Collectors = append(r.Collectors, c)
	return nil
}

func (r *Registry) Unregister(c Collector) bool {
    target := c.Collectors[:0]
    for _, item := range c.Collectors {
        if item != c {
            target = append(target, item)
        }
    }
    c.Collectors = item

    return true
}

// 实现一个notifier结构体
type Notifier struct {
}

func (n *Notifier) notify(c Collector) {
    c.sendMessageToMe()
}

// WeatherObserver【继承】Registry，Notifier，拥有了三个方法，所以是实现了Observer interface
type WeatherObserver struct {
    Registry
    Notifier
}

// 使用：WeatherObserver可以作为参数传递给test函数
test(WeatherObserver)
```

这里再次明确一个结论:
>如果一个变量类型实现了某个interface定义的所有方法，则表示变量类型实现了该interface

重点是：
> 如果一个变量类型实现了某个interface, 那么此变量类型可以和该interface相互转换

推导的论点是：
> 如果`interface`没有定义任何方法，那么任意变量类型都可以和interface类型相互转换

上面的测试函数`appendEverything(v interface{})`的参数类型是`interface{}`，这个interface没有定义任何方法，所以`appendEverything`可以接受任意变量类型的参数

同理, `Observer interface`定义了三个方法， 而`WeatherObserver`实现了``Observer interface`定义的三个方法，所以`WeatherObserver`可以转换为`Observer interface`

## 类型断言

通过【类型断言】，`interface`对象又可以转换为特定的类型对象，上面的例子中`val[0].(string)` 就是把interface转换成`string`类型

类型断言的格式有两种:
```
// 格式1：原始数据, ok := interface类型.(需要装换的类型)
v, ok := v.(string)

// 格式2: switch 原始类型 := interface类型.(type)
switch t := v.(type) {
	case int:
}
```

这样看来，文中3个使用场景，其本质上都是一个场景, interface类型和某种变量类型相互转换

除了interface类型转换以外，我们还需要关注的点是，interface类型可以调用其定义的方法，例如在`test`中调用`o.Register(c)`

那我们有没有考虑2个问题:
```
1. 为什么interface类型可以和其他类型相互装换
2. interface是如何调用其定义的方法，其方法输入参数有哪些
```
搞清楚这2个问题，基本就理解了`interface`原理，嗯，一个一个来。

## interface原理

之前我们说到，interface是一种变量类型，它有两种结构，一种是没有定义任何方法, 其数据结构如下:
```
// 只有两个指针，每个指针8bytes, 一共16bytes
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```
`_type` 指向原始的变量类型，`data`指向原始的数据值。

那`interface`是什么时候初始化的呢，意识到这个问题很关键，答案是【把其他类型的值赋值给一个空的接口类型时候，初始化interface的结构体】

```
package main

import (
	"fmt"
	"reflect"
)

func main() {
	a := "hello world"
	var b interface{}
	t := reflect.TypeOf(b)
	v := reflect.ValueOf(b)
	fmt.Printf("type:%v, value:%v\n", t, v)

	// 赋值之后，初始化了b这个接口
	// 特别注意，b初始化之后，不是变成了string类型，还是interface类型
	b = a
	t = reflect.TypeOf(b)
	v = reflect.ValueOf(b)
	fmt.Printf("type:%v, value:%v", t, v)
}

// 未初始化之前，type nil
type:<nil>, value:<invalid reflect.Value>

// 赋值之后，_type是string, data是hello world
type:string, value:hello world%
```
说到这里，如果一个类型赋值给interface类型，或者一个类型作为参数传递给interface类型，那么就会初始化inteface类型，注意是【初始化】，而不是【类型转换】。

还需要注意的是，【初始化】data指针是把原始的值拷贝一份，如果data超过8个字节(64), 那么这份拷贝值会放到堆上，否则放到栈上，data指针指向拷贝后的地址。

还有一种interface定义了相关方法，其数据结构如下:
```
// 也是16字节
type iface struct {
    tab  *itab
    data unsafe.Pointer
}


type itab struct {
    inter *interfacetype
    _type *_type  // 指向原始类型
    hash  uint32 // copy of _type.hash. 这个参数可以用于快速判断interface类型和某个类型是否一致
    _     [4]byte
    fun   [1]uintptr // 保存所有定义的函数，使用一个数组是因为可以通过地址偏移推断出其他的函数
}
```

`iface.itab.fun` 的方法是在编译期间生成的。go语言在编译期间，会为所有的内置类型和所有的interface类型都生成一份`itab`数据，这里最重要的就是`fun`指针的初始化

第一个问题：【为什么interface类型可以和其他类型相互装换】：

【其他类型可以转换为interface类型】是因为其他类型实现了`iface.itab.fun`里面所有的方法！

【interface类型可以转换为其他类型】是因为interface.hash和目标类型的hash值是一样的


第二个问题: 【什么时候发生类型转换】。
go在运行期间，执行【类型断言】，就会发生类型转换，再次注意【类型转换】和【interface初始化】的区别

第三个问题： 【interface是如何调用其定义的方法】
调用方法类似于 `o.itab.fun[0](o.data)`, 注意`o.data`是原始数据的值的拷贝。


## 方法查找优化
要判断一个类型是否可以和interface相互转换，需要查找彼此方法集，判断是否实现了interface所有的方法，那么通常来说，检索的复杂度为 `O(method_number_of_specify_type * (method_number_of_interface )`

但是 go 采用了一种更好的方法，通过对两个表的函数进行排序，并且对其进行同步遍历，检索的复杂度可以降为: `O(method_number_of_specify_type + method_number_of_interface)`

## interface与nil

interface 只有当`data`和`_type` 或者是`data`和`_itab`都为空，interface才是`nil`
```
package main

type TestStruct struct{}

func NilOrNot(v interface{}) {
    if v == nil {
        println("nil")
    } else {
        println("non-nil")
    }
}

func main() {
    var s *TestStruct
    // s作为参数传递给v, s会初始化v, v的_type指针执行*TestStruct,  所以v不为空
    NilOrNot(s)
}

$ go run main.go
non-nil
```

## ⽅法和方法集

在面向对象中，方法和实例对象绑定，在go中，方法和类型实例绑定，隐式将实例作为第⼀实参 (receiver)。

定义类型方法有如下几点需要注意：

1. 只能为当前包内命名类型定义方法。
2. 参数 receiver 类型可以是 T 或 *T。基类型T不能是接⼝或指针
3. 不支持方法重载
4. 可用实例的value和pointer调用方法，编译器会自动转换

```
package main

import "fmt"

type Name string

func (n *Name) l() {
	fmt.Println(len(*n))
}

func main() {
	var a Name
	a = "MyName"
	a.l()
}
```

## 方法集

每个类型都有与之关联的⽅方法集，这会影响到接⼝口实现规则。

类型T方法集包含全部receiver T方法。
类型*T方法集包含全部 receiver T + *T 方法。
如类型 S 包含匿名字段 T，则 S ⽅方法集包含 T 方法。
如类型 S 包含匿名字段 *T，则 S ⽅方法集包含 T + *T 方法。
不管嵌⼊入 T 或 *T，*S ⽅方法集总是包含 T + *T 方法。

为什么*T的方法集中，可以包括T方法，而T方法集中不能包括*T呢？

因为实例在调用方法时，是拷贝实例的值，当receiver是指针时，拷贝的指针
通过&依然可以获取正确的实例值，当receiver是值时，拷贝的值的地址不是实例的真实地址

因此，类型T方法集只能包括receiver T方法



## 总结
1. interface是一种变量类型，注意interface的【初始化】时机，赋值和作为函数参数传递，都会初始化interface
2. interface可以和其他类型相互转换【类型断言】
3. interface可以调用其定义的方法 【拷贝参数】
