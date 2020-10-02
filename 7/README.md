# 第7章 接口
## 引言
* 接口类型是对其他类型行为的概括与抽象
* 通过使用接口，我们可以写出更加灵活和通用的函数，这些函数`不用绑定`在一个特定的类型实现上
* Go语言的接口的独特之处在于它是`隐式实现`:对于一个具体的类型，无须声明它实现了哪些接口，只需要提供接口所必需的方法即可->这种设计让你无须改变已有类型的实现，就可以为这些类型创建新的接口，对于那些不能修改包的类型，这一点特别有用

## 7.1 接口即约定
* 接口类型是一种抽象类型，它没有暴露所含数据的布局或内部结构，当然也没有那些数据的基本操作，它所提供的仅仅是一些方法而已
* fmt.Printf把结果发送到标准输出
* fmt.fmt.Sprintf把结果以string类型返回
* 字符串的格式化是以上两个函数中最复杂的部分，接口机制解决了在两个函数分别实现字符串的格式化的问题。
* 以上两个函数都封装了fmt.Fprintf，这个函数对结果输出到哪毫不关心
``` Go
package fmt

func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}
func Sprintf(format string, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
```
* Printf第一个实参就是os.Stdout，它属于*os.File类型
* Sprintf第一个实参不是文件，但它模拟了一个文件：&buf就是一个指向内存缓冲区的指针，与文件类似，这个缓冲区也可以写入多个字节
* 其实Fprintf的第一个形参也不是文件类型，而是io.Writer接口类型,其声明如下：
``` Go
package io

// Writer 接口封装了基础的写入方法
type Writer interface {
  // Write 从  p 向底层数据流写入 len(p) 个字节的数据
  // 返回实际写入的字节数（0 <= n <= len(p)）
  // 如果没有写完，那么会返回遇到的错误
  // 在 Write 返回 n < len(p) 时，err 必须为非 nil
  // Write 不允许修改 p 的数据，即使是临时修改
  //
  // 实现时不允许残留 p 的引用
  Write(p []byte) (n int, err error)
}
```
* io.Writer接口定义了Fprintf和调用者之间的约定。一方面，这个约定要求调用者提供的具体类型（比如*os.File或者*bytes.Buffer）包含一个与其签名和行为一致的Write方法。另一方面，这个约定保证了Fprintf能使用任何满足io.Writer接口的参数。
* Fprintf只需要能调用参数的Write函数，无需假设它写入的是一个文件还是一段内存
* 因为fmt.Fprintf仅依赖于io.Write接口所约定的方法，对参数的具体类型没有要求，所有我们可以用任何满足io.Write接口的具体类型作为fmt.Fprintf的第一个实参。
* 这种可以把一种类型替换为满足同一接口的另一种类型的特性称为`可取代性`，这也是面向对象语言的典型特性
创建一个新类型来测试这个特性
``` Go
package main
import (
	"fmt"
)
type ByteCounter int
// 约定的方面一：调用者提供的具体类型（此处是ByteCounter）包含一个与其签名行为一致的Write方法。
func (c *ByteCounter) Write(p []byte) (int, error) {
	*c += ByteCounter(len(p))
	return len(p), nil
}
func main() {
	var c ByteCounter
	c.Write([]byte("hello"))
	fmt.Println(c) 

	c = 0
	var name = "Dolly"
	// 约定的方面二：约定保证了Fprintf能使用任何满足io.Writer接口的参数。
	fmt.Fprintf(&c, "hello, %s", name)
	fmt.Println(c)
}
```
除了io.Writer之外，fmt包还有另一个重要的接口。Fprintf和Fprintln提供了一个让类型控制如何输出自己的机制。定义一个String方法就可以让类型满足这个广泛使用的接口fmt.Stringer：
``` Go
package fmt
// 在字符串格式化时如果需要一个字符串
// 那么就调用这个方法来把当前值转化为字符串
// Print 这种不带格式化参数的输出方式也是调用这个方法
type Stringer interface {
    String() string
}
```
## 7.2 接口类型
* 一个接口类型定义了一套方法，如果一个具体类型要实现该接口，那么必须实现接口类型定义中的所有方法
* io.Writer是一个广泛使用的接口，它负责所有可以写入字节的类型的抽象，包括文件、内存缓冲区、网络连接、HTTP客户端、打包器（archiver）、散列器（hasher）等。
* io包还定义了很多有用的接口。Reader就抽象了所有可以读取字节的类型，Closer抽象了所有可以关闭的类型，比如文件或者网络连接。
``` Go
package io
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}
```
通过组合已有接口得到新接口的语法称之为嵌入式接口，与嵌入式结构类似，让我们可以直接使用一个接口，而不用逐一写出这个接口所包含的方法：
``` Go
type ReadWriter interface {
    Reader
    Writer
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

```
不用嵌入式来声明io.ReadWriter：
``` Go
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```
也可以混合使用两种方式：
``` Go
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Writer
}
```
三种声明的效果都是一致的。方法定义的顺序也是无意义的，真正有意义的只有接口的方法集合
## 7.3 实现接口
* 如果一个类型实现了一个接口要求的所有方法，那么这个类型实现了这个接口
* *os.File类型实现了io.Reader、Writer、Closer和ReadWriter接口
* *bytes.Buffer实现了Reader、Writer和ReaderWriter，但没有实现Closer，因为它没有Close方法
* 为了简化表述，通常说一个具体“是一个”（is-a）特定的接口类型，这其实代表着该具体类型实现了该接口。比如，*bytes.Buffer是一个io.Writer；*os.File是一个io.ReaderWriter
* 接口的赋值规则：仅当一个表达式实现了一个接口时，这个表达式才可以赋给该接口
* 只有通过接口暴露的方法才可以调用，类型的其他方法则无法通过接口来调用
* 因为空接口类型对其实现类型没有任何要求，所以我们可以把任何值赋给空接口类型
* 靠空接口类型让fmt.Println、errorf这类的函数能够接受任意类型的参数

