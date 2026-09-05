---
title: Implement an RPC Framework in Go - GeeRPC Day 1 Server and Message Encoding
description: >-
  A 7-day tutorial on implementing an RPC framework in Go/golang from scratch (7 days implement golang remote procedure call framework from scratch
  tutorial). Build an RPC framework modeled after the implementation of the golang standard library net/rpc, covering the server, an asynchronous and
  concurrent client, message encoding and decoding, service registration, and multiple transport protocols including TCP/Unix/HTTP. On Day 1, we implement a simple server along with message encoding and decoding.
date: '2020-10-07 01:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geerpc/geerpc.jpg
lang: en
---

![golang RPC framework](geerpc/geerpc.jpg)

This is the first article in the [7 Days to Implement an RPC Framework GeeRPC in Go from Scratch](https://geektutu.com/post/geerpc.html) series.

- Use `encoding/gob` to implement message encoding/decoding (serialization and deserialization)
- Implement a simple server that only receives messages without processing them, in about 200 lines of code


## Message Serialization and Deserialization

A typical RPC call looks like this:

```go
err = client.Call("Arith.Multiply", args, &reply)
```

The client's request consists of three parts: the service name `Arith`, the method name `Multiply`, and the parameters `args`; the server's response consists of two parts: the error `error` and the return value `reply`. If we abstract the parameters and return values from the request and response as the body, and put the remaining information in the header, we can define the Header data structure:

[day1-codec/codec/codec.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day1-codec)

```go
package codec

import "io"

type Header struct {
	ServiceMethod string // format "Service.Method"
	Seq           uint64 // sequence number chosen by client
	Error         string
}
```

- ServiceMethod is the service name and method name, usually mapped to a struct and method in Go.
- Seq is the sequence number of the request, which can also be seen as the request's ID, used to distinguish different requests.
- Error is the error message; the client sets it to empty, and if the server encounters an error, it places the error message in Error.


We put all the message encoding/decoding related code in the codec subdirectory. Before that, we need to initialize the project in the root directory with `go mod init geerpc` to make cross-package references easier later.

Going one step further, we abstract the interface Codec for encoding/decoding message bodies. The purpose of defining an interface is to allow different Codec implementations:

```go
type Codec interface {
	io.Closer
	ReadHeader(*Header) error
	ReadBody(interface{}) error
	Write(*Header, interface{}) error
}
```

Right after that, we abstract the constructor functions for Codec. The client and server can obtain the constructor through the Codec's `Type` and thus create Codec instances. This part of the code is similar to the factory pattern; the difference is that it returns constructor functions rather than instances.

```go
type NewCodecFunc func(io.ReadWriteCloser) Codec

type Type string

const (
	GobType  Type = "application/gob"
	JsonType Type = "application/json" // not implemented
)

var NewCodecFuncMap map[Type]NewCodecFunc

func init() {
	NewCodecFuncMap = make(map[Type]NewCodecFunc)
	NewCodecFuncMap[GobType] = NewGobCodec
}
```

We define two Codecs, `Gob` and `Json`, but only `Gob` is actually implemented in the code. In fact, the two implementations are very close — you could switch from `gob` to `json` simply by swapping the package.

First, define the `GobCodec` struct, which consists of four parts: `conn` is passed in by the constructor, usually the connection instance obtained when establishing a socket over TCP or Unix; dec and enc correspond to gob's Decoder and Encoder; buf is a buffered `Writer` created to avoid blocking, which generally improves performance.

[day1-codec/codec/gob.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day1-codec)

```go
package codec

import (
	"bufio"
	"encoding/gob"
	"io"
	"log"
)

type GobCodec struct {
	conn io.ReadWriteCloser
	buf  *bufio.Writer
	dec  *gob.Decoder
	enc  *gob.Encoder
}

var _ Codec = (*GobCodec)(nil)

func NewGobCodec(conn io.ReadWriteCloser) Codec {
	buf := bufio.NewWriter(conn)
	return &GobCodec{
		conn: conn,
		buf:  buf,
		dec:  gob.NewDecoder(conn),
		enc:  gob.NewEncoder(buf),
	}
}
```

Then implement the `ReadHeader`, `ReadBody`, `Write` and `Close` methods.

```go
func (c *GobCodec) ReadHeader(h *Header) error {
	return c.dec.Decode(h)
}

func (c *GobCodec) ReadBody(body interface{}) error {
	return c.dec.Decode(body)
}

func (c *GobCodec) Write(h *Header, body interface{}) (err error) {
	defer func() {
		_ = c.buf.Flush()
		if err != nil {
			_ = c.Close()
		}
	}()
	if err := c.enc.Encode(h); err != nil {
		log.Println("rpc codec: gob error encoding header:", err)
		return err
	}
	if err := c.enc.Encode(body); err != nil {
		log.Println("rpc codec: gob error encoding body:", err)
		return err
	}
	return nil
}

func (c *GobCodec) Close() error {
	return c.conn.Close()
}
```

## The Communication Process

The client and server need to negotiate some content before communicating. Take the HTTP message for example: it is divided into a header and a body, where the format and length of the body are specified by `Content-Type` and `Content-Length` in the header; by parsing the header, the server knows how to read the required information from the body. For an RPC protocol, this negotiation must be designed by ourselves. For performance, the first few bytes of a message are usually reserved to negotiate related information. For example, byte 1 indicates the serialization method, byte 2 the compression method, bytes 3-6 the header length, and bytes 7-10 the body length.

For GeeRPC, the only item that needs to be negotiated at the moment is the message encoding method. We carry this information in the `Option` struct. Now, we move on to implementing the server.

[day1-codec/server.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day1-codec)

```go
package geerpc

const MagicNumber = 0x3bef5c

type Option struct {
	MagicNumber int        // MagicNumber marks this's a geerpc request
	CodecType   codec.Type // client may choose different Codec to encode body
}

var DefaultOption = &Option{
	MagicNumber: MagicNumber,
	CodecType:   codec.GobType,
}
```

Generally, the information involved in protocol negotiation is transmitted in fixed bytes. However, to keep the implementation simpler, the GeeRPC client always encodes Option in JSON, while the encoding methods of the subsequent header and body are specified by CodecType in Option. The server first decodes Option with JSON, then decodes the rest of the message according to the Option's CodecType. That is, the message is sent in the following form:

```bash
| Option{MagicNumber: xxx, CodecType: xxx} | Header{ServiceMethod ...} | Body interface{} |
| <------      fixed JSON encoding      ------>  | <-------   encoding determined by CodecType   ------->|
```

Within a single connection, Option is fixed at the very beginning of the message, and there can be multiple Headers and Bodies, so the message may look like this:

```go
| Option | Header1 | Body1 | Header2 | Body2 | ...
```

## Implementing the Server

With the communication process clearly defined, the server implementation is fairly straightforward.

[day1-codec/server.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day1-codec)

```go
// Server represents an RPC Server.
type Server struct{}

// NewServer returns a new Server.
func NewServer() *Server {
	return &Server{}
}

// DefaultServer is the default instance of *Server.
var DefaultServer = NewServer()

// Accept accepts connections on the listener and serves requests
// for each incoming connection.
func (server *Server) Accept(lis net.Listener) {
	for {
		conn, err := lis.Accept()
		if err != nil {
			log.Println("rpc server: accept error:", err)
			return
		}
		go server.ServeConn(conn)
	}
}

// Accept accepts connections on the listener and serves requests
// for each incoming connection.
func Accept(lis net.Listener) { DefaultServer.Accept(lis) }
```

- First, define the `Server` struct, which has no member fields at all.
- Implement the `Accept` method, which takes `net.Listener` as its parameter, and waits for socket connections in a for loop, spawning a goroutine for each connection, with the handling delegated to the `ServeConn` method.
- DefaultServer is a default `Server` instance, mainly for the convenience of users.

Starting a server is very simple: just pass in a listener; both tcp and unix protocols are supported.

```go
lis, _ := net.Listen("tcp", ":9999")
geerpc.Accept(lis)
```

The implementation of `ServeConn` is closely related to the communication process discussed earlier. It first uses `json.NewDecoder` to deserialize and get an Option instance, then checks whether MagicNumber and CodecType are valid. Then it obtains the corresponding message codec based on CodecType, and hands over the subsequent processing to `serverCodec`.

```go
// ServeConn runs the server on a single connection.
// ServeConn blocks, serving the connection until the client hangs up.
func (server *Server) ServeConn(conn io.ReadWriteCloser) {
	defer func() { _ = conn.Close() }()
	var opt Option
	if err := json.NewDecoder(conn).Decode(&opt); err != nil {
		log.Println("rpc server: options error: ", err)
		return
	}
	if opt.MagicNumber != MagicNumber {
		log.Printf("rpc server: invalid magic number %x", opt.MagicNumber)
		return
	}
	f := codec.NewCodecFuncMap[opt.CodecType]
	if f == nil {
		log.Printf("rpc server: invalid codec type %s", opt.CodecType)
		return
	}
	server.serveCodec(f(conn))
}

// invalidRequest is a placeholder for response argv when error occurs
var invalidRequest = struct{}{}

func (server *Server) serveCodec(cc codec.Codec) {
	sending := new(sync.Mutex) // make sure to send a complete response
	wg := new(sync.WaitGroup)  // wait until all request are handled
	for {
		req, err := server.readRequest(cc)
		if err != nil {
			if req == nil {
				break // it's not possible to recover, so close the connection
			}
			req.h.Error = err.Error()
			server.sendResponse(cc, req.h, invalidRequest, sending)
			continue
		}
		wg.Add(1)
		go server.handleRequest(cc, req, sending, wg)
	}
	wg.Wait()
	_ = cc.Close()
}
```

The process of `serveCodec` is very simple. It mainly involves three stages:

- Reading requests: readRequest
- Handling requests: handleRequest
- Replying to requests: sendResponse

As mentioned before, within a single connection, multiple requests are allowed, i.e., multiple request headers and request bodies. Therefore, a for loop is used here to wait indefinitely for requests to arrive until an error occurs (e.g., the connection is closed, or the received message is malformed). There are three points to note here:

- handleRequest uses goroutines to execute requests concurrently.
- Request handling is concurrent, but response messages must be sent one by one; concurrency can easily cause multiple response messages to interleave, making them impossible for the client to parse. A lock (sending) is used to guarantee this.
- Best effort: the loop terminates only when the header fails to parse.

```go
// request stores all information of a call
type request struct {
	h            *codec.Header // header of request
	argv, replyv reflect.Value // argv and replyv of request
}

func (server *Server) readRequestHeader(cc codec.Codec) (*codec.Header, error) {
	var h codec.Header
	if err := cc.ReadHeader(&h); err != nil {
		if err != io.EOF && err != io.ErrUnexpectedEOF {
			log.Println("rpc server: read header error:", err)
		}
		return nil, err
	}
	return &h, nil
}

func (server *Server) readRequest(cc codec.Codec) (*request, error) {
	h, err := server.readRequestHeader(cc)
	if err != nil {
		return nil, err
	}
	req := &request{h: h}
	// TODO: now we don't know the type of request argv
	// day 1, just suppose it's string
	req.argv = reflect.New(reflect.TypeOf(""))
	if err = cc.ReadBody(req.argv.Interface()); err != nil {
		log.Println("rpc server: read argv err:", err)
	}
	return req, nil
}

func (server *Server) sendResponse(cc codec.Codec, h *codec.Header, body interface{}, sending *sync.Mutex) {
	sending.Lock()
	defer sending.Unlock()
	if err := cc.Write(h, body); err != nil {
		log.Println("rpc server: write response error:", err)
	}
}

func (server *Server) handleRequest(cc codec.Codec, req *request, sending *sync.Mutex, wg *sync.WaitGroup) {
	// TODO, should call registered rpc methods to get the right replyv
	// day 1, just print argv and send a hello message
	defer wg.Done()
	log.Println(req.h, req.argv.Elem())
	req.replyv = reflect.ValueOf(fmt.Sprintf("geerpc resp %d", req.h.Seq))
	server.sendResponse(cc, req.h, req.replyv.Interface(), sending)
}
```

For now, we cannot determine the type of the body, so in readRequest and handleRequest, day1 treats the body as a string. Upon receiving a request, it prints the header and replies with `geerpc resp ${req.h.Seq}`. This part will be implemented later.


## The main Function (A Simple Client)

That's all for day 1. Here we have implemented a message codec `GobCodec`, and the client and server have achieved a simple protocol exchange, allowing the client to use different encoding methods. We have also implemented a prototype of the server: establishing connections, reading, processing and replying to client requests.

Next, let's take a look at how to use the GeeRPC we just implemented in the main function.

[day1-codec/main/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day1-codec)

```go
package main

import (
	"encoding/json"
	"fmt"
	"geerpc"
	"geerpc/codec"
	"log"
	"net"
	"time"
)

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

func main() {
	addr := make(chan string)
	go startServer(addr)

	// in fact, following code is like a simple geerpc client
	conn, _ := net.Dial("tcp", <-addr)
	defer func() { _ = conn.Close() }()

	time.Sleep(time.Second)
	// send options
	_ = json.NewEncoder(conn).Encode(geerpc.DefaultOption)
	cc := codec.NewGobCodec(conn)
	// send request & receive response
	for i := 0; i < 5; i++ {
		h := &codec.Header{
			ServiceMethod: "Foo.Sum",
			Seq:           uint64(i),
		}
		_ = cc.Write(h, fmt.Sprintf("geerpc req %d", h.Seq))
		_ = cc.ReadHeader(h)
		var reply string
		_ = cc.ReadBody(&reply)
		log.Println("reply:", reply)
	}
}
```

- In `startServer`, the channel `addr` is used to ensure that the server port is successfully listening before the client sends requests.
- The client first sends `Option` for protocol exchange, then sends the message header `h := &codec.Header{}` and the message body `geerpc req ${h.Seq}`.
- Finally, it parses the server's response `reply` and prints it.

The output looks like this:

```bash
start rpc server on [::]:63662
&{Foo.Sum 0 } geerpc req 0
reply: geerpc resp 0
&{Foo.Sum 1 } geerpc req 1
reply: geerpc resp 1
&{Foo.Sum 2 } geerpc req 2
reply: geerpc resp 2
&{Foo.Sum 3 } geerpc req 3
reply: geerpc resp 3
&{Foo.Sum 4 } geerpc req 4
reply: geerpc resp 4
```

## Appendix: Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [Go Interview Questions](https://geektutu.com/post/qa-golang.html)
