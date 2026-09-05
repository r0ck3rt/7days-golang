---
title: Building a Web Framework in Go - Gee Day 5: Middleware
description: >-
  A 7-day tutorial on implementing a Web framework from scratch in Go (7 days implement golang web framework from
  scratch tutorial). Build a Web framework from scratch with Go/golang, with Gin as the design prototype. This
  article explains how to add middleware support (middlewares) to the Web framework.
date: '2019-09-02 04:10:10'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: gee-day5/middleware.jpg
lang: en
---

This is the fifth part of the [7 Days Go Web Framework Tutorial Series from scratch](https://geektutu.com/post/gee.html).

- Design and implement the middleware (Middlewares) mechanism of the Web framework.
- Implement a general-purpose `Logger` middleware that can record the time taken from request to response, **in about 50 lines of code**

## What Is Middleware

Middleware, simply put, is a technical, non-business component. A Web framework cannot possibly understand all businesses, and therefore cannot implement all features. So the framework needs a plug-in point that allows users to define their own functionality and embed it into the framework, as if it were natively supported by the framework. For middleware, two key points need to be considered:

- Where is the insertion point? Users of the framework don't care about the details of the underlying logic. If the insertion point is too low-level, the middleware logic becomes very complicated. If the insertion point is too close to the user, then there is not much advantage over the user simply defining a set of functions and manually calling them in each Handler.
- What is the input of the middleware? The input determines the extensibility. If too few parameters are exposed, users have limited room to work with.

So what should middleware look like for a Web framework? The implementation that follows is largely modeled after the Gin framework.

## Middleware Design

The definition of Gee's middleware is the same as the Handler used for route mapping; its input is the `Context` object. The insertion point is after the framework receives a request and initializes the `Context` object, allowing users to use their own middleware to do some extra processing, such as logging, and to further process the `Context`. In addition, by calling the `(*Context).Next()` function, middleware can wait until the user-defined `Handler` has finished processing, and then do some additional operations, such as calculating how long the processing took. In other words, Gee's middleware supports performing extra operations both before and after a request is processed. For example, we want to eventually be able to support middleware defined as follows, where `c.Next()` means waiting for the other middleware or the user's `Handler` to execute:

****[day4-group/gee/logger.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day5-middleware)****

```go
func Logger() HandlerFunc {
	return func(c *Context) {
		// Start timer
		t := time.Now()
		// Process request
		c.Next()
		// Calculate resolution time
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}
```

In addition, it supports setting multiple middleware, which are called in order.

In the previous article, [Group Control](https://geektutu.com/post/gee-day4.html), we mentioned that middleware is applied to the `RouterGroup`. Applied to the top-level group, it takes effect globally, and all requests are processed by the middleware. So why not apply it to each individual route? Applying it to a specific route would be no more intuitive than the user calling it directly in the Handler. A feature that only works on a single route is far too specific and not suitable to be defined as middleware.

Our previous framework design was like this: when a request is received, the route is matched, and all information about the request is stored in the `Context`. Middleware is no exception. After receiving a request, we should look up all middleware that applies to that route, store it in the `Context`, and call it in order. Why, after calling them one by one, do we still need to store them in the `Context`? Because in our design, middleware works not only before the processing flow, but also after it — that is, after the user-defined Handler finishes, the remaining operations can still be executed.

To this end, we added 2 fields to the `Context` and defined the `Next` method:

**[day4-group/gee/context.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day5-middleware)**

```go
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	Params map[string]string
	// response info
	StatusCode int
	// middleware
	handlers []HandlerFunc
	index    int
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Path:   req.URL.Path,
		Method: req.Method,
		Req:    req,
		Writer: w,
		index:  -1,
	}
}

func (c *Context) Next() {
	c.index++
	s := len(c.handlers)
	for ; c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}
```

`index` records which middleware is currently being executed. When the `Next` method is called inside a middleware, control is handed over to the next middleware, until the last middleware is called; then, from back to front, the part that each middleware defined after the `Next` call is executed. What would happen if we added the `Handler` defined by the user when mapping the route to the `c.handlers` list? You have probably guessed it.


```
func A(c *Context) {
    part1
    c.Next()
    part2
}
func B(c *Context) {
    part3
    c.Next()
    part4
}
```

Suppose we have applied middleware A and B, and the Handler mapped to the route. `c.handlers` looks like [A, B, Handler], and `c.index` is initialized to -1. When `c.Next()` is called, the flow goes like this:

- c.index++, c.index becomes 0
- 0 < 3, call c.handlers[0], which is A
- Execute part1, call c.Next()
- c.index++, c.index becomes 1
- 1 < 3, call c.handlers[1], which is B
- Execute part3, call c.Next()
- c.index++, c.index becomes 2
- 2 < 3, call c.handlers[2], which is the Handler
- The Handler finishes, return to part4 in B, execute part4
- part4 finishes, return to part2 in A, execute part2
- part2 finishes. Done.

To state the key point in one sentence, the final order is `part1 -> part3 -> Handler -> part 4 -> part2`. This exactly meets our requirements for middleware. Next, looking at the calling code will tie everything together.


## Implementation

- Define the `Use` function to apply middleware to a Group.

**[day4-group/gee/gee.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day5-middleware)**

```go
// Use is defined to add middleware to the group
func (group *RouterGroup) Use(middlewares ...HandlerFunc) {
	group.middlewares = append(group.middlewares, middlewares...)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	var middlewares []HandlerFunc
	for _, group := range engine.groups {
		if strings.HasPrefix(req.URL.Path, group.prefix) {
			middlewares = append(middlewares, group.middlewares...)
		}
	}
	c := newContext(w, req)
	c.handlers = middlewares
	engine.router.handle(c)
}
```

The ServeHTTP function has also changed. When we receive a specific request, we need to determine which middleware applies to it; here we simply use the URL prefix to decide. After obtaining the list of middleware, we assign it to `c.handlers`.

- In the handle function, the Handler obtained from route matching is added to the `c.handlers` list, and then `c.Next()` is called.

**[day4-group/gee/router.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day5-middleware)**

```go
func (r *router) handle(c *Context) {
	n, params := r.getRoute(c.Method, c.Path)

	if n != nil {
		key := c.Method + "-" + n.pattern
		c.Params = params
		c.handlers = append(c.handlers, r.handlers[key])
	} else {
		c.handlers = append(c.handlers, func(c *Context) {
			c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
		})
	}
	c.Next()
}
```

## Usage Demo

```go
func onlyForV2() gee.HandlerFunc {
	return func(c *gee.Context) {
		// Start timer
		t := time.Now()
		// if a server error occurred
		c.Fail(500, "Internal Server Error")
		// Calculate resolution time
		log.Printf("[%d] %s in %v for group v2", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}

func main() {
	r := gee.New()
	r.Use(gee.Logger()) // global midlleware
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	v2 := r.Group("/v2")
	v2.Use(onlyForV2()) // v2 group middleware
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
	}

	r.Run(":9999")
}
```

`gee.Logger()` is the middleware we introduced at the very beginning. We put this middleware together with the framework code, as one provided by the framework by default. In this example, we applied `gee.Logger()` globally, so all routes use this middleware. `onlyForV2()` is used to test the feature and is only applied to the Group corresponding to `v2`.

Next, test with curl. As you can see, both middleware take effect in the v2 Group.

```bash
$ curl http://localhost:9999/
>>> log
2019/08/17 01:37:38 [200] / in 3.14µs

(2) global + group middleware
$ curl http://localhost:9999/v2/hello/geektutu
>>> log
2019/08/17 01:38:48 [200] /v2/hello/geektutu in 61.467µs for group v2
2019/08/17 01:38:48 [200] /v2/hello/geektutu in 281µs
```