## 7.4 使用flag.Value来解析参数
``` Go
var period = flag.Duration("period", 1*time.Second, "sleep period")

func main() {
    flag.Parse()
    fmt.Printf("Sleeping for %v...", *period)
    time.Sleep(*period)
    fmt.Println()
}
```
## 7.5 接口值
* 从概念上来讲，一个接口类型的值（简称接口值）其实有两个部分：一个具体类型和该类型的一个值。二者称为接口的`动态类型`和`动态值`
* 接口的零值就是把它的动态类型和值都设置为nil
* 一个接口值是否是nil取决于它的动态类型
* 调用一个nil接口的任何方法都会导致崩溃
* 一般来讲，在编译时我们无法知道一个接口值的动态类型会是什么，所以通过接口来做调用必然需要使用`动态分发`
* 一个接口值可以指向多个任意大的动态值。从理论上来讲，无论动态值有多大，它永远在接口值内部（当然这只是一个理论模型；实际的实现是很不同的）
* 接口值可以用 == 和 != 操作符来做比较。如果两个接口值都是nil或者二者的动态类型完全一致且二者动态值相等（使用动态类型 == 操作符来做比较），那么两个接口值相等。因为接口值是可以比较的，所以它们可以作为map的键，也可以作为switch语句的操作数
* 在比较两个接口值时，如果两个接口值的动态类型一致，但对应的动态值是不可比较的（比如slice），那么这个比较会以崩溃的方式失败 

## 7.6 使用sort.Interface来排序
* Go语言的sort.Sort函数对序列和其中元素的布局无任何要求，它使用sort.Interface接口来指定通用排序算法和每个具体的序列类型之间的协议
* 一个原地排序算法需要知道三个信息：序列长度、比较两个元素的含义以及如何交换两个元素，所以sort.Interface接口就有三个方法：
``` Go
package main
type Interface interface {
	Len() int
	Less(i, j int)	bool
	Swap(i, j int)
}
```
* 实现了sort.Interface的具体类型不一定是切片类型；customSort是一个结构体类型
``` Go
type customSort struct {
    t    []*Track
    less func(x, y *Track) bool
}

func (x customSort) Len() int           { return len(x.t) }
func (x customSort) Less(i, j int) bool { return x.less(x.t[i], x.t[j]) }
func (x customSort) Swap(i, j int)    { x.t[i], x.t[j] = x.t[j], x.t[i] }
```
* 让我们定义一个多层的排序函数，它主要的排序键是标题，第二个键是年，第三个键是运行时间Length。下面是该排序的调用，其中这个排序使用了匿名排序函数：
``` Go
sort.Sort(customSort{tracks, func(x, y *Track) bool {
    if x.Title != y.Title {
        return x.Title < y.Title
    }
    if x.Year != y.Year {
        return x.Year < y.Year
    }
    if x.Length != y.Length {
        return x.Length < y.Length
    }
    return false
}})
```
* 尽管对长度为n的序列排序需要 O(n log n)次比较操作，检查一个序列是否已经有序至少需要n-1次比较。sort包中的IsSorted函数帮我们做这样的检查。像sort.Sort一样，它也使用sort.Interface对这个序列和它的排序函数进行抽象，但是它从不会调用Swap方法：这段代码示范了IntsAreSorted和Ints函数在IntSlice类型上的使用：
``` Go
values := []int{3, 1, 4, 1}
fmt.Println(sort.IntsAreSorted(values)) // "false"
sort.Ints(values)
fmt.Println(values)                     // "[1 1 3 4]"
fmt.Println(sort.IntsAreSorted(values)) // "true"
sort.Sort(sort.Reverse(sort.IntSlice(values)))
fmt.Println(values)                     // "[4 3 1 1]"
fmt.Println(sort.IntsAreSorted(values)) // "false"
```

