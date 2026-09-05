---
title: Implement an ORM Framework in Go - GeeORM Day 5 Implementing Hooks
description: >-
  7 days to implement the ORM framework GeeORM in Go/golang from scratch tutorial (7 days implement golang object relational mapping framework from scratch
  tutorial), write an ORM framework by hand, modeled after gorm and xorm. Use reflection (reflect) to obtain the hooks (hooks) bound to a struct and invoke them; support calling hooks before and after CRUD (create, read, update, delete) operations.
date: '2020-03-09 02:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geeorm/geeorm_sm.jpg
lang: en
---

This is the fifth article in the [7 Days Go ORM Framework Tutorial Series from scratch](https://geektutu.com/post/geeorm.html).

- Use reflection (reflect) to obtain the hooks (hooks) bound to a struct and invoke them.
- Support calling hooks before and after CRUD (create, read, update, delete) operations. **About 50 lines of code**

## 1 The Hook Mechanism

A hook's main idea is to pre-plant (predefine) a hook in places where functionality might be added; when we need to modify or extend the logic at that place, we simply mount the extended class or method onto that point. Hooks are widely used. For example, the Travis continuous integration service supported by GitHub: when a `git push` event occurs, Travis is triggered to pull the new code and build it. Hooks are also very common in IDEs; for example, pressing `Ctrl + s` automatically formats the code. Another example is the `hot reload` mechanism commonly used in front-end development: when the front-end code changes, it is automatically compiled and packaged, and the browser is notified to refresh the page automatically, achieving what-you-write-is-what-you-get.

The quality of a hook mechanism depends on whether the extension points are well chosen. For continuous integration, for example, repeatedly rebuilding is meaningless if the code has not changed, so hooks should be designed where the code is likely to change, such as before and after merging MRs or PRs.

So for an ORM framework, where are the suitable extension points? Obviously, before and after creating, reading, updating and deleting records are all very good choices.

For example, suppose we design an `Account` class that contains a private field `Password`; after every query, the value needs to be masked before it can be used further. If an `AfterQuery` hook is provided, which automatically masks the value of the `Password` field after a query, wouldn't that save a lot of redundant code?

## 2 Implementing Hooks

Hooks in GeeORM are bound to structs, that is, each struct needs to implement its own hooks. The hook-related code is implemented in `session/hooks.go`.

[day5-hooks/session/hooks.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day5-hooks/session)

```go
package session

import (
	"geeorm/log"
	"reflect"
)

// Hooks constants
const (
	BeforeQuery  = "BeforeQuery"
	AfterQuery   = "AfterQuery"
	BeforeUpdate = "BeforeUpdate"
	AfterUpdate  = "AfterUpdate"
	BeforeDelete = "BeforeDelete"
	AfterDelete  = "AfterDelete"
	BeforeInsert = "BeforeInsert"
	AfterInsert  = "AfterInsert"
)

// CallMethod calls the registered hooks
func (s *Session) CallMethod(method string, value interface{}) {
	fm := reflect.ValueOf(s.RefTable().Model).MethodByName(method)
	if value != nil {
		fm = reflect.ValueOf(value).MethodByName(method)
	}
	param := []reflect.Value{reflect.ValueOf(s)}
	if fm.IsValid() {
		if v := fm.Call(param); len(v) > 0 {
			if err, ok := v[0].Interface().(error); ok {
				log.Error(err)
			}
		}
	}
	return
}
```

- The hook mechanism is also implemented through reflection. `s.RefTable().Model` or `value` is the object the current session is operating on; use the `MethodByName` method to reflectively obtain that object's method.
- It is called with `s *Session` as the argument. Every hook takes a `*Session` as its argument.

Next, we just need to call the `CallMethod()` method inside the Find, Insert, Update and Delete methods. For example, the `Find` method becomes:

```go
// Find gets all eligible records
func (s *Session) Find(values interface{}) error {
	s.CallMethod(BeforeQuery, nil)
    // ...
    for rows.Next() {
        dest := reflect.New(destType).Elem()
        // ...
        s.CallMethod(AfterQuery, dest.Addr().Interface())
        // ...
	}
	return rows.Close()
}
```

- The `AfterQuery` hook can operate on each record of the result.

## 3 Testing

Create the `session/hooks.go` file and add the corresponding test cases.

```go
package session

import (
	"geeorm/log"
	"testing"
)

type Account struct {
	ID       int `geeorm:"PRIMARY KEY"`
	Password string
}

func (account *Account) BeforeInsert(s *Session) error {
	log.Info("before inert", account)
	account.ID += 1000
	return nil
}

func (account *Account) AfterQuery(s *Session) error {
	log.Info("after query", account)
	account.Password = "******"
	return nil
}

func TestSession_CallMethod(t *testing.T) {
	s := NewSession().Model(&Account{})
	_ = s.DropTable()
	_ = s.CreateTable()
	_, _ = s.Insert(&Account{1, "123456"}, &Account{2, "qwerty"})

	u := &Account{}

	err := s.First(u)
	if err != nil || u.ID != 1001 || u.Password != "******" {
		t.Fatal("Failed to call hooks after query, got", u)
	}
}
```

In this test case, two hooks are tested: `BeforeInsert` and `AfterQuery`.

- `BeforeInsert` increases the value of account.ID by 1000.
- `AfterQuery` masks the password, displaying it as six `*` characters.

## Further Reading

- [A Concise Tutorial for the Go Language](https://geektutu.com/post/quick-golang.html)
- [A Concise Tutorial for Go Test Unit Testing](https://geektutu.com/post/quick-go-test.html)
- [SQLite Common Commands Cheat Sheet](https://geektutu.com/post/cheat-sheet-sqlite.html)
