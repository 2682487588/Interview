### 1.for range

~~~go
//  @Description: 考点
// 1.for range 取得是副本进行遍历，如果进行赋值操作，每次的赋值都会指向一个地址，到最后所有的变量
//都会指向最后一个地址
//2.map打印是无序打印
func main() {
	slices := []int{0, 1, 2, 3}
	m := make(map[int]*int)
	for k, v := range slices {
		m[k] = &v
	}
	//map是无序的
	for k, v := range m {
		fmt.Println(k, "->", *v)
	}
}
~~~

### 2.init

1. 一个函数有多个init，那么init会顺序执行
2. 如果不同的包嵌套init，那么位于嵌套最里层的init会首先执行

**main.go**

~~~go
package main

import (
	"Test/Init/A"
	"fmt"
)

//
//  main
//  @Description: main
//
func main() {
	fmt.Println("main start!")
	A.A()
	fmt.Println("main end!")
}

func init() {
	fmt.Println("main.go init1")
}
func init() {
	fmt.Println("main.go init2")
}
~~~

**A.go**

~~~go
package A

import (
	"Test/Init/B"
	"fmt"
)

func A() {
	fmt.Println("A start")
	B.B()
	fmt.Println("A end")
}

func init() {
	fmt.Println("a.go init1")
}
func init() {
	fmt.Println("a.go init2")
}

~~~

**B.go**

~~~go
package B

import "fmt"

func B() {
	fmt.Println("B start")
}
func init() {
	fmt.Println("b.go init1")
}
func init() {
	fmt.Println("b.go init2")
}
~~~

