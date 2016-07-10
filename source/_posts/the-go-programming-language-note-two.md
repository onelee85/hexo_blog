---
title: The go programming language 学习笔记二
date: 2016-06-28 08:09:23
categories: 语言
tags:
  - golang
---

# Introduction

本文为学习The go programming language的总结笔记，主要分为

- Functions
- Methods
- Interfaces

# Functions

## Function Declarations

GO函数声明的语法如下

```go
func name(parameter-list) (result-list) {
	body
}
```

在GO中，返回值是可以命名的，这样函数初始化时会为每个命名的返回值初始化一个zero value。

在GO中，有返回值的函数，必须在结束的时候有一个return语句。

以下函数声明是相同的

```go
func f(i, j, k int, s, t string)
func f(i int, j int, k int, s string, t string)

func add(x int, y int) int {return x + y}
func sub(x, y int) (z int) {z = x - y; return}
func first(x int, _ int) int {return x}
func zero(int, int) int {return 0}
```

GO函数是值传递的，函数内部对函数参数的修改，一般不会影响原始的变量值，除了传入的是引用类型，例如pointer、slice、map、function或channel。

## Recursion

GO函数是支持递归调用，本节通过一个HTML解析程序来说明。

本节使用`golang.org/x/net/html`包，其中html.Parse会返回一系列的HTML的元素，例如text，comment等等，但是，我们这个程序只关注`<name key='value'>`形式的元素。

```go
package main

import (
	"fmt"
	"os"

	"golang.org/x/net/html"
)

func main() {
	doc, err := html.Parse(os.Stdin)
	if err != nil {
		fmt.Fprintf(os.Stderr, "findlinks1: %v\n", err)
		os.Exit(1)
	}
	for _, link := range visit(nil, doc) {
		fmt.Println(link)
	}
}

//!-main

//!+visit
// visit appends to links each link found in n and returns the result.
func visit(links []string, n *html.Node) []string {
	if n.Type == html.ElementNode && n.Data == "a" {
		for _, a := range n.Attr {
			if a.Key == "href" {
				links = append(links, a.Val)
			}
		}
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		links = visit(links, c)
	}
	return links
}

//!-visit

/*
//!+html
package html

type Node struct {
	Type                    NodeType
	Data                    string
	Attr                    []Attribute
	FirstChild, NextSibling *Node
}

type NodeType int32

const (
	ErrorNode NodeType = iota
	TextNode
	DocumentNode
	ElementNode
	CommentNode
	DoctypeNode
)

type Attribute struct {
	Key, Val string
}

func Parse(r io.Reader) (*Node, error)
//!-html
*/
```

其中visit函数递归的调用自身，来完成对HTML元素的子元素的遍历。

## Multiple Return Values

GO中，支持函数有多个返回值，一般，一个返回值是结果，另外一个是标明错误情况等。

```go
package main

import (
	"fmt"
	"net/http"
	"os"

	"golang.org/x/net/html"
)

// visit appends to links each link found in n, and returns the result.
func visit(links []string, n *html.Node) []string {
	if n.Type == html.ElementNode && n.Data == "a" {
		for _, a := range n.Attr {
			if a.Key == "href" {
				links = append(links, a.Val)
			}
		}
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		links = visit(links, c)
	}
	return links
}

//!+
func main() {
	for _, url := range os.Args[1:] {
		links, err := findLinks(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "findlinks2: %v\n", err)
			continue
		}
		for _, link := range links {
			fmt.Println(link)
		}
	}
}

// findLinks performs an HTTP GET request for url, parses the
// response as HTML, and extracts and returns the links.
func findLinks(url string) ([]string, error) {
	resp, err := http.Get(url)
	if err != nil {
		return nil, err
	}
	if resp.StatusCode != http.StatusOK {
		resp.Body.Close()
		return nil, fmt.Errorf("getting %s: %s", url, resp.Status)
	}
	doc, err := html.Parse(resp.Body)
	resp.Body.Close()
	if err != nil {
		return nil, fmt.Errorf("parsing %s as HTML: %v", url, err)
	}
	return visit(nil, doc), nil
}

```

上述代码中，findLinks包含了html处理过程中的错误情况，方便调用者知道函数调用过程中是否发生错误，错误类型是什么。

前面提到GO支持命名的返回值，如下

