---
title: Building a Web Framework in Go - Gee Day 2: Context
description: >-
  A tutorial on implementing a Web framework from scratch in Go (7 days implement golang web framework from scratch tutorial), build a Web framework by hand in Go/golang, and design
  a Web framework from scratch modeled after Gin. This article introduces the design idea of the request context (Context), which wraps methods for returning JSON/String/Data/HTML and other types of responses.
date: '2019-08-19 08:10:10'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: gee/gee.jpg
lang: en
---

This is the second article of the [7 Days Go Web Framework Tutorial Series from scratch](https://geektutu.com/post/gee.html).

- Extract `routing (router)` into a separate module to make it easier to enhance later.
- Design `Context` to wrap Request and Response, providing support for return types such as JSON and HTML.
- Day 2 of building the Gee framework by hand, **140 lines of framework code, about 90 lines added**

## Usage

To show what we have accomplished on Day 2, let's take a look at what it looks like when used.

[day2-context/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day2-context)


```go
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})
	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})

	r.POST("/login", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{
			"username": c.PostForm("username"),
			"password": c.PostForm("password"),
		})
	})

	r.Run(":9999")
}
```

- The parameter of `Handler` has become `gee.Context`, which provides the ability to query Query/PostForm parameters.
- `gee.Context` wraps the `HTML/String/JSON` functions, enabling quick construction of HTTP responses.

## Designing Context

### Motivation

1. For a Web service, everything boils down to constructing a response `http.ResponseWriter` from a request `*http.Request`. However, the interfaces provided by these two objects are too fine-grained. For example, to construct a complete response, we need to consider the message header (Header) and the message body (Body), while the Header contains the status code (StatusCode), the content type (ContentType), and other information that almost every request needs to set. Therefore, without effective wrapping, framework users would have to write a lot of repetitive, cumbersome code, and it would be error-prone. For common scenarios, being able to construct HTTP responses efficiently is something a good framework must take into account.

Let's use returning JSON data as a comparison to get a feel for the difference before and after wrapping.

Before wrapping

```go
obj = map[string]interface{}{
    "name": "geektutu",
    "password": "1234",
}
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusOK)
encoder := json.NewEncoder(w)
if err := encoder.Encode(obj); err != nil {
    http.Error(w, err.Error(), 500)
}
```

VS after wrapping:

```go
c.JSON(http.StatusOK, gee.H{
    "username": c.PostForm("username"),
    "password": c.PostForm("password"),
})
```


2. Wrapping the methods of `*http.Request` and `http.ResponseWriter` for the intended use cases, simplifying the calls to the related interfaces, is only one of the reasons for designing Context. For a framework, additional features also need to be supported. For example, when we parse the dynamic route `/hello/:name` in the future, where should the value of the parameter `:name` be stored? Or, if the framework needs to support middleware, where should the information produced by middleware be stored? Context is created with the arrival of each request and destroyed when the request ends, so all information strongly related to the current request should be carried by Context. Therefore, in the design of the Context structure, extensibility and complexity are kept inside, while the external interface is simplified. The handlers of routes, as well as the middleware to be implemented, all uniformly take a Context instance as their parameter — Context is like a treasure chest of a single session, where you can find anything.

### Implementation

[day2-context/gee/context.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day2-context)

```go
type H map[string]interface{}

type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	// response info
	StatusCode int
}

func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Writer: w,
		Req:    req,
		Path:   req.URL.Path,
		Method: req.Method,
	}
}

func (c *Context) PostForm(key string) string {
	return c.Req.FormValue(key)
}

func (c *Context) Query(key string) string {
	return c.Req.URL.Query().Get(key)
}

func (c *Context) Status(code int) {
	c.StatusCode = code
	c.Writer.WriteHeader(code)
}

func (c *Context) SetHeader(key string, value string) {
	c.Writer.Header().Set(key, value)
}

func (c *Context) String(code int, format string, values ...interface{}) {
	c.SetHeader("Content-Type", "text/plain")
	c.Status(code)
	c.Writer.Write([]byte(fmt.Sprintf(format, values...)))
}

func (c *Context) JSON(code int, obj interface{}) {
	c.SetHeader("Content-Type", "application/json")
	c.Status(code)
	encoder := json.NewEncoder(c.Writer)
	if err := encoder.Encode(obj); err != nil {
		http.Error(c.Writer, err.Error(), 500)
	}
}

func (c *Context) Data(code int, data []byte) {
	c.Status(code)
	c.Writer.Write(data)
}

func (c *Context) HTML(code int, html string) {
	c.SetHeader("Content-Type", "text/html")
	c.Status(code)
	c.Writer.Write([]byte(html))
}
```

- At the very beginning of the code, we give `map[string]interface{}` an alias `gee.H`, which makes constructing JSON data look more concise.
- For now, `Context` only contains `http.ResponseWriter` and `*http.Request`, and additionally provides direct access to the two frequently used properties, Method and Path.
- Provides methods to access Query and PostForm parameters.
- Provides methods to quickly construct String/Data/JSON/HTML responses.

## Router

We extracted the routing-related methods and structures and put them into a new file, `router.go`, which makes it convenient to enhance the router's functionality in the next part, such as providing support for dynamic routing. The router's handle method has undergone a subtle adjustment: the parameter of the handler has become a Context.

[day2-context/gee/router.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day2-context)

```go
type router struct {
	handlers map[string]HandlerFunc
}

func newRouter() *router {
	return &router{handlers: make(map[string]HandlerFunc)}
}

func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	log.Printf("Route %4s - %s", method, pattern)
	key := method + "-" + pattern
	r.handlers[key] = handler
}

func (r *router) handle(c *Context) {
	key := c.Method + "-" + c.Path
	if handler, ok := r.handlers[key]; ok {
		handler(c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
```

## Framework Entry Point

[day2-context/gee/gee.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day2-context)

```go
// HandlerFunc defines the request handler used by gee
type HandlerFunc func(*Context)

// Engine implement the interface of ServeHTTP
type Engine struct {
	router *router
}

// New is the constructor of gee.Engine
func New() *Engine {
	return &Engine{router: newRouter()}
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	engine.router.addRoute(method, pattern, handler)
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
	c := newContext(w, req)
	engine.router.handle(c)
}
```

With the `router`-related code extracted, `gee.go` has become much simpler. The most important part is still taking over all HTTP requests by implementing the ServeHTTP interface. Compared with the Day 1 code, this method also has a subtle adjustment: a Context object is constructed before calling router.handle. This object is still very simple for now, merely wrapping the original two arguments; later, we will gradually give Context wings.

As for how to use it, `main.go` already made its appearance at the beginning. Run `go run main.go`, and with the help of curl, let's take a look at today's results together.

```bash
$ curl -i http://localhost:9999/
HTTP/1.1 200 OK
Date: Mon, 12 Aug 2019 16:52:52 GMT
Content-Length: 18
Content-Type: text/html; charset=utf-8
<h1>Hello Gee</h1>

$ curl "http://localhost:9999/hello?name=geektutu"
hello geektutu, you're at /hello

$ curl "http://localhost:9999/login" -X POST -d 'username=geektutu&password=1234'
{"password":"1234","username":"geektutu"}

$ curl "http://localhost:9999/xxx"
404 NOT FOUND: /xxx
```
