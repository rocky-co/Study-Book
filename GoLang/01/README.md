# 1. go语言基础数据类型及定义

## 1.1 创建 Go File

Go File 分为两种，一种为 Empty file ，另一种为 Simple Application

## 1.2 Hello word

创建一个 Simple Application 后，输入：

```go
package main
import "fmt"
func main() {
	fmt.Println("Hello word!")
}
```

## 1.3 变量

### 1.3.1 变量声明

变量声明使用：`var 变量名 数据类型`，如：

```go
var a int
var s string
```

go 中的变量声明过后会自动定义一个初始值，我们可以直接输出这个初始值：

```go
var a int
var s string
fmt.Println(a, s) // 0 
```

- 整型和浮点型变量的默认值为 0。
- 字符串变量的默认值为空字符串。
- 布尔型变量默认为 bool。
- 切片、函数、指针变量的默认为 nil。

### 1.3.2 变量赋值

在定义变量时可以进行赋值，赋值的基本语法为：

```go
var 变量名 变量类型 = 变量值
```

同时也可以定义和赋值多个变量：

```go
var a, b int = 3, 4
fmt.Println(a, b)
```

定义时的变量类型可以忽略：

```go
var a, b, c = 3, 4, "yo~"
fmt.Println(a, b, c)
```

定义变量可以用 `:=` 来代替关键字 `var`：

```go
a, b, c, s := 3, 4, true, "def"
b = 5
fmt.Println(a, b, c, s)
```

> 注意： 在函数外部也可以定义 **包内(package)** 变量，但是不能采用 `:=` 简写方式  

也可以使用 `()` 来包裹变量，从而简写：

```go
var (
	aa = 3
    ss = "kkk"
    bb = true
)
```

### 1.3.3 变量类型强制转换

当我们想要进行数据类型的强制转换时，用 `数据类型(数据)` 即可进行转换，如：

```go
var a, b int = 3, 4
var c int
c = int(math.Sqrt(float64(a*a + b*b)))
fmt.Println(c) // 5
```

> math.Sqrt() 内的参数必须是浮点型，所以要用 float64 强行转化，变量 c 是 int 型，所以给 c 赋值时必须使用 int() 转化。

## 1.4  定义函数

使用关键字 `func` 定义函数：

```go
func variableZeroValue() {
	fmt.Println("Yo~")
}
```

直接引用函数名即可调用函数（main函数默认执行）：

```go
func main() {
	variableZeroValue()
}
```

## 1.5 格式化字符串

go 语言与 C 语言一样提供输出格式化字符串的功能，如当我们默认输出字符串时，字符串不带双引号

```go 
var s string
fmt.Printf(s) // 空字符串，不可见
```

当我们使用格式化输出：

```go
var s string
fmt.Printf("%q", s) // 输出：""
```

> 注意：格式化字符串是 `Printf()` 方法独有的，其余的输出如 `Println`， `Print` 都不能使用格式化输出字符。

## 1.6 内建变量类型 

- bool, string
- (u)int, (u)int8, (u)int16, (u)int32, (u)int64, unitptr
- byte, rune(字符型，长度32位)【没有char类型】
- float32, float64, complex64（复数）, complex128

i 可以被当作虚数单位（$\sqrt{-1}$）：

```go
c := 3 + 4i // c 为一个复数（3为实部，4为虚部）
fmt.Println(cmplx.Abs(c)) // 5
```