## 7.7 http.Handler接口
``` Go
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```
* ListenAndServe函数需要一个服务器地址，比如"localhost:8000"，以及一个Handler接口的实例（用来接受所有的请求）。这个函数会一直运行，知道服务出错（或者启动时就失败了）时返回一个非空的错误
* 更真实的服务器会定义多个不同的URL，每一个都会触发一个不同的行为
``` Go
func (db database) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    switch req.URL.Path {
    case "/list":
        for item, price := range db {
            fmt.Fprintf(w, "%s: %s\n", item, price)
        }
    case "/price":
        item := req.URL.Query().Get("item")
        price, ok := db[item]
        if !ok {
            w.WriteHeader(http.StatusNotFound) // 404
            fmt.Fprintf(w, "no such item: %q\n", item)
            return
        }
        fmt.Fprintf(w, "%s\n", price)
    default:
        w.WriteHeader(http.StatusNotFound) // 404
        fmt.Fprintf(w, "no such page: %s\n", req.URL)
    }
}
```
* net/http包提供了一个`请求多工转发器ServeMux`，用来简化URL和处理程序之间的关联。一个ServeMux把多个http.Handler组合成单个http.Handler。
``` Go
func main() {
    db := database{"shoes": 50, "socks": 5}
    mux := http.NewServeMux()
    mux.Handle("/list", http.HandlerFunc(db.list))
    mux.Handle("/price", http.HandlerFunc(db.price))
    log.Fatal(http.ListenAndServe("localhost:8000", mux))
}

type database map[string]dollars

func (db database) list(w http.ResponseWriter, req *http.Request) {
    for item, price := range db {
        fmt.Fprintf(w, "%s: %s\n", item, price)
    }
}

func (db database) price(w http.ResponseWriter, req *http.Request) {
    item := req.URL.Query().Get("item")
    price, ok := db[item]
    if !ok {
        w.WriteHeader(http.StatusNotFound) // 404
        fmt.Fprintf(w, "no such item: %q\n", item)
        return
    }
    fmt.Fprintf(w, "%s\n", price)
}
```
当调用db.list时，等价于以db为接受者调用database.list方法。所以db.list是一个实现了处理功能的函数（而不是一个实例），因为它没有接口所需的方法，所以它不满足http.Handler接口，也不能直接传给mux.Handler。  
表达式http.HandleFunc(db.list)其实是类型转换，而不是函数调用。注意，http.HandleFunc是一个类型。它有如下定义：
``` Go
package http

type HandlerFunc func(w ResponseWriter, r *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```
HandlerFunc演示了Go语言接口机制的一些不常见特性。它不仅是一个函数类型，还拥有自己的方法，也满足接口http.Handler。它的ServeHTTP方法就调用函数本身，所以HandlerFunc就是一个让函数值满足接口的一个适配器，在这个例子中，函数和接口的唯一方法拥有同样的签名。这个小技巧让database类型可以用不同的方式来满足http.Handler接口：一次通过list方法，一次通过price方法，依次类推。  
因为这种注册处理程序的方法太常见了，所以ServeMux引入了一个HandleFunc便捷方法来简化调用，处理程序注册部分的代码可以简化为如下形式：
``` Go
mux.HandleFunc("/list", db.list)
mux.HandleFunc("/price", db.price)
```
* 对于一个更复杂的应用，多个ServeMux会组合起来，用来处理更复杂的分发需求。
* 从上面的代码很容易看出应该怎么构建一个程序：由两个不同的web服务器监听不同的端口，并且定义不同的URL将它们指派到不同的handler。我们只要构建另外一个ServeMux并且再调用一次ListenAndServe（可能并行的）。但是在大多数程序中，一个web服务器就足够了。此外，在一个应用程序的多个文件中定义HTTP handler也是非常典型的，如果它们必须全部都显式地注册到这个应用的ServeMux实例上会比较麻烦。  
所以为了方便，net/http包提供了一个全局的ServeMux实例DefaultServerMux和包级别的http.Handle和http.HandleFunc函数。现在，为了使用DefaultServeMux作为服务器的主handler，我们不需要将它传给ListenAndServe函数；nil值就可以工作。  
然后服务器的主函数可以简化成：  
``` Go
func main() {
    db := database{"shoes": 50, "socks": 5}
    http.HandleFunc("/list", db.list)
    http.HandleFunc("/price", db.price)
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}
```

## 7.8 error接口
``` Go
type error interface {
    Error() string
}
```
构造error最简单的方法是调用errors.New，它会返回一个包含指定的错误消息的新error实例。完整的error包只有如下4行代码：
``` Go
package errors

func New(text string) error { return &errorString{text} }

type errorString struct { text string }

func (e *errorString) Error() string { return e.text }
```
直接调用errors.New比较罕见，因为有一个更易用的封装函数fmt.Errorf，它还额外提供了字符串格式化功能。
``` Go
package fmt

import "errors"

func Errorf(format string, args ...interface{}) error {
    return errors.New(Sprintf(format, args...))
}
```

## 7.9 示例：表达式求值器

