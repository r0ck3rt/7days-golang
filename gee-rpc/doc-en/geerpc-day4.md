---
title: Implement an RPC Framework in Go - GeeRPC Day 4 Timeout Processing
description: >-
  A 7-day tutorial on implementing the RPC framework GeeRPC in Go/golang from scratch (7 days
  implement golang remote procedure call framework from scratch tutorial). Build an RPC
  framework modeled after the implementation of Go's standard library net/rpc, covering the
  server, a client supporting asynchronous and concurrent calls, message encoding and
  decoding, service registration, and multiple transport protocols such as TCP/Unix/HTTP.
  Day 4 provides the RPC framework with the ability to handle timeouts (timeout processing).
date: '2020-10-08 07:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geerpc/geerpc.jpg
lang: en
---

![golang RPC framework](geerpc/geerpc.jpg)

This article is the fourth part of the [7 Days Go RPC Framework Tutorial Series from scratch](https://geektutu.com/post/geerpc.html).

- Add a mechanism for handling connection timeouts
- Add a mechanism for handling server-side processing timeouts, in about 100 lines of code

## Why Timeout Processing Is Needed

Timeout handling is a fairly basic capability of an RPC framework. Without it, both the server and the client can easily hang and exhaust resources due to network or other errors, and these problems greatly reduce service availability. Therefore, we need to add timeout handling to the RPC framework.

Looking at the entire remote-call process, the client needs to handle timeouts in the following places:

- establishing a connection with the server, which can time out
- writing the message when sending a request to the server
- waiting for the server to process the request (for example, the server has hung and does not respond)
- reading the message when receiving the response from the server

The server needs to handle timeouts in the following places:

- reading the client's request message
- writing the response message
- processing the message when invoking the mapped service method


GeeRPC adds timeout handling in three places:

1) when the client creates a connection
2) the entire process of the client's `Client.Call()` (covering all stages: sending the message, waiting for processing, and receiving the message)
3) the server processing messages, i.e. a timeout in `Server.handleRequest`.

## Connection Timeout

For simplicity, the timeout settings are placed in Option. ConnectTimeout defaults to 10s, and HandleTimeout defaults to 0, meaning no limit.

```go
type Option struct {
	MagicNumber    int           // MagicNumber marks this's a geerpc request
	CodecType      codec.Type    // client may choose different Codec to encode body
	ConnectTimeout time.Duration // 0 means no limit
	HandleTimeout  time.Duration
}

var DefaultOption = &Option{
	MagicNumber:    MagicNumber,
	CodecType:      codec.GobType,
	ConnectTimeout: time.Second * 10,
}
```

For the client connection timeout, we only need to wrap Dial with a timeout-handling shell.

[day4-timeout/client.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day4-timeout)

```go
type clientResult struct {
	client *Client
	err    error
}

type newClientFunc func(conn net.Conn, opt *Option) (client *Client, err error)

func dialTimeout(f newClientFunc, network, address string, opts ...*Option) (client *Client, err error) {
	opt, err := parseOptions(opts...)
	if err != nil {
		return nil, err
	}
	conn, err := net.DialTimeout(network, address, opt.ConnectTimeout)
	if err != nil {
		return nil, err
	}
	// close the connection if client is nil
	defer func() {
		if err != nil {
			_ = conn.Close()
		}
	}()
	ch := make(chan clientResult)
	go func() {
		client, err := f(conn, opt)
		ch <- clientResult{client: client, err: err}
	}()
	if opt.ConnectTimeout == 0 {
		result := <-ch
		return result.client, result.err
	}
	select {
	case <-time.After(opt.ConnectTimeout):
		return nil, fmt.Errorf("rpc client: connect timeout: expect within %s", opt.ConnectTimeout)
	case result := <-ch:
		return result.client, result.err
	}
}

// Dial connects to an RPC server at the specified network address
func Dial(network, address string, opts ...*Option) (*Client, error) {
	return dialTimeout(NewClient, network, address, opts...)
}
```

Here we implement a timeout-handling shell `dialTimeout`. This shell takes NewClient as an argument and adds the timeout mechanism in two places.

1) replace `net.Dial` with `net.DialTimeout`, so an error is returned if the connection creation times out.
2) run NewClient in a sub-goroutine; when it finishes, the result is sent through the channel ch. If the `time.After()` channel receives a message first, NewClient has timed out and an error is returned.

## Client.Call Timeout

The timeout mechanism of `Client.Call` is implemented with the context package, handing control over to the user for more flexible control.

