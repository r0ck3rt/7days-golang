---
title: Implement an RPC Framework in Go - GeeRPC Day 6 Load Balancing
description: >-
  A 7-day tutorial on implementing an RPC framework in Go/golang from scratch (7 days implement golang remote procedure call framework from scratch
  tutorial). Build an RPC framework modeled after the implementation of the golang standard library net/rpc, covering the server, an asynchronous and
  concurrent client, message encoding and decoding, service registration, and multiple transport protocols including TCP/Unix/HTTP. On Day 6, we
  implement two simple load balancing (load balance) algorithms: random selection and Round Robin scheduling.
date: '2020-10-08 22:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geerpc/geerpc.jpg
lang: en
---

![golang RPC framework](geerpc/geerpc.jpg)

This is the sixth article in the [7 Days to Implement an RPC Framework GeeRPC in Go from Scratch](https://geektutu.com/post/geerpc.html) series.

- Implement server-side load balancing via random selection and the Round Robin scheduling algorithm, in about 250 lines of code

## Load Balancing Strategies

Suppose there are multiple service instances, each providing the same functionality. To improve the overall throughput of the system, the instances are deployed on different machines. The client can choose any instance to invoke in order to get the desired result. But how should it choose? That depends on the load balancing strategy. For an RPC framework, several strategies come to mind easily:

- Random selection - pick one at random from the list of servers.
- Round Robin - schedule different servers in turn, executing i = (i + 1) mod n on each dispatch.
- Weighted Round Robin - building on the Round Robin algorithm, assign a weight to each service instance, giving higher weights to higher-performance machines; weights can also be adjusted dynamically based on the current load of each instance, for example, by considering the CPU and memory consumption of the servers over the last 5 minutes.
- Hash/consistent hashing - compute a hash value based on certain characteristics of the request, and route the request to the corresponding machine according to the hash. Consistent hashing can also solve the scheduling jitter problem when service instances are added dynamically. A typical application scenario of consistent hashing is distributed caching. If you are interested, read [Implement a Distributed Cache in Go - GeeCache Day 4 Consistent Hashing (Hash)](https://geektutu.com/post/geecache-day4.html)
- ...

## Service Discovery

Load balancing presupposes multiple service instances, so let's first implement the most basic service discovery module, Discovery. To keep it decoupled from the communication part, the code here is placed in the xclient subdirectory.

Define two types:

- SelectMode represents different load balancing strategies; for simplicity, GeeRPC only implements two strategies: Random and RoundRobin.
- Discovery is an interface type that contains the most basic methods needed for service discovery.
    - Refresh() refreshes the list of servers from the registry
    - Update(servers []string) manually updates the list of servers
    - Get(mode SelectMode) selects a service instance according to the load balancing strategy
    - GetAll() returns all service instances

[day6-load-balance/xclient/discovery.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day6-load-balance)

```go
package xclient

import (
	"errors"
	"math"
	"math/rand"
	"sync"
	"time"
)

type SelectMode int

const (
	RandomSelect     SelectMode = iota // select randomly
	RoundRobinSelect                   // select using Robbin algorithm
)

type Discovery interface {
	Refresh() error // refresh from remote registry
	Update(servers []string) error
	Get(mode SelectMode) (string, error)
	GetAll() ([]string, error)
}
```

Next, we implement a discovery struct that does not require a registry, with the list of servers maintained manually: MultiServersDiscovery

```go
// MultiServersDiscovery is a discovery for multi servers without a registry center
// user provides the server addresses explicitly instead
type MultiServersDiscovery struct {
	r       *rand.Rand   // generate random number
	mu      sync.RWMutex // protect following
	servers []string
	index   int // record the selected position for robin algorithm
}

// NewMultiServerDiscovery creates a MultiServersDiscovery instance
func NewMultiServerDiscovery(servers []string) *MultiServersDiscovery {
	d := &MultiServersDiscovery{
		servers: servers,
		r:       rand.New(rand.NewSource(time.Now().UnixNano())),
	}
	d.index = d.r.Intn(math.MaxInt32 - 1)
	return d
}
```

- r is an instance that generates random numbers; its seed is initialized with a timestamp to avoid generating the same sequence of random numbers every time.
- index records the position that the Round Robin algorithm has already polled up to; to avoid always starting from 0, it is initialized with a random value.

Then, implement the Discovery interface

```go
var _ Discovery = (*MultiServersDiscovery)(nil)

// Refresh doesn't make sense for MultiServersDiscovery, so ignore it
func (d *MultiServersDiscovery) Refresh() error {
	return nil
}

// Update the servers of discovery dynamically if needed
func (d *MultiServersDiscovery) Update(servers []string) error {
	d.mu.Lock()
	defer d.mu.Unlock()
	d.servers = servers
	return nil
}

// Get a server according to mode
func (d *MultiServersDiscovery) Get(mode SelectMode) (string, error) {
	d.mu.Lock()
	defer d.mu.Unlock()
	n := len(d.servers)
	if n == 0 {
		return "", errors.New("rpc discovery: no available servers")
	}
	switch mode {
	case RandomSelect:
		return d.servers[d.r.Intn(n)], nil
	case RoundRobinSelect:
		s := d.servers[d.index%n] // servers could be updated, so mode n to ensure safety
		d.index = (d.index + 1) % n
		return s, nil
	default:
		return "", errors.New("rpc discovery: not supported select mode")
	}
}

// returns all servers in discovery
func (d *MultiServersDiscovery) GetAll() ([]string, error) {
	d.mu.RLock()
	defer d.mu.RUnlock()
	// return a copy of d.servers
	servers := make([]string, len(d.servers), len(d.servers))
	copy(servers, d.servers)
	return servers, nil
}
```

## Client with Load Balancing Support

Next, we expose a client with load balancing support, XClient, to users.

[day6-load-balance/xclient/xclient.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day6-load-balance)

```go
package xclient

import (
	"context"
	. "geerpc"
	"io"
	"reflect"
	"sync"
)

type XClient struct {
	d       Discovery
	mode    SelectMode
	opt     *Option
	mu      sync.Mutex // protect following
	clients map[string]*Client
}

var _ io.Closer = (*XClient)(nil)

func NewXClient(d Discovery, mode SelectMode, opt *Option) *XClient {
	return &XClient{d: d, mode: mode, opt: opt, clients: make(map[string]*Client)}
}

func (xc *XClient) Close() error {
	xc.mu.Lock()
	defer xc.mu.Unlock()
	for key, client := range xc.clients {
		// I have no idea how to deal with error, just ignore it.
		_ = client.Close()
		delete(xc.clients, key)
	}
	return nil
}
```

The constructor of XClient takes three arguments: a service discovery instance Discovery, the load balancing mode SelectMode, and the protocol options Option. To reuse already-established Socket connections as much as possible, clients holds the successfully created Client instances, and a Close method is provided to close the established connections when finished.

Next, implement the most basic client capability, `Call`.

```go
func (xc *XClient) dial(rpcAddr string) (*Client, error) {
	xc.mu.Lock()
	defer xc.mu.Unlock()
	client, ok := xc.clients[rpcAddr]
	if ok && !client.IsAvailable() {
		_ = client.Close()
		delete(xc.clients, rpcAddr)
		client = nil
	}
	if client == nil {
		var err error
		client, err = XDial(rpcAddr, xc.opt)
		if err != nil {
			return nil, err
		}
		xc.clients[rpcAddr] = client
	}
	return client, nil
}

func (xc *XClient) call(rpcAddr string, ctx context.Context, serviceMethod string, args, reply interface{}) error {
	client, err := xc.dial(rpcAddr)
	if err != nil {
		return err
	}
	return client.Call(ctx, serviceMethod, args, reply)
}

// Call invokes the named function, waits for it to complete,
// and returns its error status.
// xc will choose a proper server.
func (xc *XClient) Call(ctx context.Context, serviceMethod string, args, reply interface{}) error {
	rpcAddr, err := xc.d.Get(xc.mode)
	if err != nil {
		return err
	}
	return xc.call(rpcAddr, ctx, serviceMethod, args, reply)
}
```

We encapsulate the ability to reuse Clients in the `dial` method. The logic of dial is as follows:

1) Check whether `xc.clients` has a cached Client. If so, check whether it is in an available state: if it is, return the cached Client; if not, remove it from the cache.
2) If step 1) did not return a cached Client, a new Client needs to be created; cache it and return it.

