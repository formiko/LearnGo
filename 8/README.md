# goroutine和通道

## 8.1 goroutine
* 在Go里，每一个并发执行的活动称为goroutine
* 当一个程序启动时，只有一个goroutine来调用main函数，称它为主goroutine
* 新的goroutine通过go语句进行创建。语法上，一个go语句是在普通的函数或者方法调用前加上go关键字前缀。go语句使函数在一个新创建的goroutine中调用。go语句本身的执行立即完成
``` Go
f()	// 调用 f(); 等待它返回
go f()	// 新建一个调用 f() 的 goroutine，不用等待
```

## 8.2 示例：并发时钟服务器

## 8.3 示例：并发回声服务器

## 8.4 通道