```go
func CountWordsAndImages(url string) (words, images int, err error)
{
	resp, err := http.Get(url)
    if err != nil {
    	return
    }
    
    doc, err := html.Parse(resp.Body)
    resp.Body.Close()
    if err != nil {
    	err = fmt.Errorf("parsing HTML:%s", err)
    }
    words, images = countWordsAndImages(doc)
    return
}
```

如上所示，返回值的命名的，所以空的return语句等同于`return words, images, err`

## Errors

通常，在GO中，如果函数需要处理异常情况，会增加一个返回值err，来表示函数出错的原因。内置的error是一个interface类型，当返回的err为nil时，则表示执行成功，否则表示执行出错，可以用`fmt.Println(err)`来输出错误的信息。

大部分情况下，如果err不为nil，则函数的其他返回值是无效值，不过，有些函数的部分返回值可能是有效的，例如，`Read`函数可能会返回它能够读到的字节数，对于这种函数，编写者应该清晰地在文档中注明这种情况。

### Error-Handling Strategies

当一个函数返回错误，调用者必须能够采取正确的行为来处理不同的错误，一般分为5种情况：

#### 传播错误

一种比较常见的方法是传播错误到当前函数的上一层，由上层来处理具体地错误。在`findLinks`函数中，有如下代码

```go
resp, err := http.Get(url)
if err != nil {
	return nil, err
}
```
除了上面的直接传播错误之外，还有经过稍微整合之后的传播，如下

```go
doc, err := html.Parse(resp.Body)
resp.Body.Close()
if err != nil {
	return nil, fmt.Errorf("parsing %s as HTML: %v", url, err)
}
```
这里把出错的URL封装到err中一起传递给上层调用者。

#### 重试

第二种处理错误的方式是重试，对于可能是暂时的问题（例如网络问题），可以通过重试几次或一段时间，直到成功或者最终失败。

```go
func WaitForServer(url string) error {
	const timeout = 1 * time.Minute
    for tries := 0; time.Now().Before(deadline); tries++ {
    	_, err := http.Head(url)
        if err == nil {
        	return nil
        }
        log.Printf("server not responding(%s), retrying...", err)
        time.Sleep(time.Second << uint(tries))
    }
    return fmt.Error("server %s failed to respond after %s", url, timeout)
}
```

#### 不可能的错误---打印错误，优雅地终止程序

这种错误处理方式只适用于main package，对于普通的library package，只允许传播错误。

```go
if err := WaitForServer(url); err != nil {
	fmt.Fprintf(os.Stderr, "Site is down: %v\n", err)
    os.Exit(1)
}
```

可以用`log.Fatalf`来完成上述功能

```go
if err := WaitForServer(url); err != nil {
	log.Fatalf("Site is down: %v\n", err)
}
```

#### 打印出错误，继续以不完全功能执行

有时候，可以打印出错误，然后继续以不完全功能执行

```go
if err := Ping(); err != nil {
	fmt.Fprintf(os.Stderr, "ping failed :%v; networking disabled\n", err)
}
```

上述程序打印出网络问题，然后在没网的条件下继续执行。

#### 完全忽略错误

少数情况下，我们可以完全地忽略错误。

```go
dir, err := ioutil.TempDir("", "scratch")
if err != nil {
	return fmt.Errorf("failed to create temp dir: %v", err)
}

os.RemoveAll(dir)
```
最后的`os.RemoveAll(dir)`可能会出错，但是操作系统可以定期地清理临时文件夹，因此，这种情况下可以忽略错误，但是，如果要确定要忽略错误，则需要在代码中表明这样做的用意。

### End of File(EOF)

在文件IO中，一般EOF并不是一种错误，只是代表文件中的数据已经读完了。对于EOF，调用者一般需要做特殊的处理。

```go
in := bufio.NewReader(os.Stdin)
for {
	r, _, err := in.ReadRune()
    if err == io.EOF {
    	break
    }
    
    if err != nil {
    	return fmt.Errorf("read failed: %v", err)
    }
}
```

## Function Values

在GO中，函数也能是值类型赋值给其他的变量，例如

```go
func square(n int) int
func negative(n int) int
func product(m, n int) int

f := square
fmt.Println(f(3))

f = negative
fmt.Println(f(3))

f = product // compiler error: f is func(int) int
```
函数的zero value是nil，调用nil的函数会panic

```go
var f func(int) int
f(3)  // f is nil, panic
```