```go
// Call invokes the named function, waits for it to complete,
// and returns its error status.
func (client *Client) Call(ctx context.Context, serviceMethod string, args, reply interface{}) error {
	call := client.Go(serviceMethod, args, reply, make(chan *Call, 1))
	select {
	case <-ctx.Done():
		client.removeCall(call.Seq)
		return errors.New("rpc client: call failed: " + ctx.Err().Error())
	case call := <-call.Done:
		return call.Error
	}
}
```

Users can create a context object with timeout detection via `context.WithTimeout` to control the call. For example:

```go
ctx, _ := context.WithTimeout(context.Background(), time.Second)
var reply int
err := client.Call(ctx, "Foo.Sum", &Args{1, 2}, &reply)
...
```

## Server-side Processing Timeout

The implementation of this part is very close to the client's, done with `time.After()` combined with `select+chan`.

[day4-timeout/server.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day4-timeout)

```go
func (server *Server) handleRequest(cc codec.Codec, req *request, sending *sync.Mutex, wg *sync.WaitGroup, timeout time.Duration) {
	defer wg.Done()
	called := make(chan struct{})
	sent := make(chan struct{})
	go func() {
		err := req.svc.call(req.mtype, req.argv, req.replyv)
		called <- struct{}{}
		if err != nil {
			req.h.Error = err.Error()
			server.sendResponse(cc, req.h, invalidRequest, sending)
			sent <- struct{}{}
			return
		}
		server.sendResponse(cc, req.h, req.replyv.Interface(), sending)
		sent <- struct{}{}
	}()

	if timeout == 0 {
		<-called
		<-sent
		return
	}
	select {
	case <-time.After(timeout):
		req.h.Error = fmt.Sprintf("rpc server: request handle timeout: expect within %s", timeout)
		server.sendResponse(cc, req.h, invalidRequest, sending)
	case <-called:
		<-sent
	}
}
```

Here we need to ensure that `sendResponse` is called only once, so the entire process is split into two stages, `called` and `sent`. In this code, only one of the following two cases can occur:

1) the called channel receives a message, which means processing did not time out; continue to execute sendResponse.
2) `time.After()` receives a message before called, which means processing has already timed out; both called and sent will be blocked, and `sendResponse` is called at `case <-time.After(timeout)`.

## Test Cases

The first test case tests the connection timeout. The NewClient function takes 2s, with two scenarios: ConnectionTimeout set to 1s, and set to 0.

[day4-timeout/client_test.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day4-timeout)

```go
func TestClient_dialTimeout(t *testing.T) {
	t.Parallel()
	l, _ := net.Listen("tcp", ":0")

	f := func(conn net.Conn, opt *Option) (client *Client, err error) {
		_ = conn.Close()
		time.Sleep(time.Second * 2)
		return nil, nil
	}
	t.Run("timeout", func(t *testing.T) {
		_, err := dialTimeout(f, "tcp", l.Addr().String(), &Option{ConnectTimeout: time.Second})
		_assert(err != nil && strings.Contains(err.Error(), "connect timeout"), "expect a timeout error")
	})
	t.Run("0", func(t *testing.T) {
		_, err := dialTimeout(f, "tcp", l.Addr().String(), &Option{ConnectTimeout: 0})
		_assert(err == nil, "0 means no limit")
	})
}
```

The second test case tests the processing timeout. `Bar.Timeout` takes 2s. Scenario one: the client sets the timeout to 1s while the server has no limit; scenario two: the server sets the timeout to 1s while the client has no limit.

```go
type Bar int

func (b Bar) Timeout(argv int, reply *int) error {
	time.Sleep(time.Second * 2)
	return nil
}

func startServer(addr chan string) {
	var b Bar
	_ = Register(&b)
	// pick a free port
	l, _ := net.Listen("tcp", ":0")
	addr <- l.Addr().String()
	Accept(l)
}

func TestClient_Call(t *testing.T) {
	t.Parallel()
	addrCh := make(chan string)
	go startServer(addrCh)
	addr := <-addrCh
	time.Sleep(time.Second)
	t.Run("client timeout", func(t *testing.T) {
		client, _ := Dial("tcp", addr)
		ctx, _ := context.WithTimeout(context.Background(), time.Second)
		var reply int
		err := client.Call(ctx, "Bar.Timeout", 1, &reply)
		_assert(err != nil && strings.Contains(err.Error(), ctx.Err().Error()), "expect a timeout error")
	})
	t.Run("server handle timeout", func(t *testing.T) {
		client, _ := Dial("tcp", addr, &Option{
			HandleTimeout: time.Second,
		})
		var reply int
		err := client.Call(context.Background(), "Bar.Timeout", 1, &reply)
		_assert(err != nil && strings.Contains(err.Error(), "handle timeout"), "expect a timeout error")
	})
}
```

## Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [Go Written Test and Interview Questions](https://geektutu.com/post/qa-golang.html)
