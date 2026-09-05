---
title: Building a Web Framework in Go - Gee Day 7 Panic Recover
description: >-
  A 7-day tutorial on implementing a Web framework from scratch in Go (7 days implement golang web framework from
  scratch tutorial). Build a Web framework from scratch with Go/golang, with Gin as the design prototype. This
  article explains how to add an error handling mechanism to the Web framework.
date: '2020-01-09 09:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: gee-day7/go-panic.png
lang: en
---

This is the seventh part of the [7 Days Go Web Framework Tutorial Series from scratch](https://geektutu.com/post/gee.html).

- Implement the error handling mechanism.

## panic

In Go, the common way to handle errors is to return an error and let the caller decide what to do next. But for unrecoverable errors, you can trigger a panic manually; of course, a panic is also triggered automatically when errors such as an array index out of range occur while the program is running. A panic stops the currently running program and exits.

Here is an example of triggering a panic manually:

```go
// hello.go
func main() {
	fmt.Println("before panic")
	panic("crash")
	fmt.Println("after panic")
}
```

```bash
$ go run hello.go

before panic
panic: crash

goroutine 1 [running]:
main.main()
        ~/go_demo/hello/hello.go:7 +0x95
exit status 2
```

Here is a panic triggered by an array index out of range:

```go
// hello.go
func main() {
	arr := []int{1, 2, 3}
	fmt.Println(arr[4])
}
```

```bash
$ go run hello.go
panic: runtime error: index out of range [4] with length 3
```

## defer

A panic causes the program to be aborted, but before exiting, the tasks already deferred on the current goroutine are processed first, and only then does the program exit. The effect is similar to `try...catch` in Java.

```go
// hello.go
func main() {
	defer func() {
		fmt.Println("defer func")
	}()

	arr := []int{1, 2, 3}
	fmt.Println(arr[4])
}
```

```go
$ go run hello.go 
defer func
panic: runtime error: index out of range [4] with length 3
```

You can defer multiple tasks. Multiple deferred tasks in the same function are executed in reverse order, that is, the last deferred task is executed first.

Here, after the deferred tasks finish executing, the panic is still thrown, causing the program to end abnormally.

## recover

Go also provides the recover function, which prevents the entire program from terminating because of a panic. The recover function only takes effect inside a defer.

```go
// hello.go
func test_recover() {
	defer func() {
		fmt.Println("defer func")
		if err := recover(); err != nil {
			fmt.Println("recover success")
		}
	}()

	arr := []int{1, 2, 3}
	fmt.Println(arr[4])
	fmt.Println("after panic")
}

func main() {
	test_recover()
	fmt.Println("after recover")
}
```

```go
$ go run hello.go 
defer func
recover success
after recover
```

As we can see, recover caught the panic and the program ended normally. *after panic* in *test_recover()* was not printed, which is correct: when a panic is triggered, control is handed over to the defer. It is just like in Java: when an exception occurs in the `try` block, control is handed over to `catch`, and the code in the catch block is executed next. Meanwhile, *after recover* was printed in *main()*, which means the program has returned to normal and continues to execute until the end.

## Gee's Error Handling Mechanism

For a Web framework, an error handling mechanism is essential. It may be that the framework itself is not thoroughly tested, causing nil pointer exceptions in some situations. Or users may pass incorrect parameters that trigger exceptions, such as an array index out of range or a nil pointer. A system crash caused by such reasons is simply unacceptable.

The framework we implemented in [Day 6](https://geektutu.com/post/gee-day6.html) has no exception handling mechanism; if the code contains a bug that triggers a panic, it can easily crash.

For example, the following code:

```go
func main() {
	r := gee.New()
	r.GET("/panic", func(c *gee.Context) {
		names := []string{"geektutu"}
		c.String(http.StatusOK, names[100])
	})
	r.Run(":9999")
}
```
In the code above, we registered the route `/panic` for gee, and the handler of this route contains an index out of range, `names[100]`. If you visit *localhost:9999/panic*, the web service will crash.

Today, we will add a very simple error handling mechanism to gee: when such an error occurs, return *Internal Server Error* to the user, and print the necessary error information in the log to make troubleshooting easier.

We implemented the middleware mechanism before; error handling can also be a middleware, enhancing the capability of the gee framework.

Add a new file **gee/recovery.go**, in which we implement the `Recovery` middleware.

```go
func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```

The implementation of `Recovery` is very simple: use defer to attach the error recovery function, call *recover()* inside this function to catch the panic, print the stack trace to the log, and return *Internal Server Error* to the user.

You may have noticed that there is a *trace()* function here. This function is used to obtain the stack trace of the panic. The complete code is as follows:

[day7-panic-recover/gee/recovery.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day7-panic-recover)

```go
package gee

import (
	"fmt"
	"log"
	"net/http"
	"runtime"
	"strings"
)

// print stack trace for debug
func trace(message string) string {
	var pcs [32]uintptr
	n := runtime.Callers(3, pcs[:]) // skip first 3 caller

	var str strings.Builder
	str.WriteString(message + "\nTraceback:")
	for _, pc := range pcs[:n] {
		fn := runtime.FuncForPC(pc)
		file, line := fn.FileLine(pc)
		str.WriteString(fmt.Sprintf("\n\t%s:%d", file, line))
	}
	return str.String()
}

func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```

In *trace()*, `runtime.Callers(3, pcs[:])` is called. Callers returns the program counters of the call stack: the 0th caller is Callers itself, the 1st is the layer above, trace, and the 2nd is the layer above that, the `defer func`. Therefore, to keep the log a bit cleaner, we skip the first 3 callers.

Next, `runtime.FuncForPC(pc)` is used to get the corresponding function, and then `fn.FileLine(pc)` is used to get the file name and line number where the function was called, which are printed in the log.

At this point, the error handling mechanism of the gee framework is complete.

## Usage Demo

[day7-panic-recover/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day7-panic-recover)

```go
package main

import (
	"net/http"

	"gee"
)

func main() {
	r := gee.Default()
	r.GET("/", func(c *gee.Context) {
		c.String(http.StatusOK, "Hello Geektutu\n")
	})
	// index out of range for testing Recovery()
	r.GET("/panic", func(c *gee.Context) {
		names := []string{"geektutu"}
		c.String(http.StatusOK, names[100])
	})

	r.Run(":9999")
}
```

Next, let's test: first visit the home page, then visit the buggy `/panic` route — the service still returns a response normally. Then we successfully visit the home page once more, which shows that the service is fully up and running.

```bash
$ curl "http://localhost:9999"
Hello Geektutu
$ curl "http://localhost:9999/panic"
{"message":"Internal Server Error"}
$ curl "http://localhost:9999"
Hello Geektutu
```

In the background log we can see the following content: both the cause of the error and the stack trace are printed. From the log, we can easily tell that an `index out of range` error occurred at *day7-panic-recover/main.go:47*.

```bash
2020/01/09 01:00:10 Route  GET - /
2020/01/09 01:00:10 Route  GET - /panic
2020/01/09 01:00:22 [200] / in 25.364µs
2020/01/09 01:00:32 runtime error: index out of range
Traceback:
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:523
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/panic.go:44
        /tmp/7days-golang/day7-panic-recover/main.go:47
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/recovery.go:37
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/logger.go:15
        /tmp/7days-golang/day7-panic-recover/gee/context.go:41
        /tmp/7days-golang/day7-panic-recover/gee/router.go:99
        /tmp/7days-golang/day7-panic-recover/gee/gee.go:130
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:2775
        /usr/local/Cellar/go/1.12.5/libexec/src/net/http/server.go:1879
        /usr/local/Cellar/go/1.12.5/libexec/src/runtime/asm_amd64.s:1338

2020/01/09 01:00:32 [500] /panic in 395.846µs
2020/01/09 01:00:38 [200] / in 6.985µs
```

## References

- [Package runtime - golang.org](https://golang.org/pkg/runtime/)
- [Is it possible get information about caller function in Golang? - StackOverflow](https://stackoverflow.com/questions/35212985/is-it-possible-get-information-about-caller-function-in-golang)