函数可以作为另一个函数的参数，从而可以方便的自定义一些行为，例如

```go
func add1(r rune) rune {return r + 1}
fmt.Println(strings.Map(add1, "HAL-9000")) // "IBM.:111"
fmt.Println(strings.Map(add1, "VMS")) //"WNT"
fmt.Println(strings.Map(add1, "Admix")) //"Benjy"
```
## Anonymous Functions

匿名函数和普通函数类似，除了它在func之后不带名字，例如

```go
strings.Map(func (r rune) rune {return r + 1}, "HAL-9000")
```
另外，匿名函数是有状态的，如下

```go
func squares() func() int {
	var x int
    return func() int {
    	x++
        return x * x
    }
}

func main() {
	f := squares()
    fmt.Println(f())  // 1
    fmt.Println(f())  // 4
    fmt.Println(f())  // 9
    fmt.Println(f())  // 16
}
```

可以看出，x的值看起来是被保存下来，每次都是在上次的基础上加1，这种类型的function value，也称为闭包。

### Caveat: Capturing Iteration Variables

本节通过一个例子来说明

```go
var rmdirs []func()
for _, d := range tempDirs() {
	dir := d
    os.MkdirAll(dir, 0755)
    rmdirs = append(rmdirs, func() {os.RemoveAll(dir)})
}

for _, rmdir := range rmdirs {
	rmdir()
}
```

上面为什么需要用`dir: = d`赋值语句了，直接把dir传进`MkdirAll`不就好了吗？

在上面的代码中，for循环创建了一个新的作用域，其中dir被声明了，所有的在这个循环内的function value都会共享同一份dir的存储空间，因此，如果不采用上述代码，那么，传进function value的dir就会全部是一样的。

下面的代码也会有上述问题

```
var rmdirs []func()
dirs := tempDirs()
for i:= 0; i < len(dirs); i++ {
    os.MkdirAll(dir[i], 0755)
    rmdirs = append(rmdirs, func() {os.RemoveAll(dir)})
}

for _, rmdir := range rmdirs {
	rmdir()
}
```

其中，i的值也是循环内部所有function value共享的，导致最终i都是等于`len(dirs) - 1`。

## Variadic Functions

可变参数函数指的是函数的参数是可变的，像fmt.Printf那种。

```go
func sum(vals ...int) int {
	total := 0
    for _, val := range vals {
    	total += val
    }
    return total
}

fmt.Println(sum())
fmt.Println(sum(3))
fmt.Println(sum(1, 2, 3, 4))

values := []int{1, 2, 3, 4}
fmt.Println(sum(values...)) //可以把slice转成函数参数转进去
```
上述调用会隐式地创建一个array，然后传递整个array对应的slice进入函数。

可变参数的函数和传递slice参数的函数还是不一样的，如下

```go
func f(...int) {}
func g([]int) {}
fmt.Printf("%T\n", f) //"func(...int)"
fmt.Printf("%T\n", g) //"func([]int)"
```
可变参数函数通常用来字符串格式化，例如

```go
func errorf(linenum int, format string, args ...interface{}) {
	fmt.Fprintf(os.Stderr, "Line %d:", linenum)
    fmt.Fprintf(os.Stderr, format, args...)
    fmt.Println(os.Stderr)
}

linenum, name := 12, "count"
errorf(linenum, "undefined: %s", name)
```
## Deferred Function Calls

defer语句的含义是函数和参数表达式会在语句执行的时候会被计算出来，但是函数真正的执行会在调用defer语句的函数结束后才会执行，包括函数的return语句，panic等。在一个函数中，可以有任意个defer语句，它们最终的执行是它们defer的相反顺序。

defer语句可以经常用来成对的操作，例如open和close，connect和disconnect，lock和unlock，可以保证资源会被释放。defer语句的正确的顺序为当一个资源被成功获取之后。

