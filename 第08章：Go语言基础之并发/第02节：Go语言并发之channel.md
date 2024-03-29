# 第02节：Go语言并发之channel

##### 上一节我们讲述了Go语言并发之goroution,这节我们来给大家讲解Go语言并发之channel;

### 一、channel

单纯的将函数并发执行是没有意义的，函数与函数之间需要交换数据才能体现并发执行函数的意义。
虽然可以使用共享内存进行数据交换，但是供共享内存在不同的`goroutine`中容易发生竞态问题，为了保证数据交换的正确性，必须使用互斥量对内存进行加锁，这样做法势必造成性能问题;
Go语言的并发模型是`CSP(Communicating Sequential Processes)`，提倡 **通过通信共享内存**而不是**通过共享内存而实现通信**

如果说`goroutine`是Go程序并发执行体，`channel`就是它们之间的链接，`channel`是一个可以让一个`goroutine`发送给特定值到另一个`goroutine`的通信机制;
Go语言中的通道(channel)是一种特殊类型，通道像一个传送带或者对列，总是遵循先入先出的规则，保证收发数据顺序，每一个通道都是一个具体类型的导管，也就是声明channel的时候需要为其指定元素类型;

### 二、channel类型

`channel`是一种类型，一种引用类型，声明通道类型的格式如下:

```go
var 变量 chan 元素类型
```

举几个例子:

```go
var ch1 chan int   //声明一个传递整型的通道
var ch2 chan bool  //声明一个传递不布尔值得通道
var ch3 chan []int //声明一个传递int切片的通道
```

### 三、创建channel

通道是引用类型，通道类型的空值是`nil`

```go
var ch chan int
fmt.println(ch) // <nil>
```

声明通道后需要使用`make`函数初始化之后才能使用。
创建channel的格式如下：

```go
make(chan 元素类型，[缓冲大小])
```

channel的缓冲大小是可选的。
举几个例子:

```go
ch4 := make(chan int)
ch5 := make(chan bool)
ch6 := make(chan []int)
```

### 四、channel操作

通道有发送(send)、接收(receive)和关闭(close)三种操作;
发送和接收都使用`<-`符号;
现在我们先使用以下语句定义一个通道:
```go
ch := make(chan int)
```

##### 发送

将一个值发送到通道中

```go
ch <- 10//把10送到ch中
```

##### 接收
    
将一个值发送到通道中。

```go
x := <- ch  //从ch中接收并赋值给变量;
<-          //从ch中接收值，忽略结果;
```

##### 关闭

我们通过调用的内置`close`函数来关闭通道，

```go
close(ch)
```

关于关闭通道需要注意的事情是，只有在通知接收方goroutine所有的数据发送完毕的时候才需要关闭通，通道是可以被垃圾回收机制回收的，它和关闭文件不一样，在结束操作之后关闭文件是必要的，但关闭通道不是必须的;

关闭通道有以下特点:

1. 对一个关闭通道在发送值就会导致panic;
2. 对一个关闭的通道进行接收会一直获取值直到通道为空为止;
3. 对一个已经关闭的并且没有值得通道执行操作会得到对应类型的零值;
4. 关闭一个已经关闭的通道会导致panic;

### 五、无缓冲的通道

无缓冲的通道又称为阻塞通道，我们来看一下下面代码：

```go
func main() {
	ch := make(chan int)
	ch <- 10
	fmt.Println("发送成功")
}
```

上面代码能够通过编译，但是执行时还会出现以下错误:

```go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        C:/Users/HP/Desktop/项目部/项目部资料/go-vscode/Go/ggag/main.go:7 +0x5b
exit status 2
```

为什么会出现deadlock错误呢？
因为我们使用 `ch <- 10`创建的是无缓冲通道，无缓冲通道只有在有人接收值得时候才能发送值，就例如你的小区没有快递代收点，快递员给你打电话必须要亲手把这个送到你手上，简单的说是无缓冲的通道必须有接收才能发送;
上面代码会出现阻塞在`ch <- 10`这一行代码形成死锁，那该如何解决这个问题呢？
一种方法是启用一个`goroutine`去接收值，例如:

```go
func recv(c chan int) {
	ret := <-c
	fmt.Println("接收成功", ret)
}
func main() {
	ch := make(chan int)
	go recv(ch) // 启用goroutine从通道接收值
	ch <- 10
	fmt.Println("发送成功")
}
```

无缓冲通道上发送消息会形成阻塞，直接到另外一个`goroutine`在该通道上执行接收操作，这时值才能发送成功，两个`goroutien`将继续执行，相反如果接收操作先执行，接收方的`goroutine`将阻塞，知道另一个`goroutine`在该通道上发送另一个值。
使用五缓冲通道进行通信信号将导致发送和接收的`goroutine`同步化，因此，无缓冲通道也被称为`同步通道`

### 六、有缓冲通道

解决上面问题的方法还有一种就是使用缓冲区的通道，我们可以使用make函数初始化通道的时候为其指定通道的容量，例如：

