---
title: Implement a Distributed Cache in Go - GeeCache Day 6: Preventing Cache Breakdown
description: >-
  A tutorial on implementing the distributed cache GeeCache from scratch in Go (7 days implement golang distributed cache from scratch tutorial), building a distributed cache modeled after
  groupcache. This article introduces the concepts of cache avalanche, cache breakdown and cache penetration, and implements and tests singleflight to prevent cache breakdown.
date: '2020-02-17 07:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geecache-day6/singleflight_logo.jpg
lang: en
---

![geecache single flight](geecache-day6/singleflight.jpg)

This article is part 6 of the [7 Days Go Distributed Cache Tutorial Series from scratch](https://geektutu.com/post/geecache.html).

- A brief introduction to cache avalanche, cache breakdown and cache penetration.
- Using singleflight to prevent cache breakdown, with implementation and testing. **about 70 lines of code**

## 1 Cache Avalanche, Cache Breakdown and Cache Penetration

[GeeCache Day 5](https://geektutu.com/post/geecache-day5.html) mentioned cache avalanches and cache breakdowns; here is a summary:

> **Cache avalanche**: all cached entries expire at the same moment, causing a sudden surge of DB requests and a sharp increase in pressure, which triggers an avalanche. Cache avalanches are usually caused by a cache server going down, by cached keys being assigned the same expiration time, and so on.

> **Cache breakdown**: an existing key receives a large number of simultaneous requests at the moment the cache expires, and these requests all break through to the DB, causing a sudden surge of DB requests and a sharp increase in pressure.

> **Cache penetration**: querying data that does not exist. Because the data does not exist, it is never written to the cache, so every such request goes to the DB; if the instantaneous traffic is large enough, it penetrates to the DB and causes it to crash.

## 2 Implementing singleflight

Remember the test results at the end of [GeeCache Day 5](https://geektutu.com/post/geecache-day5.html)?

```bash
2020/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
2020/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
2020/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
```

We sent N concurrent requests `?key=Tom`, and node 8003 sent N simultaneous requests to 8001. If access to the database is not restricted in any way, N requests will very likely hit the database as well, easily leading to cache breakdown and penetration. Even with safeguards for the database in place, HTTP requests are very resource-intensive operations, and it is unnecessary for node 8003 to send three requests to 8001 for the same key. So in this situation, how can we send only one request to the remote node?

geecache implements a package named singleflight to solve this problem.

[day6-single-flight/geecache/singleflight/singleflight.go - github](https://github.com/geektutu/7days-golang/tree/master/gee-cache/day6-single-flight/geecache/singleflight)

First, create the `call` and `Group` types.

```go
package singleflight

import "sync"

type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}

type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call
}
```

- `call` represents a request that is in progress, or has already finished. A `sync.WaitGroup` lock is used to avoid re-entry.
- `Group` is the main data structure of singleflight, managing the requests (calls) for different keys.

Implement the `Do` method

```go
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

- The Do method takes 2 parameters: the first is `key` and the second is a function `fn`. What Do does is ensure that, for the same key, no matter how many times Do is called, the function `fn` is only called once; once the fn call finishes, it returns the return value or error.

`g.mu` is a lock added to protect the Group's field `m` from concurrent reads and writes. To make the `Do` function easier to understand, let's temporarily remove `g.mu`, and also remove the part that lazily initializes `g.m` — the purpose of lazy initialization is simply to use memory more efficiently.

The remaining logic is then quite clear:

```go
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	if c, ok := g.m[key]; ok {
		c.wg.Wait()   // if a request is in progress, wait for it
		return c.val, c.err  // the request has finished, return the result
	}
	c := new(call)
	c.wg.Add(1)       // increment the counter before starting the request
	g.m[key] = c      // add to g.m, indicating that a request for key is already being processed

	c.val, c.err = fn() // call fn, start the request
	c.wg.Done()         // the request has finished

    delete(g.m, key)    // update g.m
    
	return c.val, c.err // return the result
}
```

There is no message passing between the concurrent goroutines, which makes `sync.WaitGroup` a perfect fit.

- wg.Add(1) increments the counter by 1.
- wg.Wait() blocks until the counter is released.
- wg.Done() decrements the counter by 1.

## 3 Using singleflight

[day6-single-flight/geecache/geecache.go - github](https://github.com/geektutu/7days-golang/tree/master/gee-cache/day6-single-flight/geecache)

```go
type Group struct {
	name      string
	getter    Getter
	mainCache cache
	peers     PeerPicker
	// use singleflight.Group to make sure that
	// each key is only fetched once
	loader *singleflight.Group
}

func NewGroup(name string, cacheBytes int64, getter Getter) *Group {
    // ...
	g := &Group{
        // ...
		loader:    &singleflight.Group{},
	}
	return g
}

func (g *Group) load(key string) (value ByteView, err error) {
	// each key is only fetched once (either locally or remotely)
	// regardless of the number of concurrent callers.
	viewi, err := g.loader.Do(key, func() (interface{}, error) {
		if g.peers != nil {
			if peer, ok := g.peers.PickPeer(key); ok {
				if value, err = g.getFromPeer(peer, key); err == nil {
					return value, nil
				}
				log.Println("[GeeCache] Failed to get from peer", err)
			}
		}

		return g.getLocally(key)
	})

	if err == nil {
		return viewi.(ByteView), nil
	}
	return
}
```

- Modify `Group` in `geecache.go` to add the field loader, and update the constructor `NewGroup`.
- Modify the `load` function by simply wrapping the original load logic with `g.loader.Do`; this ensures that, in concurrent scenarios, the `load` process is called only once for the same key.

## 4 Testing

Run `run.sh` and you can see the effect.

```bash
$ ./run.sh
2020/02/16 22:36:00 [Server http://localhost:8003] Pick peer http://localhost:8001
2020/02/16 22:36:00 [Server http://localhost:8001] GET /_geecache/scores/Tom
2020/02/16 22:36:00 [SlowDB] search key Tom
630630630
```

As you can see, three concurrent requests were made to the API, but 8003 sent only one request to 8001, and that was enough.

If the concurrency is not high enough, you may still see three requests being made to 8001. In that case, the three requests are executed serially and the `singleflight` lock mechanism is never triggered; increase the number of concurrent requests and test again. That is, copy the `curl` command in `run.sh` N times.

## Recommended

- [A Quick Go Tutorial # Concurrent Programming](https://geektutu.com/post/quick-golang.html#7-%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B-goroutine)
- [A Quick Guide to Go Unit Testing](https://geektutu.com/post/quick-go-test.html)
