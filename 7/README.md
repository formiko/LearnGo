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