借此可以验证 [欧拉公式](https://zh.wikipedia.org/wiki/欧拉公式)：

```go
result := cmplx.Pow(math.E, 1i*math.Pi) + 1
fmt.Println(result) // (0+1.2246467991473515e-16i) 
```

> `1i` 表示强制定义变量 `i` 为虚数

这里的结果之所以不正确，是因为 `math.E` 遵循浮点数规则，使用 `math.Pow()` 函数作为底数的话会存在精度问题，所以 go 语言中提供了一个 `cmplex.Exp()` 方法可以将 e 直接作为底数使用，参数位传入指数【仍存在问题】：

```go
result := cmplx.Exp(1i*math.Pi) + 1
fmt.Println("%f.3", result)
```

## 1.7 常量

### 1.7.1 常量的定义

```go
const filename = "abc.txt"
```

> 在 go 语言中定义常量不推荐使用全大写

### 1.7.2 常量的自动转化

常量可以自动转化为各类型的数值，不需要手动转换：

```go
const a, b = 3, 4
var c int
c = int(math.Sqrt(a*a + b*b))
fmt.Println(filename, c) // 5
```

### 1.7.3 特殊常量

**枚举类型：**

```go
const (
    cpp = iota
    _
    python
    golang
    javascript
)
fmt.Println(cpp, javascript, python, golang) // 0, 4, 2, 3
```

`iota` 是一个从 0 开始的自增值，也可以将其进行运算：

```go
const (
    b = 1 << (10 * iota)
    kb
    mb
    gb
    tb
    pb
)
fmt.Println(b, kb, mb, gb, tb, pb) 
// 1 1024 1048576 1073741824 1099511627776 1125899906842624
```

# 2. 条件、分支与循环

## 2.1 if ... else ...

go 语言的 if 语句不需要使用括号，如读取一个文件并加载，如果出错则输出错误：

```go
const filename = 01
contents, err := ioutil.ReadFile(filename)
if err != nil {
    fmt.Println(err)
} else {
    fmt.Printf("%s\n", contents)
}
```

## 2.2 if 句的赋值操作

我们可以在 if 语句中进行赋值操作：

```go
const filename = 01
if contents, err := ioutil.ReadFile(filename); err != nil {
    fmt.Println(err)
} else {
    fmt.Printf("%s\n", contents)
}
fmt.Println(contents) // undefined: contents
```

> 注意：如果使用简写，那么 contents 与 err 变量的作用域只存在于判断语句内部，在外部调用会报错

## 2.3 switch

go 语言的 switch 语句与其他语言类似：

```go
func eval(a, b int, op string) int {
	var result int
	switch op {
	case "+":
		result = a + b
	case "-":
		result = a - b
	case "*":
		result = a * b
	case "/":
		result = a / b
	default:
		panic("unsupported operator: " + op) // panic 产生一个报错信息
	}
	return result  
}
```

> switch 会自动 break，除非使用 fallthrough

## 2. for 循环

### 2.4.1 一般形式

基本语法：

```go
for init; condition; post { }
```

- init： 一般为赋值表达式，给控制变量赋初值
- condition： 关系表达式或逻辑表达式，循环控制条件
- post： 一般为赋值表达式，给控制变量增量或减量

利用 for 循环创建一个转换为二进制的函数方法：

```go
func convertToBin(n int) string {
	result := ""
	for ; n > 0; n /= 2 {
		lsb := n % 2
		result = strconv.Itoa(lsb) + result
	}
	return result
}

func main() {
	fmt.Println(
		convertToBin(5),  // 101
		convertToBin(13), // 1011
	)
}
```

### 2.4.2 类while循环

基本语法：

```go
for condition { }
```

利用 for 循环读取文件的每一行信息：

```go
func printFile(filename string) {
	file, err := os.Open(filename)
	if err != nil {
		panic(err)
	}
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}
}
```

### 2.4.3 死循环

基本语法：

```go
for { }
```

# 3. 函数

- 返回值类型写在最后面
- 可返回刀割之
- 函数作为参数
- 没有默认参数，可选参数，只有可变参数列表

## 3.1 函数定义

基本语法：

```go
func 函数名 [(形参名 形参类型, ...)] [[返回值变量名] 返回值类型, ...] { ... ... }
```

> 当传入的多个形参都为同一类型时，可以只写最后一个形参的形参类型，之前的形参会默认与最后一个形参的数据类型相同

## 3.2 函数的多个返回值

### 3.2.1 基本用法

一个函数可以有单个返回值，也可以有多个返回值，如要使用一个函数返回除法运算的结果和余数：

```go
func div(a, b int) (int, int) {
	return a / b, a % b
}

func main() {
	fmt.Println(div(13, 3)) // 4 1
}
```

### 3.2.2 为返回值命名

为返回值命名可以让函数的使用者更清楚返回值的意义，同时还可以使编辑器自动生成变量名：

```go
func div(a, b int) (q, r int) { // 这里a、q都没有赋数据类型，默认跟后面一样为int
	return a / b, a % b
}

func main() {
	q, r := div(13, 3)
	fmt.Print(q, r) // 4 1
}
```

同时如果对返回值进行命名，在函数体内进行赋值操作后，末尾直接使用 return 就可以将结果返回：

```go
func div(a, b int) (q, r int) {
	q = a / b
	r = a % b
	return
}

func main() {
	q, r := div(13, 3)
	fmt.Print(q, r) // 4 1
}
```

> 为返回值命名不推荐使用在复杂函数内，否则返回值的位置过于混乱

### 3.2.3 获取单个返回值

如果只想使用返回函数的单个值，可以利用下划线 `_` 来取代不需要使用的值

```go
func main() {
	q, _ := div(13, 3) // 只使用一个值
	fmt.Print(q)
}
```

> 因为 go 语言对变量的使用相对严格，如果我们定义了一个 r 去接受第二个返回值，但不实用变量 r ，则系统会抛出错误，所以使用下划线 `_` 可以取出一个不被使用的值。

## 3.3 函数式编程

Go语言允许将函数作为一个参数传入到另一个函数中：

![1568000333424](.\01.assets\1568000333424.png)

```go
func apply(op func(int, int) int, a, b int) int {
	return op(a, b)
}

func main(){
   result2 := apply(func(a int, b int) int {
       return int(math.Pow(
           float64(a), float64(b)))
   }, 3, 4)
   fmt.Println(result2) // 81
}
```

## 3.4 可变参数列表

```go
func sum(numbers ...int) int {
	s := 0
	for i := range numbers {
		s += numbers[i]
	}
	return s
}

func main(){
	fmt.Println("可变参数列表的累加结果：", sum(1, 2, 3, 4)) // 10
}
```

# 4. 指针

```go
var a int = 2
var pa *int = &a
*pa = 3
fmt.Println(a) // 2
```

> Go语言的指针不能运算

## 4.1 参数传递

Go语言只有值传递一种方式，使用指针可以减少内存的占用

![1568001230648](.\01.assets\1568001230648.png)

示例：使用指针交换a，b的值

```go
func swap(a, b *int) {
	*b, *a = *a, *b
}
func mian(){
    a, b := 2, 3
	swap(&a, &b)
	fmt.Printf("a=%d, b=%d", a, b) // a=3, b=2
}
```

