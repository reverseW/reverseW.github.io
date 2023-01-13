---
layout: post
title:  "Go语言channel的输出顺序"
date:   2023-1-12 22:51:44 +0800
categories: programming
---


关于有缓冲和无缓冲``channel``，网上有许多介绍。
简单来讲，有缓冲的``channel``提供了一个有限长度的有序通知队列，生产者在队列充满前都是无阻塞的向里面填入内容，
且当队列非空时消费者可以无阻塞的从``channel``中接收通知，队列空时消费者阻塞。
而无缓冲的``channel``提供的是阻塞的通知机制，生产者在向其中填入内容时可能会阻塞，
阻塞的条件是消费者此时是否已经阻塞在``channel``上等待接收，如果没有则生产者阻塞。


但是，关于无缓冲``channel``的通知顺序，网上没有太多介绍。
本文通过简单的实验确认了，无缓冲``channel``的通知顺序同样是按照生产者发出通知的顺序来的。
不过如果生产者本身发起通知的顺序不定，则``channel``的输出顺序也会不定。

## 1 生产者按序写入channel

实验代码如下。这段代码对一个无缓冲``channel``中按照每100ms的间隔向里面写入，标号顺序为0~9。
在等待足够长的时间之后（2s），再反复从``channel``取出标号。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int)

	for i := 0; i < 10; i++ {
		go func(idx int) {
			ch <- idx
		}(i)
		time.Sleep(time.Millisecond * 100)
	}

	time.Sleep(time.Second * 2)

	for i := 0; i < 10; i++ {
		fmt.Printf("%v\n", <-ch)
	}
}
```

反复运行，可以看到总是有如下输出：

```
$ go run main.go
0
1
2
3
4
5
6
7
8
9
```

说明无缓冲``channel``的通知顺序同样是按照生产者发出通知的顺序来的。


## 2 生产者无序写入channel

使用强制睡眠并不能反映真实代码中常见的多``goroutine``竞争``channel``的场景。
于是，去掉上述代码的部门睡眠，得到如下代码：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int)

	for i := 0; i < 10; i++ {
		go func(idx int) {
			ch <- idx
		}(i)
	}

	time.Sleep(time.Second * 2)

	for i := 0; i < 10; i++ {
		fmt.Printf("%v\n", <-ch)
	}
}
```

运行可以发现，输出的顺序并没有按照0~9的顺序来。
由于``goroutine``的启动与调度顺序并不完全按照代码调用顺序来，所以``channel``的输出也不会按照0~9的顺序出现。
实际打印出的顺序就是生产者``goroutine``中写入``channel``的顺序。
多次运行，其中两次的结果如下：

```
$ go run main.go
3
0
1
2
6
4
5
7
8
9
```

```
$ go run main.go
9
0
1
2
3
4
5
6
7
8
```

本文通过代码实验了``channel``的输出顺序，确定了``channel``具有保序的特性。