```go
func title(url string) error {
	resp, err := http.Get(url)
    if err != nil {
    	return err
    }
    defer resp.Body.Close()
    
    ct := resp.Header.Get("Content-Type")
    if ct != "text/html" && !strings.HasPrefix(ct, "text/html;") {
    	return fmt.Errorf("%s has type %s, not text/html", url, ct)
    }
    
    doc, err := html.Parse(resp.Body)
    if err != nil {
    	return fmt.Errorf("parsing %s as HTML: %v", url, err)
    }
    
    return nil
}

func ReadFile(filename string) ([]byte, error) {
	f, err := os.Open(filename)
    if err != nil {
    	returun nil, err
    }
    defer f.Close()
    return ReadAll(f)
}

var mu sync.Mutex
var m = make(map[string]int)

func lookup(key string) int {
	mu.Lock()
    defer mu.Unlock()
    return m[key]
}
```
defer语句也可以用来模拟"on entry"和"on exit"函数，如下

```go
func bigSlowOperation() {
	defer trace("bigSlowOperation")()
    time.Sleep(10 * time.Second)
}

func trace(msg string) func() {
	start := time.Now()
    log.Printf("enter %s", msg)
    return func() {
    	log.Printf("exit %s (%s)", msg, time.Slice(start))
    }
}
```
每次bigSlowOPeration被调用，它会打印entry和exit消息。

defer的匿名函数可以在函数返回前，修改返回值。

```go
func tuple(x int) (result int) {
	defer func() {result += x}
    return double(x)
}
fmt.Println(triple(4))
```
注意，defer的语句的最终调用是在函数结束的时候，有时候因为这个特性可能会出现问题

```go
for _, filename := range filenames {
	f, err := os.Open(filename)
    if err != nil {
    	return err
    }
    def f.Close()
}
```
for循环中的所有文件不会马上被关闭，只有当函数退出时才会关闭，这可能会造成资源的浪费，可以修改如下

```go
for _, filename := range filenames {
	if err := doFile(filename); err != nil {
    	return err
    }
}

func doFile(filename string) error {
	f, err := os.Open(filename)
    if err != nil {
    	return err
    }
    def f.Close()
}
```
上述方法可以有效地解决这个问题，文件打开使用完后，马上就会被关闭。

## Panic

GO的类型系统可以在编译期间检查很多错误，但是像数组的越界，空指针，需要在运行时做检查，当GO检测到这些错误的时候，它会panic。

一个典型的panic的包括以下流程，如下

1. 正常的执行终止
2. 所有defer的语句执行
3. 打印panic消息，包括panic值，对每个goroutine，打印stack trace

内置的panic的函数也可以显式地被调用，它可以接受任意类型的参数。panic一般发生在一些不可能的场景的情况下，例如

```go
switch s := suit(drawCard()); s {
	case "Spades":
    case "Hearts":
    case "Diamons":
    case "Clubs":
    default:
    	panic("fmt.Sprintf("invalid suite %q")", s))
}
```
由于GO的panic会导致程序终止，因此，一般用于不可恢复的错误。对于预期内的错误，采用一般的错误处理流程即可。

## Recover

通常终止程序是处理panic的正确方式，但有时候也可以做一定程度的recover，例如，退出前清理，对于一个web服务器来讲，如果发生了非预期的错误，需要关闭链接，而不是让客户端hang住。

如果内置的recover函数被defer的函数调用，并且调用defer语句的函数panic了，那么recover函数会结束当前的panic状态，并且返回panic值。如果没有panic，recover函数不起任何作用，并且返回nil。

```go
func Parse(input string) (s *Syntax, err error) {
	defer func() {
    	if p := recover(); p != nil {
        	err = fmt.Errorf("internal error: %v", p)
        }
    }
}
```
在Parse中的defer函数从panic中recover，它使用recover返回的panic值来构建错误的消息。除了panic值之外，还有打印出`runtime.Stack`信息。

最好不要recover从别的package跑出来的panic，因为无法知道是否应该recover，也不很难知道应该以何种方式recover。

# Methods

在GO中，实现面向对象编程的方法，即一个对象是指一个拥有method的值或变量，method指和指定类型关联的函数。

## Method Declarations

method的定义比普通函数多了一个部分，即函数名前面有一个额外的参数，这个参数表明这个函数所属的类型。例如

```go
package geometry
import "math"
type Point struct {X, Y float64}
func Distance(p, q Point) float64 {
	return math.Hypot(q.X-p.X, q.Y-p.Y)
}

func (p Point) Distance(q Point) float64 {
	return math.Hypot(q.X-p.X, q.Y-p.Y)
}
```
上面额外的参数p称为method receiver，在GO中不用使用特殊的this或self来命名method receiver，因为receiver的名称会被经常使用，因此，建议用短的变量名来定义它。
