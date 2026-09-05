---
title: 7 Days to Implement an RPC Framework GeeRPC in Go from Scratch
description: >-
  A 7-day tutorial on implementing an RPC framework in Go/golang from scratch (7 days implement golang remote procedure call framework from scratch
  tutorial). Build an RPC framework modeled after the implementation of the golang standard library net/rpc, covering the server, an asynchronous and
  concurrent client, message encoding and decoding, service registration, and multiple transport protocols including TCP/Unix/HTTP. On top of that, it adds features such as protocol exchange, a registry,
  service discovery, load balancing, and timeout processing.
date: '2020-10-07 00:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geerpc/geerpc.jpg
lang: en
---

![golang RPC framework](geerpc/geerpc.jpg)

## 1 A Look at RPC Frameworks

RPC (Remote Procedure Call) is a computer communication protocol that allows a program to invoke procedures in a different process space. The RPC client and server may run on the same machine or on different machines. To the programmer, using RPC feels just like calling a local program, with no need to care about the internal implementation details.

There are many ways for applications to communicate with each other, such as the Restful API based on the HTTP protocol, which is widely used between browsers and servers. Compared with RPC, Restful APIs have relatively unified standards, making them more general-purpose and compatible, and they support different languages. The HTTP protocol is text-based and generally offers better readability. But its drawbacks are obvious:

- Restful interfaces require extra definitions; both the client and the server need additional code to handle them, whereas RPC calls are closer to direct invocation.
- Restful messages based on the HTTP protocol are redundant and carry too much useless information, while RPC typically uses a custom protocol format to reduce redundant traffic.
- RPC can adopt more efficient serialization protocols, converting text to binary for transmission to achieve higher performance.
- Because of RPC's flexibility, it is easier to extend and integrate features such as a registry and load balancing.

## 2 What Problems Does an RPC Framework Solve

What problems does an RPC framework need to solve? Or, put another way, why do we need an RPC framework?

Imagine two applications on two machines that need to communicate. First, which transport protocol should be used? If the two applications run on different machines, TCP or HTTP is usually chosen; if they run on the same machine, Unix Socket is also an option. Once the transport protocol is settled, the message encoding format must be decided as well, for example the most common JSON or XML; if messages are large, other encoding schemes like protobuf may be chosen, and the encoded message may even be compressed. The receiving side performs the reverse process: decompress first, then decode.

With the transport protocol and message encoding solved, a series of usability issues still need to be addressed. For example, what happens when a connection times out? Are asynchronous requests and concurrency supported?

If there are many server instances and the client does not care about their addresses or deployment locations, only whether it can get the expected result, that leads to the problems of a registry and load balancing. Simply put, the client and the server are unaware of each other's existence: when the server starts, it registers itself with the registry; when the client makes a call, it fetches all available instances from the registry and picks one to call. That way, the server and the client only need to be aware of the registry. A registry usually also needs to support dynamically adding and removing services, and to use heartbeats to ensure services remain available.

Going further, suppose the servers are provided by different teams. Without a unified RPC framework, each team's service provider would have to implement its own message encoding/decoding, connection pools, send/receive threads, timeout handling, and other repetitive technical work outside the "business logic", resulting in overall inefficiency. Therefore, these shared capabilities "outside the business logic" are exactly what an RPC framework needs to provide.

## 3 About GeeRPC

Go is widely used in cloud computing and microservices, and there is no shortage of mature RPC and microservice frameworks. `grpc`, `rpcx`, and `go-micro` are all very mature frameworks. Generally speaking, RPC is a subset of a microservice framework; a microservice framework can implement the RPC part itself, or it can choose a different RPC framework as the communication foundation.

Considering performance and features, the mature frameworks mentioned above are all quite large in code size, and they are usually deeply coupled with third-party libraries such as `protobuf`, `etcd`, and `zookeeper`, making it hard to see the essence of the framework directly. The goal of GeeRPC is to implement the most important parts of an RPC framework with as little code as possible, to help everyone understand what needs to be considered when designing an RPC framework. Code simplicity comes first, features second.

Therefore, GeeRPC implements the Go standard library `net/rpc` from scratch, and on top of that adds features such as protocol exchange, a registry, service discovery, load balancing, and timeout processing. It is completed in seven days, with about 1000 lines of code in the end.

## 4 Table of Contents

- Day 1 - [Server and Message Encoding](https://geektutu.com/post/geerpc-day1.html) | [Code](ghttps://github.com/geektutu/7days-golang/tree/master/ee-rpc/day1-codec)
- Day 2 - [Asynchronous and Concurrent Client](https://geektutu.com/post/geerpc-day2.html) | [Code](ghttps://github.com/geektutu/7days-golang/tree/master/ee-rpc/day2-client)
- Day 3 - [Service Registration](https://geektutu.com/post/geerpc-day3.html) | [Code](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day3-service )
- Day 4 - [Timeout Processing](https://geektutu.com/post/geerpc-day4.html) | [Code](ghttps://github.com/geektutu/7days-golang/tree/master/ee-rpc/day4-timeout )
- Day 5 - [HTTP Protocol Support](https://geektutu.com/post/geerpc-day5.html) | [Code](ghttps://github.com/geektutu/7days-golang/tree/master/ee-rpc/day5-http-debug)
- Day 6 - [Load Balancing](https://geektutu.com/post/geerpc-day6.html) | [Code](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day6-load-balance)
- Day 7 - [Service Discovery and Registry](https://geektutu.com/post/geerpc-day7.html) | [Code](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day7-registry)

## Appendix: Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [Go Interview Questions](https://geektutu.com/post/qa-golang.html)
