---
title: Building a Web Framework in Go from Scratch - Gee Tutorial
description: >-
  A tutorial on implementing a Web framework from scratch in Go (7 days implement golang web framework from scratch tutorial), build a Web framework by hand in Go/golang, and design
  a Web framework from scratch modeled after Gin.
date: '2019-08-11 10:10:10'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: gee/gee.jpg
lang: en
---

![gee](gee/gee.jpg)

## Designing a Framework

Most of the time, when we need to build a Web application, our first reaction is to think about which framework to use. Frameworks differ greatly in their design philosophy and the features they provide. Take Python's `django` and `flask` for example: the former is large and all-inclusive, while the latter is small and elegant. Go is no different — new frameworks keep emerging, such as `Beego`, `Gin`, `Iris`, and so on. So why must we use a framework instead of the standard library directly? Before designing a framework, we need to answer the question of what core problems a framework solves for us. Only by understanding this can we figure out what features we need to implement in the framework.

Let's first look at how the standard library `net/http` handles a request.

```go
func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/count", counter)
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```

`net/http` provides the basic Web capabilities, namely listening on a port, mapping static routes, and parsing HTTP messages. However, some simple requirements in Web development are not supported and need to be implemented manually.

- Dynamic routing: rules such as `hello/:name` and `hello/*`.
- Authentication: there is no grouping/unified authentication capability; it has to be implemented in the handler of every route mapping.
- Templates: there is no unified, simplified HTML mechanism.
- ...

When we step away from a framework and use the underlying library, the places that require frequent manual handling are exactly where the value of a framework lies. But not every frequently handled place is suitable to be handled inside a framework. Python has a very famous Web framework called [`bottle`](https://github.com/bottlepy/bottle); the entire framework consists of a single file, `bottle.py`, totaling 4400 lines — it can be called a micro-framework. Understanding the features provided by this micro-framework can help us understand the core capabilities of a framework.

- Routing: mapping requests to functions, with support for dynamic routing. For example `'/hello/:name`.
- Templates: provides a template rendering mechanism using the built-in template engine.
- Utilites: provides mechanisms for handling cookies, headers, and so on.
- Plugin: Bottle itself has limited functionality, but it provides a plugin mechanism. Plugins can be installed globally, or take effect for only a few routes.
- ...

## The Gee Framework

In this tutorial, we will implement a simple Web framework in Go, named `Gee` — the first three letters of [`geektutu.com`](https://geektutu.com). The first Go Web framework I came across was `Gin`. `Gin`'s code totals 14K lines, of which 9K are tests, which means the actual amount of code is only **5K**. `Gin` is also a framework I really like; it is very much like `Flask` in Python — small and beautiful.

Many designs in this tutorial, `Implement the Gee Framework in 7 Days`, including the source code, are modeled after `Gin`, and you will see many traces of `Gin`.

Due to time constraints, and to keep things as concise and clear as possible, many parts of this framework implement very simple functionality, but they embody the core design principles of a framework as much as possible. For example, the `Router` design: although the dynamic routing rules it supports are limited, the matching algorithm is implemented with a `Trie tree` for the sake of performance — performance is one of the most important metrics for a `Router`.

I hope this tutorial can inspire you. If you have any good suggestions for Gee, feel free to open [issues - Github](https://github.com/geektutu/7days-golang/issues) and PRs. If you find any problems in the tutorial, you can leave a comment at the end of the article directly.

## Contents

- Day 1: [Prerequisites (http.Handler Interface)](https://geektutu.com/post/gee-day1.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day1-http-base)
- Day 2: [Designing Context (Context)](https://geektutu.com/post/gee-day2.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day2-context)
- Day 3: [Trie Tree Routing (Router)](https://geektutu.com/post/gee-day3.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day3-router)
- Day 4: [Group Control (Group)](https://geektutu.com/post/gee-day4.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day4-group)
- Day 5: [Middleware (Middleware)](https://geektutu.com/post/gee-day5.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day5-middleware)
- Day 6: [HTML Templates (Template)](https://geektutu.com/post/gee-day6.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day6-template)
- Day 7: [Panic Recovery (Panic Recover)](https://geektutu.com/post/gee-day7.html), [Code - Github](https://github.com/geektutu/7days-golang/tree/master/gee-web/day7-panic-recover)

## Recommended Reading

- [A Concise Tutorial for Go](https://geektutu.com/post/quick-golang.html)
- [A Concise Tutorial on Go Unit Testing](https://geektutu.com/post/quick-golang.html)
- [A Concise Tutorial for Gin in Go](https://geektutu.com/post/quick-go-gin.html)
