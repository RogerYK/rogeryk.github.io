---
layout: post
title: "Golang: slice无法作为map的Key"
---


最近在工作中遇到这样一个需求。就是查询两个数据库中的结果后，在内存中join起来。一般来说Hash Join的效率是最高的。所以自然想到使用Golang
中内置的map。但是在join时关联键并不止一个，所以使用slice来储存，但这时遇到了问题。 如下。  
```go
package main

import "fmt"

func main() {
	m := make(map[interface{}]int)

	key := []interface{}{1, 2}
	m[key] = 1
	fmt.Println(m[key])
}
```
编译可以通过，但是运行代码会报错:
```bash
panic: runtime error: hash of unhashable type []interface {}
```

## 原因  

报错的原因在于slice不能作为map的key。为什么呢？因为map的key的类型必须定义==和!=运算符。而slice
类型是没有定义这两个运算符的，仅可以和nil进行比较。如下。  
[Map spec](https://golang.org/ref/spec#Map_types):
> The comparison operators == and != must be fully defined for operands of the key type; thus the key type must not be a function, map, or slice. If the key type is an interface type, these comparison operators must be defined for the dynamic key values; failure will cause a run-time panic.

而在Golang中，类型的比较运算符是无法由用户来定义的，而是按照一个预定的规则，规定了那些类型定义了这些运算。然而slice被排除在外。   
[Comparison operators spec](https://golang.org/ref/spec#Comparison_operators)
> Slice, map, and function values are not comparable. However, as a special case, a slice, map, or function value may be compared to the predeclared identifier nil. Comparison of pointer, channel, and interface values to nil is also allowed and follows from the general rules above.


## 解决办法
虽然slice无法作为map的key，但是我们注意到，array并未被排除在外。  
[Comparison operators spec](https://golang.org/ref/spec#Comparison_operators)  
> Array values are comparable if values of the array element type are comparable. Two array values are equal if their corresponding elements are equal.

array是可以比较的。所以array可以作为map的key。我们把上面的代码改一下。
```go
package main

import "fmt"

func main() {
	m := make(map[interface{}]int)

	key := [2]interface{}{1, 2}
	m[key] = 1
	fmt.Println(m[key])
}
```
运行通过，没有问题。  
但是，关联键的数量是不确定的，就是说数组的长度在编译时是未知的。我们用一个变量n来代表关联键的数量，代码改为。
```go
package main

import "fmt"

func main() {
	m := make(map[interface{}]int)
	n := 2
	key := [n]interface{}{1, "hello"}
	m[key] = 1
	fmt.Println(m[key])
}
```
编译不通过.
```bash
./main.go:8:9: non-constant array bound n
```
Golang中不允许使用一个变量来限定数组长度, 这怎么办，总不能使用一个长度足够大的数组来处理所有情况吧。这时突然想到了golang
中的反射，其中的```reflect.ArrayOf(count n, ele Type) Type```可以来获取一个长度为n
的数组类型，然后就可以用```reflect.New(typ Type) Value```来在创建数组的值, 这样就可以创建一个长度为n的数组了。 
代码如下:  
```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	m := make(map[interface{}]int)

	n := 2
	var i interface{}
	t := reflect.TypeOf(&i).Elem()
	t = reflect.ArrayOf(n, t)
	rv := reflect.New(t)
	rv = rv.Elem()
	rv.Index(0).Set(reflect.ValueOf(1))
	rv.Index(1).Set(reflect.ValueOf("hello"))
	m[rv.Interface()] = 1
	key := [2]interface{}{1, "hello"}
	fmt.Println(m[key])
}
```
代码运行通过没有问题。这里注意一下获取```interface{}```的类型时，要先获取```*interface{}```的类型，再Elem
获取。因为一个空的```interface{}```值为nil，会被忽略返回nil
。这样做的缺点就是使用了反射，效率会不好。不过也是没有办法的办法，有性能要求的还是自行实现一个更灵活的hashmap。

## 总结
slice和map这种引用类型都是不能比较的。这中设计其实可以理解，因为Golang
中其他的类型的比较都是简单的值比较，指针也只是比较指向的是否同一对象。而slice和map
这种特殊类型无法直接按照这种方式比较，所以就去除了。其实这不是什么大问题，很多语言中的容器，数组都是不能比较的，需要使用时，自己定义一个新的就好了。但是Golang中比较运算符无法重载，这就导致了问题。希望Go2可以解决这些问题吧。

## 参考
1. <https://stackoverflow.com/questions/20297503/slice-as-a-key-in-map>
2. <https://stackoverflow.com/questions/20309751/is-it-possible-to-define-equality-for-named-types-structs?noredirect=1&lq=1> 
3. <https://golang.org/ref/spec#Map_types>
4. <https://golang.org/ref/spec#Comparison_operators >