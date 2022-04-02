---
title: Go中的Context
date: 2020-09-22 14:27:26
update: 2020-11-13 14:27:26
categories: [Go]
tags:
    - Go
    - Context
    - 上下文
    - 架构设计
---
### 原文信息

[@ricardo.linck](https://levelup.gitconnected.com/@ricardo.linck)
[原文:Context in Golang!](https://levelup.gitconnected.com/context-in-golang-98908f042a57)

<!-- more -->
---

Golang应用程序使用Contexts来进行控制与管理非常关健的应用可靠性,例如在[concurrent programming](https://levelup.gitconnected.com/goroutines-and-channels-concurrent-programming-in-go-9f9f8495c34d)中的数据共享与取消.这听起来似乎很琐碎,但实际并非如此.在Golang中Contexts的入口点是`context`包.它非常有用,并且可能是整个语言功能最多的包之一.如果您还没有遇到任何有关上下文之类的东西,您大概很快就会遇到(或者您只是没有注意到它).上下文的用法非常广泛,以至于多数软件包都依赖它,并也假设您也会这样做.它绝对是Golang生态系统中的一个关键组件.

这里是`context`软件包的官方文档 https://golang.org/pkg/context/.它真的很棒,并且包含了很多例子.为了尝试拓展它们,来让我们看看我在真实场景是如何使用的.


### 使用上下文来包含您的数据

一个常见的使用上下文的用户之一是用于共享数据,或者使用请求作用域的值.当您有多个函数并且想在他们之间共享数据,您可以使用上下文.

最简单的方法是使用函数 context.WithValue.这个函数会根据父上下文创建一个新的上下文,并对您指定的Key添加一个值.您可以把内部实现看做是上下文的内部是一个map.

所以您可以添加或者使用Key来找回Values,这是非常强大的,因为它允许您在上下文内部存储任何类型的数据.

下面是一个用上下文添加和找回数据的例子.


``` go
package main

import (
	"context"
	"fmt"
)

func main() {
	ctx := context.Background()
	ctx = addValue(ctx)
	readValue(ctx)
}

func addValue(ctx context.Context) context.Context {
	return context.WithValue(ctx, "key", "test-value")
}

func readValue(ctx context.Context) {
	val := ctx.Value("key")
	fmt.Println(val)
}
```
> 在上下文中添加和找回值

在Context包设计背后有一种重要的方面,任何操作都会返回一个新的context.Context结构.这意味着您需要记住运行时要用带的返回值,并尽可能的使用新的上下文覆盖旧上下文.

这是来自于不可更改性(immutability)的关键设计.如果您想了解更多的关于gokang中的不可更改性,您可以阅读我的[这篇文章](https://levelup.gitconnected.com/immutability-in-golang-7a13199060bb)

要创建一个带有取消功能的上下文,您只需要使用函数`context.WithCancel(ctx)`将您的上下文通过参数传递进去.这会返回一个新的上下文与一个取消函数.您只需要调用取消函数,就可以取消上下文.

下面这个例子来自于[对冲请求(Hedged Request)](https://medium.com/swlh/hedged-requests-tackling-tail-latency-9cea0a05f577)实现的带有取消功能的上下文.来让我们快速的回顾一下[对冲请求(Hedged Request)](https://medium.com/swlh/hedged-requests-tackling-tail-latency-9cea0a05f577):我们对一个外部服务发起请求,如果在我们定义的时间没有返回,我们会发出第二个请求.当请求返回了,所有其他的请求都会被取消掉.

``` go
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"net/http"
	neturl "net/url"
	"time"
)

func queryWithHedgedRequestsWithContext(urls []string) string {
	ch := make(chan string, len(urls))
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	for _, url := range urls {
		go func(u string, c chan string) {
			c <- executeQueryWithContext(u, ctx)
		}(url, ch)

		select {
		case r := <-ch:
			cancel()
			return r
		case <-time.After(21 * time.Millisecond):
		}
	}

	return <-ch
}

func executeQueryWithContext(url string, ctx context.Context) string {
	start := time.Now()
	parsedURL, _ := neturl.Parse(url)
	req := &http.Request{URL: parsedURL}
	req = req.WithContext(ctx)

	response, err := http.DefaultClient.Do(req)

	if err != nil {
		fmt.Println(err.Error())
		return err.Error()
	}

	defer response.Body.Close()
	body, _ := ioutil.ReadAll(response.Body)
	fmt.Printf("Request time: %d ms from url%s\n", time.Since(start).Nanoseconds()/time.Millisecond.Nanoseconds(), url)
	return fmt.Sprintf("%s from %s", body, url)
}
```

每个请求都是在一个独立的go routine中触发的.这个上下文被传递给所有触发的请求.唯一的逻辑就是将上下文传播给Http client.以便当取消函数辈调用时,可以优雅的取消请求和底层连接.对于接受context.Context作为参数的函数来说,这是一个非常常见的模式,它们要么主动地对上下文采取行动(比如检查它们是否已经取消),要么将它们传递给处理它的底层函数(本例中是通过http.Request的Do函数接受上下文)

### 超时上下文

在处理外部请求时,超时是一种非常常见的模式,类似通过Http或gRPC查询数据库或者从其他服务中获取数据.使用Context包处理这些产经非常简单.您所需要做的就是调用函数`context.WithTimeout(ctx,time)`,传递您的上下文与实际的超时时间,类似这样

``` go
ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
```

您仍然可以接受到取消函数,以防您想手动触发它.它的工作方式与普通的超时上下文相同.

> 一个好做法是,使用defer调用取消函数,避免内存泄露

这个例子的行为非常直接.如果超时了,上下文会被取消.在HTTP调用的情况下,它的工作原理与上面的例子基本相同

### gRPC

Context是gRPC在golang的实现中的一个基本部分.它即用来共享数据(如何取消[元数据](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md))也用来控制流量,类似于取消流或请求.这是我的两个例子,来自于[GitHub存储库](https://github.com/RicardoLinck/grpc-go).

#### Metadata

``` go
func (*server) Sum(ctx context.Context, req *calculatorpb.SumRequest) (*calculatorpb.SumResponse, error) {
	log.Printf("Sum rpc invoked with req: %v\n", req)
	md, _ := metadata.FromIncomingContext(ctx)
	log.Printf("Metadata received: %v", md)
	return &calculatorpb.SumResponse{
		Result: req.NumA + req.NumB,
	}, nil
}
```
> Server implementation receiving metadata

```go
func sum(c calculatorpb.CalculatorServiceClient) {
	req := &calculatorpb.SumRequest{
		NumA: 3,
		NumB: 10,
	}
	ctx := metadata.AppendToOutgoingContext(context.Background(), "user", "test")
	res, err := c.Sum(ctx, req)
	if err != nil {
		log.Fatalf("Error calling Sum RPC: %v", err)
	}
	log.Printf("Response: %d\n", res.Result)
}
```
> Client implementation sending metadata

#### Calcellation:

``` go
func (*server) Greet(ctx context.Context, req *greetpb.GreetRequest) (*greetpb.GreetResponse, error) {
	log.Println("Greet rpc invoked!")

	time.Sleep(500 * time.Millisecond)

	if ctx.Err() == context.Canceled {
		return nil, status.Error(codes.Canceled, "Client cancelled the request")
	}

	first := req.Greeting.FirstName
	return &greetpb.GreetResponse{
		Result: fmt.Sprintf("Hello %s", first),
	}, nil
}
```
> Server implementation handling context cancellation

``` go

func greetWithTimeout(c greetpb.GreetServiceClient) {
	req := &greetpb.GreetRequest{
		Greeting: &greetpb.Greeting{
			FirstName: "Ricardo",
			LastName:  "Linck",
		},
	}
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()
	res, err := c.Greet(ctx, req)
	if err != nil {
		grpcErr, ok := status.FromError(err)

		if ok {
			if grpcErr.Code() == codes.DeadlineExceeded {
				log.Fatal("Deadline Exceeded")
			}
		}

		log.Fatalf("Error calling Greet RPC: %v", err)
	}
	log.Printf("Response: %s\n", res.Result)
}
``` 
### OpenTelemetry

`OpenTelemetry `还严重依赖于上下文来实现所谓的**上下文传播(Context Propagation)**.这是一种将不同系统中请求捆绑起来的做法.实现方式是将Span信息`注入(Inject)`到上下文中,作为您使用的协议的一部分(例如HTTP或gRPC).在另一个服务上,您需要`提取(Extrace)`Span信息.我在两篇文章中写过关于OpenTelemetry的文章,您可以在之类找到[part 1](https://medium.com/swlh/distributed-tracing-with-opentelemetry-part-1-6719df95a364),[part 2](https://levelup.gitconnected.com/distributed-tracing-with-opentelemetry-part-2-cc5a9a8aa88c).在这里您可以找到更多的关于OpenTelemetry的信息,以及使用gRPC和HTTP的例子.

### 最后的一些想法

上下文是作为Golang基本特性的一部分.因此理解并知道如何使用它们是非常重要的.`Context`包提供了一个非常简单和轻量级的API来与这个关键组件进行交互.关于`context.Context`的另一个重要的事情是,它可以用于多种事情.我们再这篇文章中涉及到了很多场景,在其中一些场景中,一个单一的上下文可以用来控制和携带范围值.这使得上下文成为创建可靠和简单代码的一个非常重要和强大的工具.
