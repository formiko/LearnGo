# LearnGo
## 在未命名的结构体中声明方法
```Go
//	声明一个未命名结构体
var a struct {
	Point	//	内嵌另一个结构体类型
}
type Point struct {
	X, Y float64
}
func (p *Point) ScaleBy(factor float64) {
	p.X *= factor
	p.Y *= factor
}
```