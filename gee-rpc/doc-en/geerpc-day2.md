---
title: Implement an RPC Framework in Go - GeeRPC Day 2 Asynchronous and Concurrent Client
description: >-
  A 7-day tutorial on implementing an RPC framework in Go/golang from scratch (7 days implement golang remote procedure call framework from scratch
  tutorial). Build an RPC framework modeled after the implementation of the golang standard library net/rpc, covering the server, an asynchronous and
  concurrent client, message encoding and decoding, service registration, and multiple transport protocols including TCP/Unix/HTTP. On Day 2, we implement a client that supports asynchronous (asynchronous) and concurrent (concurrent) calls.
date: '2020-10-08 02:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geerpc/geerpc.jpg
lang: en
---

![golang RPC framework](geerpc/geerpc.jpg)

This is the second article in the [7 Days to Implement an RPC Framework GeeRPC in Go from Scratch](https://geektutu.com/post/geerpc.html) series.

- Implement a high-performance client that supports asynchronous and concurrent calls, in about 250 lines of code


## Designing Call

For `net/rpc`, a function must meet the following five requirements to be invokable remotely:

- the method's type is exported.
- the method is exported.
- the method has two arguments, both exported (or builtin) types.
- the method's second argument is a pointer.
- the method has return type error.

To make it more intuitive:

```go
func (t *T) MethodName(argType T1, replyType *T2) error
```

According to the above requirements, we first define the Call struct to carry the information needed for an RPC call.

[day2-client/client.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day2-client)

```go
// Call represents an active RPC.
type Call struct {
	Seq           uint64
	ServiceMethod string      // format "<service>.<method>"
	Args          interface{} // arguments to the function
	Reply         interface{} // reply from the function
	Error         error       // if error occurs, it will be set
	Done          chan *Call  // Strobes when call is complete.
}

func (call *Call) done() {
	call.Done <- call
}
```

To support asynchronous calls, a field Done is added to the Call struct. Done is of type `chan *Call`, and when the call completes, `call.done()` is invoked to notify the caller.


## Implementing Client

Next, we will implement Client, the most essential part of the GeeRPC client.

```go
// Client represents an RPC Client.
// There may be multiple outstanding Calls associated
// with a single Client, and a Client may be used by
// multiple goroutines simultaneously.
type Client struct {
	cc       codec.Codec
	opt      *Option
	sending  sync.Mutex // protect following
	header   codec.Header
	mu       sync.Mutex // protect following
	seq      uint64
	pending  map[uint64]*Call
	closing  bool // user has called Close
	shutdown bool // server has told us to stop
}

var _ io.Closer = (*Client)(nil)

var ErrShutdown = errors.New("connection is shut down")

// Close the connection
func (client *Client) Close() error {
	client.mu.Lock()
	defer client.mu.Unlock()
	if client.closing {
		return ErrShutdown
	}
	client.closing = true
	return client.cc.Close()
}

// IsAvailable return true if the client does work
func (client *Client) IsAvailable() bool {
	client.mu.Lock()
	defer client.mu.Unlock()
	return !client.shutdown && !client.closing
}
```

The fields of Client are rather involved:

- cc is the message codec, similar to the server's, used to serialize outgoing requests and deserialize incoming responses.
- sending is a mutex, similar to the server's, to ensure requests are sent in order, i.e., to prevent multiple request messages from getting mixed up.
- header is the message header for each request. The header is only needed when a request is being sent, and request sending is mutually exclusive, so each client only needs one; declaring it in the Client struct allows it to be reused.
- seq is used to number outgoing requests; each request has a unique number.
- pending stores requests that have not been fully processed; the key is the number, and the value is the Call instance.
- closing and shutdown: if either is set to true, the Client is in an unusable state, but there is a subtle difference — closing is set when the user actively closes it by calling `Close`, while shutdown is usually set when an error occurs.

Next, implement the three methods related to Call.

```go
func (client *Client) registerCall(call *Call) (uint64, error) {
	client.mu.Lock()
	defer client.mu.Unlock()
	if client.closing || client.shutdown {
		return 0, ErrShutdown
	}
	call.Seq = client.seq
	client.pending[call.Seq] = call
	client.seq++
	return call.Seq, nil
}

func (client *Client) removeCall(seq uint64) *Call {
	client.mu.Lock()
	defer client.mu.Unlock()
	call := client.pending[seq]
	delete(client.pending, seq)
	return call
}

func (client *Client) terminateCalls(err error) {
	client.sending.Lock()
	defer client.sending.Unlock()
	client.mu.Lock()
	defer client.mu.Unlock()
	client.shutdown = true
	for _, call := range client.pending {
		call.Error = err
		call.done()
	}
}
```

- registerCall: adds the call parameter to client.pending and updates client.seq.
- removeCall: removes the corresponding call from client.pending by seq and returns it.
- terminateCalls: called when an error occurs on either the server or the client; it sets shutdown to true and notifies all pending calls of the error.

For a client, receiving responses and sending requests are the two most important capabilities. Let's implement receiving first. A received response falls into three cases:

- The call does not exist, possibly because the request was not sent completely or was cancelled for other reasons, but the server still processed it.
- The call exists, but the server failed to process it, i.e., h.Error is not empty.
- The call exists and the server processed it normally; in this case, the value of Reply needs to be read from the body.

```go
func (client *Client) receive() {
	var err error
	for err == nil {
		var h codec.Header
		if err = client.cc.ReadHeader(&h); err != nil {
			break
		}
		call := client.removeCall(h.Seq)
		switch {
		case call == nil:
			// it usually means that Write partially failed
			// and call was already removed.
			err = client.cc.ReadBody(nil)
		case h.Error != "":
			call.Error = fmt.Errorf(h.Error)
			err = client.cc.ReadBody(nil)
			call.done()
		default:
			err = client.cc.ReadBody(call.Reply)
			if err != nil {
				call.Error = errors.New("reading body " + err.Error())
			}
			call.done()
		}
	}
	// error occurs, so terminateCalls pending calls
	client.terminateCalls(err)
}
```

When creating a Client instance, the initial protocol exchange must be completed first, i.e., sending the `Option` information to the server. After negotiating the message encoding method, a goroutine is spawned to call `receive()` to receive responses.

```go
func NewClient(conn net.Conn, opt *Option) (*Client, error) {
	f := codec.NewCodecFuncMap[opt.CodecType]
	if f == nil {
		err := fmt.Errorf("invalid codec type %s", opt.CodecType)
		log.Println("rpc client: codec error:", err)
		return nil, err
	}
	// send options with server
	if err := json.NewEncoder(conn).Encode(opt); err != nil {
		log.Println("rpc client: options error: ", err)
		_ = conn.Close()
		return nil, err
	}
	return newClientCodec(f(conn), opt), nil
}

func newClientCodec(cc codec.Codec, opt *Option) *Client {
	client := &Client{
		seq:     1, // seq starts with 1, 0 means invalid call
		cc:      cc,
		opt:     opt,
		pending: make(map[uint64]*Call),
	}
	go client.receive()
	return client
}
```

We also need to implement the `Dial` function so that users can pass in the server address to create a Client instance. To simplify usage, Option is made an optional parameter via `...*Option`.

```go
func parseOptions(opts ...*Option) (*Option, error) {
	// if opts is nil or pass nil as parameter
	if len(opts) == 0 || opts[0] == nil {
		return DefaultOption, nil
	}
	if len(opts) != 1 {
		return nil, errors.New("number of options is more than 1")
	}
	opt := opts[0]
	opt.MagicNumber = DefaultOption.MagicNumber
	if opt.CodecType == "" {
		opt.CodecType = DefaultOption.CodecType
	}
	return opt, nil
}

// Dial connects to an RPC server at the specified network address
func Dial(network, address string, opts ...*Option) (client *Client, err error) {
	opt, err := parseOptions(opts...)
	if err != nil {
		return nil, err
	}
	conn, err := net.Dial(network, address)
	if err != nil {
		return nil, err
	}
	// close the connection if client is nil
	defer func() {
		if client == nil {
			_ = conn.Close()
		}
	}()
	return NewClient(conn, opt)
}
```

At this point, the GeeRPC client is already capable of establishing connections and receiving responses. The last capability to implement is sending requests.

```go
func (client *Client) send(call *Call) {
	// make sure that the client will send a complete request
	client.sending.Lock()
	defer client.sending.Unlock()

	// register this call.
	seq, err := client.registerCall(call)
	if err != nil {
		call.Error = err
		call.done()
		return
	}

	// prepare request header
	client.header.ServiceMethod = call.ServiceMethod
	client.header.Seq = seq
	client.header.Error = ""

	// encode and send the request
	if err := client.cc.Write(&client.header, call.Args); err != nil {
		call := client.removeCall(seq)
		// call may be nil, it usually means that Write partially failed,
		// client has received the response and handled
		if call != nil {
			call.Error = err
			call.done()
		}
	}
}

// Go invokes the function asynchronously.
// It returns the Call structure representing the invocation.
func (client *Client) Go(serviceMethod string, args, reply interface{}, done chan *Call) *Call {
	if done == nil {
		done = make(chan *Call, 10)
	} else if cap(done) == 0 {
		log.Panic("rpc client: done channel is unbuffered")
	}
	call := &Call{
		ServiceMethod: serviceMethod,
		Args:          args,
		Reply:         reply,
		Done:          done,
	}
	client.send(call)
	return call
}

// Call invokes the named function, waits for it to complete,
// and returns its error status.
func (client *Client) Call(serviceMethod string, args, reply interface{}) error {
	call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
	return call.Error
}
```

- `Go` and `Call` are the two RPC invocation interfaces exposed to users by the client. `Go` is an asynchronous interface that returns the call instance.
- `Call` is a wrapper around `Go` that blocks on call.Done, waiting for the response to return; it is a synchronous interface.

At this point, a GeeRPC client that supports asynchronous and concurrent calls is complete.

## Demo

On day 1, GeeRPC only implemented the server, so we manually simulated the whole communication process in the main function. Today, let's replace the communication part in the main function with the client we just built.

[day2-client/main/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day2-client)

startServer remains unchanged.

```go
func startServer(addr chan string) {
	// pick a free port
	l, err := net.Listen("tcp", ":0")
	if err != nil {
		log.Fatal("network error:", err)
	}
	log.Println("start rpc server on", l.Addr())
	addr <- l.Addr().String()
	geerpc.Accept(l)
}
```

In the main function, `client.Call` is used to make 5 concurrent RPC synchronous calls, with both arguments and return values of type string.

```go
func main() {
    log.SetFlags(0)
	addr := make(chan string)
	go startServer(addr)
	client, _ := geerpc.Dial("tcp", <-addr)
	defer func() { _ = client.Close() }()

	time.Sleep(time.Second)
	// send request & receive response
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			args := fmt.Sprintf("geerpc req %d", i)
			var reply string
			if err := client.Call("Foo.Sum", args, &reply); err != nil {
				log.Fatal("call Foo.Sum error:", err)
			}
			log.Println("reply:", reply)
		}(i)
	}
	wg.Wait()
}
```

The output looks like this:

```bash
start rpc server on [::]:50658
&{Foo.Sum 5 } geerpc req 3
&{Foo.Sum 1 } geerpc req 0
&{Foo.Sum 3 } geerpc req 1
&{Foo.Sum 2 } geerpc req 4
&{Foo.Sum 4 } geerpc req 2
reply: geerpc resp 1
reply: geerpc resp 5
reply: geerpc resp 3
reply: geerpc resp 2
reply: geerpc resp 4
```

## Appendix: Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [Go Interview Questions](https://geektutu.com/post/qa-golang.html)
