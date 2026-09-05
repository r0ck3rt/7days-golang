---
title: Implement an RPC Framework in Go - GeeRPC Day 3 Service Registration
description: >-
  A 7-day tutorial on implementing the RPC framework GeeRPC in Go/golang from scratch (7 days
  implement golang remote procedure call framework from scratch tutorial). Build an RPC
  framework modeled after the implementation of Go's standard library net/rpc, covering the
  server, a client supporting asynchronous and concurrent calls, message encoding and
  decoding, service registration, and multiple transport protocols such as TCP/Unix/HTTP.
  Day 3 implements service registration, mapping Go structs to services via reflection.
date: '2020-10-08 03:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geerpc/geerpc.jpg
lang: en
---

![golang RPC framework](geerpc/geerpc.jpg)

This article is the third part of the [7 Days Go RPC Framework Tutorial Series from scratch](https://geektutu.com/post/geerpc.html).

- Implement service registration via reflection
- Implement service invocation on the server side, in about 150 lines of code

## Mapping Structs to Services

A fundamental capability of an RPC framework is calling remote services as if they were local programs. So how do we map a program to a service? For Go, this question becomes: how do we map the methods of a struct to a service?

For `net/rpc`, a function must satisfy the following five conditions to be callable remotely:

- the method's type is exported.
- the method is exported.
- the method has two arguments, both exported (or builtin) types.
- the method's second argument is a pointer.
- the method has return type error.

More intuitively:

```go
func (t *T) MethodName(argType T1, replyType *T2) error
```

Suppose the client sends a request containing ServiceMethod and Argv.

```json
{
    "ServiceMethod"： "T.MethodName"
    "Argv"："0101110101..." // the serialized byte stream
}
```

From "T.MethodName" we can determine that the method being called is MethodName of type T. If this were implemented with hard-coded logic, it would probably look like this:

```go
switch req.ServiceMethod {
    case "T.MethodName":
        t := new(t)
        reply := new(T2)
        var argv T1
        gob.NewDecoder(conn).Decode(&argv)
        err := t.MethodName(argv, reply)
        server.sendMessage(reply, err)
    case "Foo.Sum":
        f := new(Foo)
        ...
}
```

In other words, if the mapping between structs and services is implemented with hard-coded logic, every exposed method requires an equal amount of code. Is there a way to automate this mapping process? Reflection comes to the rescue.

With reflection, we can easily obtain all the methods of a struct, and from each method, all of its parameter types and return values. For example:

```go
func main() {
	var wg sync.WaitGroup
	typ := reflect.TypeOf(&wg)
	for i := 0; i < typ.NumMethod(); i++ {
		method := typ.Method(i)
		argv := make([]string, 0, method.Type.NumIn())
		returns := make([]string, 0, method.Type.NumOut())
		// j starts from 1; the 0th argument is wg itself.
		for j := 1; j < method.Type.NumIn(); j++ {
			argv = append(argv, method.Type.In(j).Name())
		}
		for j := 0; j < method.Type.NumOut(); j++ {
			returns = append(returns, method.Type.Out(j).Name())
		}
		log.Printf("func (w *%s) %s(%s) %s",
			typ.Elem().Name(),
			method.Name,
			strings.Join(argv, ","),
			strings.Join(returns, ","))
    }
}
```

The output is:

```go
func (w *WaitGroup) Add(int)
func (w *WaitGroup) Done()
func (w *WaitGroup) Wait()
```

## Implementing service via Reflection

In the previous two days we completed the client and the server. The client is relatively complete in functionality, but the server is not: it merely prints the request header without actually processing it. Today's main goal is to fill in this missing functionality. First, we implement the mapping between structs and services via reflection, with the code placed in a separate file `service.go`.

[day3-service/service.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day3-service)

First, define the methodType struct:

```go
type methodType struct {
	method    reflect.Method
	ArgType   reflect.Type
	ReplyType reflect.Type
	numCalls  uint64
}

func (m *methodType) NumCalls() uint64 {
	return atomic.LoadUint64(&m.numCalls)
}

func (m *methodType) newArgv() reflect.Value {
	var argv reflect.Value
	// arg may be a pointer type, or a value type
	if m.ArgType.Kind() == reflect.Ptr {
		argv = reflect.New(m.ArgType.Elem())
	} else {
		argv = reflect.New(m.ArgType).Elem()
	}
	return argv
}

func (m *methodType) newReplyv() reflect.Value {
	// reply must be a pointer type
	replyv := reflect.New(m.ReplyType.Elem())
	switch m.ReplyType.Elem().Kind() {
	case reflect.Map:
		replyv.Elem().Set(reflect.MakeMap(m.ReplyType.Elem()))
	case reflect.Slice:
		replyv.Elem().Set(reflect.MakeSlice(m.ReplyType.Elem(), 0, 0))
	}
	return replyv
}
```

Each methodType instance holds the complete information of a method, including:

- method: the method itself
- ArgType: the type of the first argument
- ReplyType: the type of the second argument
- numCalls: used later to count how many times the method has been called

In addition, we implemented two methods, `newArgv` and `newReplyv`, to create instances of the corresponding types. There is a small detail in `newArgv`: pointer types and value types are instantiated in slightly different ways.

Second, define the service struct:

```go
type service struct {
	name   string
	typ    reflect.Type
	rcvr   reflect.Value
	method map[string]*methodType
}
```

The definition of service is also very concise: name is the name of the mapped struct, such as `T` or `WaitGroup`; typ is the type of the struct; rcvr is the struct instance itself — we keep rcvr because it is needed as the 0th argument when calling the method; method is a map that stores all eligible methods of the mapped struct.

Next, implement the constructor `newService`, whose argument is any struct instance to be mapped as a service.

```go
func newService(rcvr interface{}) *service {
	s := new(service)
	s.rcvr = reflect.ValueOf(rcvr)
	s.name = reflect.Indirect(s.rcvr).Type().Name()
	s.typ = reflect.TypeOf(rcvr)
	if !ast.IsExported(s.name) {
		log.Fatalf("rpc server: %s is not a valid service name", s.name)
	}
	s.registerMethods()
	return s
}

func (s *service) registerMethods() {
	s.method = make(map[string]*methodType)
	for i := 0; i < s.typ.NumMethod(); i++ {
		method := s.typ.Method(i)
		mType := method.Type
		if mType.NumIn() != 3 || mType.NumOut() != 1 {
			continue
		}
		if mType.Out(0) != reflect.TypeOf((*error)(nil)).Elem() {
			continue
		}
		argType, replyType := mType.In(1), mType.In(2)
		if !isExportedOrBuiltinType(argType) || !isExportedOrBuiltinType(replyType) {
			continue
		}
		s.method[method.Name] = &methodType{
			method:    method,
			ArgType:   argType,
			ReplyType: replyType,
		}
		log.Printf("rpc server: register %s.%s\n", s.name, method.Name)
	}
}

func isExportedOrBuiltinType(t reflect.Type) bool {
	return ast.IsExported(t.Name()) || t.PkgPath() == ""
}
```

`registerMethods` filters out the methods that meet the criteria:

- two arguments of exported or builtin types (three under reflection: the 0th is the receiver itself, similar to self in Python or this in Java)
- exactly one return value, of type error

Finally, we also need to implement the `call` method, which invokes a method through reflect values.

```go
func (s *service) call(m *methodType, argv, replyv reflect.Value) error {
	atomic.AddUint64(&m.numCalls, 1)
	f := m.method.Func
	returnValues := f.Call([]reflect.Value{s.rcvr, argv, replyv})
	if errInter := returnValues[0].Interface(); errInter != nil {
		return errInter.(error)
	}
	return nil
}
```

## Test Cases for service

To verify the correctness of the service implementation, we wrote a few test cases for service.go.

[day3-service/service_test.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day3-service)

Define the struct Foo with two methods: the exported method Sum and the unexported method sum.

```go
type Foo int

type Args struct{ Num1, Num2 int }

func (f Foo) Sum(args Args, reply *int) error {
	*reply = args.Num1 + args.Num2
	return nil
}

// it's not a exported Method
func (f Foo) sum(args Args, reply *int) error {
	*reply = args.Num1 + args.Num2
	return nil
}

func _assert(condition bool, msg string, v ...interface{}) {
	if !condition {
		panic(fmt.Sprintf("assertion failed: "+msg, v...))
	}
}
```

Test the newService and call methods.

```go
func TestNewService(t *testing.T) {
	var foo Foo
	s := newService(&foo)
	_assert(len(s.method) == 1, "wrong service Method, expect 1, but got %d", len(s.method))
	mType := s.method["Sum"]
	_assert(mType != nil, "wrong Method, Sum shouldn't nil")
}

func TestMethodType_Call(t *testing.T) {
	var foo Foo
	s := newService(&foo)
	mType := s.method["Sum"]

	argv := mType.newArgv()
	replyv := mType.newReplyv()
	argv.Set(reflect.ValueOf(Args{Num1: 1, Num2: 3}))
	err := s.call(mType, argv, replyv)
	_assert(err == nil && *replyv.Interface().(*int) == 4 && mType.NumCalls() == 1, "failed to call Foo.Sum")
}
```

## Integrating into the Server

Through reflection, structs are now mapped to services, but the request handling process is still incomplete. From receiving a request to replying, the following steps are missing: first, deserialize the request body according to the argument type; second, invoke `service.call` to complete the method call; third, serialize the reply into a byte stream, build the response message, and send it back.

Back to the code, we just need to complete the two TODO items left in `server.go`: `readRequest` and `handleRequest`.

Before that, we also need to implement a `Register` method for Server.

[day3-service/server.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day3-service)

```go
// Server represents an RPC Server.
type Server struct {
	serviceMap sync.Map
}

// Register publishes in the server the set of methods of the
func (server *Server) Register(rcvr interface{}) error {
	s := newService(rcvr)
	if _, dup := server.serviceMap.LoadOrStore(s.name, s); dup {
		return errors.New("rpc: service already defined: " + s.name)
	}
	return nil
}

// Register publishes the receiver's methods in the DefaultServer.
func Register(rcvr interface{}) error { return DefaultServer.Register(rcvr) }
```

Also implement the `findService` method, which looks up the corresponding service in serviceMap via `ServiceMethod`:

```go
func (server *Server) findService(serviceMethod string) (svc *service, mtype *methodType, err error) {
	dot := strings.LastIndex(serviceMethod, ".")
	if dot < 0 {
		err = errors.New("rpc server: service/method request ill-formed: " + serviceMethod)
		return
	}
	serviceName, methodName := serviceMethod[:dot], serviceMethod[dot+1:]
	svci, ok := server.serviceMap.Load(serviceName)
	if !ok {
		err = errors.New("rpc server: can't find service " + serviceName)
		return
	}
	svc = svci.(*service)
	mtype = svc.method[methodName]
	if mtype == nil {
		err = errors.New("rpc server: can't find method " + methodName)
	}
	return
}
```

The implementation of `findService` looks a bit verbose, but the logic is quite clear. Since ServiceMethod is composed as "Service.Method", we first split it into two parts: the first part is the name of the Service, and the second part is the method name. We first look up the corresponding service instance in serviceMap, then find the corresponding methodType in the service instance's method map.

With the groundwork in place, we first complete the readRequest method:

```go
// request stores all information of a call
type request struct {
	h            *codec.Header // header of request
	argv, replyv reflect.Value // argv and replyv of request
	mtype        *methodType
	svc          *service
}

func (server *Server) readRequest(cc codec.Codec) (*request, error) {
	h, err := server.readRequestHeader(cc)
	if err != nil {
		return nil, err
	}
	req := &request{h: h}
	req.svc, req.mtype, err = server.findService(h.ServiceMethod)
	if err != nil {
		return req, err
	}
	req.argv = req.mtype.newArgv()
	req.replyv = req.mtype.newReplyv()

	// make sure that argvi is a pointer, ReadBody need a pointer as parameter
	argvi := req.argv.Interface()
	if req.argv.Type().Kind() != reflect.Ptr {
		argvi = req.argv.Addr().Interface()
	}
	if err = cc.ReadBody(argvi); err != nil {
		log.Println("rpc server: read body err:", err)
		return req, err
	}
	return req, nil
}
```

The most important part of readRequest is creating the two argument instances via `newArgv()` and `newReplyv()`, and then deserializing the request body into the first argument argv via `cc.ReadBody()`. Here we again need to note that argv may be a value type or a pointer type, so the handling differs slightly.

Next, complete the handleRequest method:

```go
func (server *Server) handleRequest(cc codec.Codec, req *request, sending *sync.Mutex, wg *sync.WaitGroup) {
	defer wg.Done()
	err := req.svc.call(req.mtype, req.argv, req.replyv)
	if err != nil {
		req.h.Error = err.Error()
		server.sendResponse(cc, req.h, invalidRequest, sending)
		return
	}
	server.sendResponse(cc, req.h, req.replyv.Interface(), sending)
}
```

Compared with readRequest, handleRequest is very simple: it completes the method call via `req.svc.call` and passes replyv to sendResponse for serialization.

At this point, everything for today has been implemented: service registration and invocation are now supported on the server side.

## Demo

Finally, we still need to write an executable program (main) to verify today's results.

[day3-service/main/main.go](https://github.com/geektutu/7days-golang/tree/master/gee-rpc/day3-service)

First, define the struct Foo and the method Sum:

```go
package main

import (
	"geerpc"
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
```

Second, register Foo with the Server and start the RPC service:

```go
func startServer(addr chan string) {
	var foo Foo
	if err := geerpc.Register(&foo); err != nil {
		log.Fatal("register error:", err)
	}
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

Third, build the arguments, send the RPC request, and print the result.

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
			args := &Args{Num1: i, Num2: i * i}
			var reply int
			if err := client.Call("Foo.Sum", args, &reply); err != nil {
				log.Fatal("call Foo.Sum error:", err)
			}
			log.Printf("%d + %d = %d", args.Num1, args.Num2, reply)
		}(i)
	}
	wg.Wait()
}
```

The output is as follows:

```bash
rpc server: register Foo.Sum
start rpc server on [::]:57509
1 + 1 = 2
2 + 4 = 6
3 + 9 = 12
0 + 0 = 0
4 + 16 = 20
```

## Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [Go Written Test and Interview Questions](https://geektutu.com/post/qa-golang.html)
