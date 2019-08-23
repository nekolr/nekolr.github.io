---
title: Golang 快速入门
date: 2019/4/15 15:12:0
tags: [Golang]
categories: [Golang]
---
本着快速入门的原则，只记录必要的知识点，其他细节可以在具体使用时进行补充。

<!--more-->
# 类型
Golang 的基本类型有三大类：布尔类型、数字类型和字符串类型。

# 变量

## 变量声明和赋值
使用 var name type 的格式声明变量，如 `var age int`。  

如果变量声明后没有赋值，则会自动初始化为零值（Zero Value），比如 int 类型的为 0，bool 类型的为 false。如果要在声明的同时初始化，则加上值，比如 `var age int = 22`。  

## 类型推断
如果变量有初始值，那么可以省去声明类型，如 `var age = 22`。  

## 声明多个变量
```go
// 方式一
var width, height int = 20, 30
// 方式二：因式分解关键字
var (
    name = "saber"
    age = 22
    height int
)
```

## 简短声明
```go
height := 165
name, age := "saber", 22
```

## 类型转换
Golang 中没有自动类型提升或类型转换，把一个变量赋值给另一个不同类型的变量需要显式地进行类型转换。  

```go
i := 44
j := 44.5
fmt.Println(i + j) // 编译不通过
fmt.Println(i + int(j)) // 编译通过
```

# 常量
常量通过 const 关键字声明，比如：`const Pi = 3.14`。常量可以赋值给“合适的”类型，而不需要进行显式的类型转换。  

```go
const A = 1
var f float64 = A
```

# 函数

## 函数的声明
函数声明的通用语法为：  

```go
func functionname(parametername type) returntype {
    // 函数体
}
```

函数的参数列表和返回值可以为空，比如：  

```go
func funcname() {

}
```

## 多返回值和命名返回值
```go
// 多返回值
func calculate(length, width float64) (float64, float64) {
    var area = length * width
    var perimeter = (length + width) * 2
    return area, perimeter
}
```

```go
// 命名返回值
func calculate(length, width float64) (area, perimeter float64) {
    area = length * width
    perimeter = (length + width) * 2
    return // 不需要明确指定返回值，默认返回 area 和 perimeter
}
```

## 空白符
`_` 在 Golang 中为空白符，可以用来表示任何类型的任何值，常用于接收不需要的结果值。  

```go
func main() {
    // 只要面积，不要周长
    area, _ := calculate(11.2, 5.4)
    fmt.Println("area is %f", area)
}
```

# 包

## main 函数和 main 包
所有可执行的 Go 程序都必须包含一个 main 函数，这个函数就是程序执行的入口。main 函数应该放在 main 包中。

## 自定义包
属于某个包的源文件都应该放置在名称与包名相同的文件夹下。

## 导出自定义包
只有在包中使用大写字母开头的变量或者函数才能够被其它包访问。

## init 函数
init 函数是一种特殊的函数，该函数不应该有任何返回值和参数列表，它由编译器调用，我们无法显式调用。  

```go
func init() {
    // 函数体
}
```

init 函数常用于执行初始化的任务，也可以在执行前验证程序的正确性。

## 包初始化
包的初始化过程主要分为：初始化包级别的变量和调用 init 函数。

以 main 包为例，首先初始化包级别的变量，然后调用 init 函数，最终调用 main 函数。如果导入了别的包，则会按照导入包的顺序，追溯到最终的依赖包，并对该包进行标准的初始化过程，然后回到上一层包继续执行该过程，以此类推。

虽然包可能被引用多次，但是它的 init 函数只会初始化一次。

# 条件控制

## if else
基本格式为：  

```go
if condition {

} else if condition {

} else {

}
```

在 if 语句中还可以添加一个 statement 语句部分，用于在条件判断之前运行，该语句中声明的变量只在条件判断语句中有效。  

```go
if num := 10; num % 2 == 0 {
    fmt.Println(num, "is even")
} else {
    fmt.Println(num, "is odd")
}
```

## switch
与其他语言不同，case 语句之后不需要加 break。如果想在一个 case 语句执行完后把控制权转到下一个 case 语句，则可以在上一个 case 语句的最后加入 `fallthrough`。  

```go
// 包含 statement 语句
switch num := 4; num {
case 1:
    fmt.Println("num is 1")
case 2:
    fmt.Println("num is 2")
default:
    fmt.Println("num is unknown")
}

// 多表达式
switch num := 2; num {
case 1, 2:
    fmt.Println("num is valid")
default:
    fmt.Println("num is unknown")
}

// fallthrough
switch num := 1; num {
case 1:
    fmt.Println("num is 1")
    fallthrough
case 2:
    fmt.Println("num is valid")
default:
    fmt.Println("num is unknown")
}
```

# 循环控制
for 循环是 Golang 唯一的循环控制语句，没有 while 和 do while。在 for 循环中，三部分的语句都是可选的。  

```go
for i := 1; i < 10; i++ {
    fmt.Println(i)
}

for i < 10 {
    fmt.Println(i)
}
// 无限循环
for {

}
```

# 参考
> [Go 系列教程](https://studygolang.com/subject/2)