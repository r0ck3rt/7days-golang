---
title: Implement a Distributed Cache in Go from Scratch - GeeCache
description: >-
  7 Days Go Distributed Cache Tutorial Series from scratch (7 days implement golang distributed cache from scratch tutorial). Build a distributed cache by hand, modeled after the implementation of
  groupcache. Features include single-node/distributed caching, LRU (Least Recently Used) cache eviction strategy, preventing cache breakdown, consistent hashing (Consistent Hash), protobuf-based communication, and more.
date: '2020-02-08 09:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geecache/geecache_sm.jpg
lang: en
---

![distributed cache geecache](geecache/geecache.jpg)

## 1 A Word on Distributed Caching

The first time a request comes in, the results of some time-consuming operations are temporarily stored; when the same request arrives later, the cached data is returned directly. I think this is how most people understand caching. Caches are everywhere in computer systems. For example, when you visit a web page, the page and the static files it references, such as JS/CSS, are cached either in the local browser or on CDN servers depending on the strategy, so the second visit feels much faster. Take the number of likes on Weibo as another example: it would be impossible for every user's every visit to query all the like records from the database and count them—database operations are time-consuming and can hardly sustain such large traffic. So data like likes is generally cached in a Redis cluster.

> In the business world, cash is king; in the world of architecture, cache is king.

The simplest form of cache is an in-memory key-value cache. When it comes to key-value pairs, the dict type comes to mind easily—called a map in Go. So why not just create a map and insert new data into it each time? Isn't that a key-value cache? What could go wrong with this approach?

1) What if memory runs out?

We could just delete a few entries at random. But is random deletion the right approach, or should we delete in time order? Or is there a better eviction strategy? Different data has different access frequencies, so wouldn't it be better to prioritize evicting the least frequently accessed data? Access frequency may change over time, in which case evicting the least recently used data may be a better choice. We need to implement a reasonable eviction strategy.

2) What about concurrent write conflicts?

Access to a cache is generally never serial. A map has no concurrency protection, so to handle concurrent scenarios, modification operations (including insertions, updates, and deletions) need locking.

3) What if a single machine doesn't have enough performance?

The resources of a single computer are limited—computation, storage, and so on. As business volume and traffic grow, a single machine easily hits a bottleneck. To leverage multiple machines and process requests in parallel for better performance, the cache application must support distribution. This is called scaling horizontally. The opposite is scaling vertically, which improves system performance by increasing the computation, storage, and bandwidth of a single node. Hardware cost and performance are not linearly related, so in most cases, a distributed system is a better choice.

4) ...

## 2 About GeeCache

Designing a distributed cache system requires considering resource control, eviction strategies, concurrency, communication between distributed nodes, and many other issues. Moreover, for different application scenarios, trade-offs must be made between different features—for example, should cache updates be supported? Or is it assumed that a cache entry cannot be changed before it is evicted? Different trade-offs lead to different implementations.

[groupcache](https://github.com/golang/groupcache) is the Go version of memcached, intended to replace memcached in certain specific scenarios. The author of groupcache is also the author of memcached. Whether you want to understand single-node caching or distributed caching, studying the implementation of this library in depth is very meaningful.

`GeeCache` basically imitates the implementation of [groupcache](https://github.com/golang/groupcache). To keep the code volume around 500 lines (groupcache is about 3,000 lines), some features have been trimmed. But overall, the implementation remains very close to groupcache. Supported features include:

- Single-node caching and HTTP-based distributed caching
- Least Recently Used (LRU) cache eviction strategy
- Using Go's locking mechanisms to prevent cache breakdown
- Using consistent hashing to select nodes for load balancing
- Using protobuf for efficient binary communication between nodes
- ...

`GeeCache` is implemented over 7 days. Each day's work can be run and tested independently. Like building blocks, the features implemented each day combine into the final distributed cache system. Each day involves about 100 lines of code.

## 3 Table of Contents

- Day 1: [LRU Cache Eviction Strategy](https://geektutu.com/post/geecache-day1.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day1-lru)
- Day 2: [Single-Node Concurrent Cache](https://geektutu.com/post/geecache-day2.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day2-single-node)
- Day 3: [HTTP Server](https://geektutu.com/post/geecache-day3.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day3-http-server)
- Day 4: [Consistent Hashing (Hash)](https://geektutu.com/post/geecache-day4.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day4-consistent-hash)
- Day 5: [Distributed Nodes](https://geektutu.com/post/geecache-day5.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day5-multi-nodes)
- Day 6: [Preventing Cache Breakdown](https://geektutu.com/post/geecache-day6.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day6-single-flight)
- Day 7: [Communication with Protobuf](https://geektutu.com/post/geecache-day7.html) | [Code - Github](https://github.com/geektutu/7days-golang/blob/master/gee-cache/day7-proto-buf)

## Further Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [A Concise Go Test Unit Testing Tutorial](https://geektutu.com/post/quick-go-test.html)
- [A Concise Go Protobuf Tutorial](https://geektutu.com/post/quick-go-protobuf.html)