```go
func main() {
	ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
	ch <- 10
	fmt.Println("发送成功")
}
```

只要通道容量大于零，那么该通道就是有缓冲通道，通道的容量表示通过中能存放内存的数量，就像你小区里面的快递储物柜只有那么多的格子，格子满了你就装不下了，只有取走一个快递才能放一个;
我们可以使用内置的`len`函数获取通道元素的数量，使用`cap`函数取通道的容量，虽然我们很少会这么做。

### 七、for range从通道循环取值

当向通道中发送完数据时，我们可以通过`close`函数来关闭通道。
当通道关闭后时，在往通道里面发送值会引发panlic，从该通道里接收的值一直都是类型零值，那如何判断一个通道是否被关闭了呢？
我们来看下面这个例子:

```go
// channel 练习
func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)
	// 开启goroutine将0~100的数发送到ch1中
	go func() {
		for i := 0; i < 100; i++ {
			ch1 <- i
		}
		close(ch1)
	}()
	// 开启goroutine从ch1中接收值，并将该值的平方发送到ch2中
	go func() {
		for {
			i, ok := <-ch1 // 通道关闭后再取值ok=false
			if !ok {
				break
			}
			ch2 <- i * i
		}
		close(ch2)
	}()
	// 在主goroutine中从ch2中接收值打印
	for i := range ch2 { // 通道关闭后会退出for range循环
		fmt.Println(i)
	}
}
```
从上面例子中我们看到有两种方式在接收值得时候判断该通道是否被关闭，不过我们通常使用的是`for range`的方式，使用`for range`遍历通道，当通道被关闭时就会退出`for range`

##### 单向通道

有的时候我们会将通道作为参数在多个任务函数间传递，很多时候我们在不同的任务函数中使用通道都会进行限制，比如限制函数在通道在函数中只能发送或者接受。
Go语言提供了**单向通道**来处理这种情况，例如，我们把上面例子改造一下：

```go
func counter(out chan<- int) {
	for i := 0; i < 100; i++ {
		out <- i
	}
	close(out)
}

func squarer(out chan<- int, in <-chan int) {
	for i := range in {
		out <- i * i
	}
	close(out)
}
func printer(in <-chan int) {
	for i := range in {
		fmt.Println(i)
	}
}

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)
	go counter(ch1)
	go squarer(ch2, ch1)
	printer(ch2)
}
```

其中：

* **chan<- int** 是一个只能发送的通道，可以发送但是不能接收;

* **<-chan int** 是一个只能接收的通道，可以接收但是不能发送;

在函数传参及任何赋值操作中将双向通道转换为单向通道是可以的，但反过来是不可以的。

### 八、worker pool(goroutine池)

在工作中我们通常会使用可以指定启动的goroutine数量-`worker pool`模式，控制`goroutine`的数量，防止`goroutine`泄露和暴涨;
一个简单的`work pool`示例代码如下:

```go
func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		fmt.Printf("worker:%d start job:%d\n", id, j)
		time.Sleep(time.Second)
		fmt.Printf("worker:%d end job:%d\n", id, j)
		results <- j * 2
	}
}


func main() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)
	// 开启3个goroutine
	for w := 1; w <= 3; w++ {
		go worker(w, jobs, results)
	}
	// 5个任务
	for j := 1; j <= 5; j++ {
		jobs <- j
	}
	close(jobs)
	// 输出结果
	for a := 1; a <= 5; a++ {
		<-results
	}
}
```

##### select多路复用

在某些场景下我们需要同时从多个通道接收数据，通道在接收数据时，如果没有数据时，如果没有数据可以接收将会引发阻塞。你也许会写出一下代码使用遍历方式来实现：

```go
for{
    // 尝试从ch1接收值
    data, ok := <-ch1
    // 尝试从ch2接收值
    data, ok := <-ch2
    …
}
```
这种方式虽然可以实现多个通道接收值得需求，但是运行性能可能会差很多，为了应对这种场景，Go内置了`select`关键字，可以同时相应多个通道的操作。
`select`的使用类似于switch语句，它有一系列case分支和一个默认的分支，每个case会对应一个通道的通信(接收或发送)过程，`select`会一直等待，知道某个`case`的通信操作完成时，就会执行`case`分支对应语句，具体格式如下:

```go
select{
    case <-ch1:
        ...
    case data := <-ch2:
        ...
    case ch3<-data:
        ...
    default:
        默认操作
}
```
举个小例子演示一下`select`的使用:

```go
func main() {
	ch := make(chan int, 1)
	for i := 0; i < 10; i++ {
		select {
		case x := <-ch:
			fmt.Println(x)
		case ch <- i:
		}
	}
}
```

使用`select`可以提高代码可读性：
* 可处理一个或者多个channel的发送/接收操作
* 如果多个case同时满足，select会随机挑选一个
* 对于没有casede select{}函数会一直等待，可用于阻塞main函数

### 九、总结

![images](../images/0802_panic.png)

本节我们学习了什么是channel，channel是什么类型channel的操作(发送、接收、关闭),有无缓冲通道，通道类型，worker pool等;