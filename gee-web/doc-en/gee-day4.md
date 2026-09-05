---
title: Building a Web Framework in Go - Gee Day 4: Group
description: >-
  A 7-day tutorial on implementing a Web framework from scratch in Go (7 days implement golang web framework from
  scratch tutorial). Build a Web framework from scratch with Go/golang, with Gin as the design prototype. This
  article explains the significance of group control (Group Control) and the implementation of nested route groups.
date: '2019-09-01 23:10:10'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: gee-day4/group.jpg
lang: en
---

This is the fourth part of the [7 Days Go Web Framework Tutorial Series from scratch](https://geektutu.com/post/gee.html).

- Implement route group control (Route Group Control), **in about 50 lines of code**

## The Purpose of Grouping

Group control is one of the fundamental features that a Web framework should provide. Here, grouping refers to the grouping of routes. Without route grouping, we would have to control every route individually. But in real business scenarios, a set of routes often requires similar handling. For example:

- Routes starting with `/post` can be accessed anonymously.
- Routes starting with `/admin` require authentication.
- Routes starting with `/api` are RESTful APIs intended for third-party platforms, and require third-party authentication.

In most cases, route groups are distinguished by a common prefix. Therefore, the group control we implement today is also based on prefixes, and supports nested groups. For example, `/post` is a group, and `/post/a` and `/post/b` can be sub-groups of it. Middleware applied to the `/post` group also applies to its sub-groups, and each sub-group can additionally apply its own specific middleware.

Middleware provides the framework with unlimited extensibility. Applying it to groups makes group control far more valuable than simply sharing a common route prefix. For example, the `/admin` group can apply an authentication middleware, while the `/` group applies a logging middleware. Since `/` is the default top-level group, this effectively adds logging capability to all routes, that is, to the entire framework.

We will introduce middleware, the feature that provides this extensibility, in the next section.

## Nested Groups

What attributes does a Group object need? First of all, a prefix, such as `/` or `/api`. To support nested groups, it needs to know the parent of the current group. And of course, as analyzed at the very beginning, middleware is applied to groups, so we also need to store the middleware applied to this group. Remember that we previously called the function `(*Engine).addRoute()` to map all routing rules to Handlers. If a Group object needs to map routing rules directly — for example, we want to be able to call it like this when using the framework:

```go
r := gee.New()
v1 := r.Group("/v1")
v1.GET("/", func(c *gee.Context) {
	c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
})
```

Then the Group object also needs access to the `Router`. For convenience, we can store a pointer to the `Engine` in the Group. All resources of the framework are coordinated by the `Engine`, so all kinds of interfaces can be accessed indirectly through the `Engine`.

So, the final definition of Group looks like this:

**[day4-group/gee/gee.go](https://github.com/geektutu/7days-golang/tree/master/gee-web/day4-group)**

```go
RouterGroup struct {
	prefix      string
	middlewares []HandlerFunc // support middleware
	parent      *RouterGroup  // support nesting
	engine      *Engine       // all groups share a Engine instance
}
```

We can abstract one step further, treating the `Engine` as the top-level group, which means the `Engine` has all the capabilities of `RouterGroup`.

```go
Engine struct {
	*RouterGroup
	router *router
	groups []*RouterGroup // store all groups
}
```

Then we can delegate all routing-related functions to `RouterGroup`.

```go
// New is the constructor of gee.Engine
func New() *Engine {
	engine := &Engine{router: newRouter()}
	engine.RouterGroup = &RouterGroup{engine: engine}
	engine.groups = []*RouterGroup{engine.RouterGroup}
	return engine
}

// Group is defined to create a new RouterGroup
// remember all groups share the same Engine instance
func (group *RouterGroup) Group(prefix string) *RouterGroup {
	engine := group.engine
	newGroup := &RouterGroup{
		prefix: group.prefix + prefix,
		parent: group,
		engine: engine,
	}
	engine.groups = append(engine.groups, newGroup)
	return newGroup
}

func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
	pattern := group.prefix + comp
	log.Printf("Route %4s - %s", method, pattern)
	group.engine.router.addRoute(method, pattern, handler)
}

// GET defines the method to add GET request
func (group *RouterGroup) GET(pattern string, handler HandlerFunc) {
	group.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
func (group *RouterGroup) POST(pattern string, handler HandlerFunc) {
	group.addRoute("POST", pattern, handler)
}
```

Take a closer look at the `addRoute` function: it calls `group.engine.router.addRoute` to map routes. Since the `Engine` inherits all the properties and methods of `RouterGroup` in a sense — its `engine` field points to itself — with this implementation we can add routes both the way we did before and through groups.

## Usage Demo

The demo for testing the framework can now be written like this:

```go
func main() {
	r := gee.New()
	r.GET("/index", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Index Page</h1>")
	})
	v1 := r.Group("/v1")
	{
		v1.GET("/", func(c *gee.Context) {
			c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
		})

		v1.GET("/hello", func(c *gee.Context) {
			// expect /hello?name=geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
		})
	}
	v2 := r.Group("/v2")
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
		v2.POST("/login", func(c *gee.Context) {
			c.JSON(http.StatusOK, gee.H{
				"username": c.PostForm("username"),
				"password": c.PostForm("password"),
			})
		})

	}

	r.Run(":9999")
}
```

A quick test with curl:

```bash
$ curl "http://localhost:9999/v1/hello?name=geektutu"
hello geektutu, you're at /v1/hello

$ curl "http://localhost:9999/v2/hello/geektutu"
hello geektutu, you're at /hello/geektutu
```