In addition, we add a commonly used feature to XClient: `Broadcast`.

```go
// Broadcast invokes the named function for every server registered in discovery
func (xc *XClient) Broadcast(ctx context.Context, serviceMethod string, args, reply interface{}) error {
	servers, err := xc.d.GetAll()
	if err != nil {
		return err
	}
	var wg sync.WaitGroup
	var mu sync.Mutex // protect e and replyDone
	var e error
	replyDone := reply == nil // if reply is nil, don't need to set value
	ctx, cancel := context.WithCancel(ctx)
	for _, rpcAddr := range servers {
		wg.Add(1)
		go func(rpcAddr string) {
			defer wg.Done()
			var clonedReply interface{}
			if reply != nil {
				clonedReply = reflect.New(reflect.ValueOf(reply).Elem().Type()).Interface()
			}
			err := xc.call(rpcAddr, ctx, serviceMethod, args, clonedReply)
			mu.Lock()
			if err != nil && e == nil {
				e = err
				cancel() // if any call failed, cancel unfinished calls
			}
			if err == nil && !replyDone {
				reflect.ValueOf(reply).Elem().Set(reflect.ValueOf(clonedReply).Elem())
				replyDone = true
			}
			mu.Unlock()
		}(rpcAddr)
	}
	wg.Wait()
	return e
}
```

