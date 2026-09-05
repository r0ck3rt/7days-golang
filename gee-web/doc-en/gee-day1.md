---
title: Building a Web Framework in Go - Gee Day 1 http.Handler
description: >-
  A tutorial on implementing a Web framework from scratch in Go (7 days implement golang web framework from scratch tutorial), build a Web framework by hand in Go/golang, and design
  a Web framework from scratch modeled after Gin. This article introduces the use of Go's standard library net/http and the http.Handler interface, intercepting all HTTP requests and
  handing them over to the Gee framework.
date: '2019-08-12 08:10:10'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: gee/gee.jpg
lang: en
---

This is the first article of the [7 Days Go Web Framework Tutorial Series from scratch](https://geektutu.com/post/gee.html).

- A brief introduction to the `net/http` library and the `http.Handler` interface.
- Building the prototype of the `Gee` framework, **about 50 lines of code**.

## Starting a Web Service with the Standard Library

Go ships with the `net/http` library, which wraps the fundamental interfaces for HTTP network programming, and the `Gee` Web framework we are going to build is based on `net/http`. Next, let's take a quick look at how to use this library through an example.

**[day1-http-base/base1/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day1-http-base/base1)**

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":9999", nil))
}

// handler echoes r.URL.Path
func indexHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
}

// handler echoes r.URL.Header
func helloHandler(w http.ResponseWriter, req *http.Request) {
	for k, v := range req.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
}
```

We set up two routes, `/` and `/hello`, bound to _indexHandler_ and _helloHandler_ respectively, so different HTTP requests invoke different handlers. Visiting `/` returns the response `URL.Path = /`, while the response of `/hello` is the key-value pair information in the request header.

Let's test it with the curl tool, and we will get the following result.

```bash
$ curl http://localhost:9999/
URL.Path = "/"
$ curl http://localhost:9999/hello
Header["Accept"] = ["*/*"]
Header["User-Agent"] = ["curl/7.54.0"]
```

The last line of the _main_ function is used to start the Web service. The first argument is the address, where `:9999` means listening on port _9999_. The second argument represents the instance that handles all HTTP requests; `nil` means using the instance from the standard library to handle them. This second argument is exactly the entry point for implementing our Web framework on top of the `net/http` standard library.

## Implementing the http.Handler Interface

```go
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```

What is the type of the second argument? By looking into the source code of `net/http`, we can find that `Handler` is an interface which requires implementing the method _ServeHTTP_. In other words, as long as we pass in any instance that implements the _ServerHTTP_ interface, all HTTP requests will be handled by that instance. Let's give it a try right away.

**[day1-http-base/base2/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day1-http-base/base2)**

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

// Engine is the uni handler for all requests
type Engine struct{}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/":
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	case "/hello":
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	default:
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}

func main() {
	engine := new(Engine)
	log.Fatal(http.ListenAndServe(":9999", engine))
}
```

- We define an empty struct `Engine` that implements the method `ServeHTTP`. This method has two arguments: the second argument is _Request_, an object that contains all the information about the HTTP request, such as the request address, Header, and Body; the first argument is _ResponseWriter_, which can be used to construct the response to the request.

- In the _main_ function, we pass the `engine` instance we just created as the second argument to the _ListenAndServe_ method. At this point, we have taken the first step toward implementing a Web framework, that is, redirecting all HTTP requests to our own handling logic. Remember? Before implementing `Engine`, we called _http.HandleFunc_ to build the mapping between routes and handlers, which means we could only write handling logic for specific routes, such as `/hello`. But after implementing `Engine`, we intercept all HTTP requests and have a unified control entry point. Here we can freely define the rules of route mapping, and we can also add some unified processing logic, such as logging and exception handling.

- The result of running the code is the same as before.

## The Prototype of the Gee Framework

Next, we will reorganize the code above and build the prototype of the entire framework.

The final code directory structure looks like this.

```bash
gee/
  |--gee.go
  |--go.mod
main.go
go.mod
```

### go.mod

**[day1-http-base/base3/go.mod](https://github.com/geektutu/7days-golang/tree/master/gee-web/day1-http-base/base3)**

```bash
module example

go 1.13

require gee v0.0.0

replace gee => ./gee
```

- In `go.mod`, use `replace` to point gee to `./gee`

> Since go 1.11, referencing packages with relative paths requires the approach above.

### main.go

**[day1-http-base/base3/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day1-http-base/base3)**

```go
package main

import (
	"fmt"
	"net/http"

	"gee"
)

func main() {
	r := gee.New()
	r.GET("/", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})

	r.GET("/hello", func(w http.ResponseWriter, req *http.Request) {
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	})

	r.Run(":9999")
}
```

Seeing this, if you have used the `gin` framework, you will surely find it extremely familiar. The design and API of the `gee` framework are both modeled after `gin`. Use `New()` to create a gee instance, use the `GET()` method to add routes, and finally use `Run()` to start the Web service. The routes here are only static routes; dynamic routes like `/hello/:name` are not supported yet — we will implement dynamic routing next time.

### gee.go

**[day1-http-base/base3/gee/gee.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day1-http-base/base3)**

```go
package gee

import (
	"fmt"
	"net/http"
)

// HandlerFunc defines the request handler used by gee
type HandlerFunc func(http.ResponseWriter, *http.Request)

// Engine implement the interface of ServeHTTP
type Engine struct {
	router map[string]HandlerFunc
}

// New is the constructor of gee.Engine
func New() *Engine {
	return &Engine{router: make(map[string]HandlerFunc)}
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handler
}

// GET defines the method to add GET request
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
func (engine *Engine) POST(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

// Run defines the method to start a http server
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {
		handler(w, req)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}
```

Now, `gee.go` is the highlight. Let's focus on the implementation of this part.

- First, we define the type `HandlerFunc`, which is provided for framework users to define the handler methods for route mapping. In `Engine`, we add a route mapping table `router`, whose key consists of the request method and the static route path, for example `GET-/`, `GET-/hello`, and `POST-/hello`. In this way, for the same route, if the request methods differ, different handlers can be mapped; the value is the handler mapped by the user.

- When the user calls the `(*Engine).GET()` method, the route and the handler are registered into the mapping table _router_. The `(*Engine).Run()` method is a wrapper around _ListenAndServe_.

- The _ServeHTTP_ method implemented by `Engine` works as follows: parse the request path, look it up in the route mapping table, and if found, execute the registered handler. If not found, return _404 NOT FOUND_.

Run `go run main.go`, then access it with the _curl_ tool, and the result is the same as at the very beginning.

```bash
$ curl http://localhost:9999/
URL.Path = "/"
$ curl http://localhost:9999/hello
Header["Accept"] = ["*/*"]
Header["User-Agent"] = ["curl/7.54.0"]
curl http://localhost:9999/world
404 NOT FOUND: /world
```

At this point, the prototype of the entire `Gee` framework has taken shape. We implemented the route mapping table, provided methods for users to register static routes, and wrapped the function for starting the service. Of course, so far we have not implemented anything more powerful than the `net/http` standard library. Don't worry — dynamic routing, middleware, and other features will be added soon.