Broadcast sends the request to all service instances. If any instance returns an error, one of the errors is returned; if the calls succeed, one of the results is returned. There are several points to note:

1) For better performance, requests are made concurrently.
2) Under concurrency, a mutex is needed to ensure that error and reply are assigned correctly.
3) With the help of context.WithCancel, fail fast when an error occurs.

## Demo

It's Demo time again — let's verify today's work with a simple demo.

First, the code that starts the RPC service is similar to before. Sum is a normal method, while Sleep is used to verify that the timeout mechanism of XClient works properly.

[day6-load-balance/main/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day6-load-balance)

```go
package main

import (
	"context"
	"geerpc"
	"geerpc/xclient"
	"log"
	"net"
	"sync"
	"time"
)

type Foo int

type Args struct{ Num1, Num2 int }

func (f Foo) Sum(args Args, reply *int) error {
	*reply = args.Num1 + args.Num2
	return nil
}

func (f Foo) Sleep(args Args, reply *int) error {
	time.Sleep(time.Second * time.Duration(args.Num1))
	*reply = args.Num1 + args.Num2
	return nil
}

func startServer(addrCh chan string) {
	var foo Foo
	l, _ := net.Listen("tcp", ":0")
	server := geerpc.NewServer()
	_ = server.Register(&foo)
	addrCh <- l.Addr().String()
	server.Accept(l)
}
```

We wrap things in a `foo` function so that after `Call` or `Broadcast`, success or failure logs are printed uniformly.

```go
func foo(xc *xclient.XClient, ctx context.Context, typ, serviceMethod string, args *Args) {
	var reply int
	var err error
	switch typ {
	case "call":
		err = xc.Call(ctx, serviceMethod, args, &reply)
	case "broadcast":
		err = xc.Broadcast(ctx, serviceMethod, args, &reply)
	}
	if err != nil {
		log.Printf("%s %s error: %v", typ, serviceMethod, err)
	} else {
		log.Printf("%s %s success: %d + %d = %d", typ, serviceMethod, args.Num1, args.Num2, reply)
	}
}
```

call invokes a single service instance, while broadcast invokes all service instances.

```go
func call(addr1, addr2 string) {
	d := xclient.NewMultiServerDiscovery([]string{"tcp@" + addr1, "tcp@" + addr2})
	xc := xclient.NewXClient(d, xclient.RandomSelect, nil)
	defer func() { _ = xc.Close() }()
	// send request & receive response
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			foo(xc, context.Background(), "call", "Foo.Sum", &Args{Num1: i, Num2: i * i})
		}(i)
	}
	wg.Wait()
}

func broadcast(addr1, addr2 string) {
	d := xclient.NewMultiServerDiscovery([]string{"tcp@" + addr1, "tcp@" + addr2})
	xc := xclient.NewXClient(d, xclient.RandomSelect, nil)
	defer func() { _ = xc.Close() }()
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			foo(xc, context.Background(), "broadcast", "Foo.Sum", &Args{Num1: i, Num2: i * i})
			// expect 2 - 5 timeout
			ctx, _ := context.WithTimeout(context.Background(), time.Second*2)
			foo(xc, ctx, "broadcast", "Foo.Sleep", &Args{Num1: i, Num2: i * i})
		}(i)
	}
	wg.Wait()
}


func main() {
	log.SetFlags(0)
	ch1 := make(chan string)
	ch2 := make(chan string)
	// start two servers
	go startServer(ch1)
	go startServer(ch2)

	addr1 := <-ch1
	addr2 := <-ch2

	time.Sleep(time.Second)
	call(addr1, addr2)
	broadcast(addr1, addr2)
}
```

The output looks like this:

```go
rpc server: register Foo.Sleep
rpc server: register Foo.Sum
rpc server: register Foo.Sleep
rpc server: register Foo.Sum
call Foo.Sum success: 4 + 16 = 20
call Foo.Sum success: 0 + 0 = 0
call Foo.Sum success: 3 + 9 = 12
call Foo.Sum success: 2 + 4 = 6
call Foo.Sum success: 1 + 1 = 2
broadcast Foo.Sum success: 3 + 9 = 12
broadcast Foo.Sum success: 1 + 1 = 2
broadcast Foo.Sum success: 0 + 0 = 0
broadcast Foo.Sum success: 4 + 16 = 20
broadcast Foo.Sum success: 2 + 4 = 6
broadcast Foo.Sleep success: 0 + 0 = 0
broadcast Foo.Sleep success: 1 + 1 = 2
broadcast Foo.Sleep error: rpc client: call failed: context deadline exceeded
broadcast Foo.Sleep error: rpc client: call failed: context deadline exceeded
broadcast Foo.Sleep error: rpc client: call failed: context deadline exceeded
```

## Appendix: Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [Go Interview Questions](https://geektutu.com/post/qa-golang.html)
