---
title: Implement an ORM Framework in Go - GeeORM Day 6: Transactions
description: >-
  A 7-day tutorial on implementing an ORM framework GeeORM in Go/golang from scratch (7 days implement golang object relational mapping framework from scratch
  tutorial). Build an ORM framework modeled after the implementations of gorm and xorm. Introduces transactions in databases; wraps transactions, using
  user-defined callback functions to implement atomic operations.
date: '2020-03-09 05:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geeorm/geeorm_sm.jpg
lang: en
---

This article is part of the [7 Days Go ORM Framework Tutorial Series from scratch](https://geektutu.com/post/geeorm.html).

- Introduces transactions in databases.
- Wraps transactions, using user-defined callback functions to implement atomic operations. **About 100 lines of code**

## ACID Properties of Transactions

> A database transaction is a sequence of database operations that accesses and may modify various data items. These operations are either all executed or none executed, forming an indivisible unit of work. A transaction consists of all the database operations executed between the start and the end of the transaction.

Take a simple example: a money transfer. When A transfers 10,000 yuan to B, the database needs to execute at least 2 operations:

- 1) Deduct 10,000 yuan from A's account.
- 2) Add 10,000 yuan to B's account.

These two operations must either both be executed, meaning the transfer succeeded. If any one of them fails, all previous operations must be rolled back, meaning the transfer failed. One operation completing while the other fails is an unacceptable outcome. This scenario is a perfect fit for the features of database transactions.

If a database supports transactions, it must possess the four ACID properties.

- 1) Atomicity: All operations in a transaction are indivisible in the database; either all of them are completed, or none of them are executed.
- 2) Consistency: The result of several transactions executing in parallel must be consistent with the result of executing them serially in some order.
- 3) Isolation: The execution of a transaction is not interfered with by other transactions, and the intermediate results of a transaction's execution must be invisible to other transactions.
- 4) Durability: For any committed transaction, the system must guarantee that the changes made by the transaction to the database are not lost, even in the event of a database failure.

## Transactions in SQLite and the Go Standard Library

What does the native SQL for creating a transaction in SQLite look like?

```sql
sqlite> BEGIN;
sqlite> DELETE FROM User WHERE Age > 25;
sqlite> INSERT INTO User VALUES ("Tom", 25), ("Jack", 18);
sqlite> COMMIT;
```

`BEGIN` starts a transaction, `COMMIT` commits a transaction, and `ROLLBACK` rolls back a transaction. Every transaction starts with `BEGIN` and ends with `COMMIT` or `ROLLBACK`.

The Go standard library's database/sql package provides interfaces that support transactions. Let's look at a simple example to see how the Go standard library supports transactions.

```go
package main

import (
	"database/sql"
	_ "github.com/mattn/go-sqlite3"
	"log"
)

func main() {
	db, _ := sql.Open("sqlite3", "gee.db")
	defer func() { _ = db.Close() }()
	_, _ = db.Exec("CREATE TABLE IF NOT EXISTS User(`Name` text);")

	tx, _ := db.Begin()
	_, err1 := tx.Exec("INSERT INTO User(`Name`) VALUES (?)", "Tom")
	_, err2 := tx.Exec("INSERT INTO User(`Name`) VALUES (?)", "Jack")
	if err1 != nil || err2 != nil {
		_ = tx.Rollback()
		log.Println("Rollback", err1, err2)
	} else {
		_ = tx.Commit()
		log.Println("Commit")
	}
}
```

Implementing transactions in Go is actually very close to the native SQL statements. Call `db.Begin()` to obtain a `*sql.Tx` object, use `tx.Exec()` to execute a series of operations, roll back via `tx.Rollback()` if an error occurs, and commit via `tx.Commit()` if no error occurs.

## Transaction Support in GeeORM

Previously, every operation in GeeORM was automatically committed as soon as it finished executing, and each operation was independent of the others. Previously, SQL statements were executed directly using the `sql.DB` object; to support transactions, execution needs to be changed to `sql.Tx`. Add a new field `tx *sql.Tx` to the Session struct: when `tx` is not nil, SQL statements are executed using `tx`; otherwise, they are executed using `db`. This preserves the original execution method while adding support for transactions.

[day6-transaction/session/raw.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day6-transaction/session)

```go
type Session struct {
	db       *sql.DB
	dialect  dialect.Dialect
	tx       *sql.Tx
	refTable *schema.Schema
	clause   clause.Clause
	sql      strings.Builder
	sqlVars  []interface{}
}

// CommonDB is a minimal function set of db
type CommonDB interface {
	Query(query string, args ...interface{}) (*sql.Rows, error)
	QueryRow(query string, args ...interface{}) *sql.Row
	Exec(query string, args ...interface{}) (sql.Result, error)
}

var _ CommonDB = (*sql.DB)(nil)
var _ CommonDB = (*sql.Tx)(nil)

// DB returns tx if a tx begins. otherwise return *sql.DB
func (s *Session) DB() CommonDB {
	if s.tx != nil {
		return s.tx
	}
	return s.db
}
```

Create a new file `session/transaction.go` that wraps the three transaction interfaces: Begin, Commit, and Rollback.

[day6-transaction/session/transaction.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day6-transaction/session)

```go
package session

import "geeorm/log"

func (s *Session) Begin() (err error) {
	log.Info("transaction begin")
	if s.tx, err = s.db.Begin(); err != nil {
		log.Error(err)
		return
	}
	return
}

func (s *Session) Commit() (err error) {
	log.Info("transaction commit")
	if err = s.tx.Commit(); err != nil {
		log.Error(err)
	}
	return
}

func (s *Session) Rollback() (err error) {
	log.Info("transaction rollback")
	if err = s.tx.Rollback(); err != nil {
		log.Error(err)
	}
	return
}
```

- Call `s.db.Begin()` to obtain a `*sql.Tx` object and assign it to s.tx.
- Another purpose of the wrapping is to print logs uniformly, which makes it easier to locate problems.


The final step is to provide users with a foolproof, one-step interface in `geeorm.go`.

[day6-transaction/geeorm.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day6-transaction)

```go
type TxFunc func(*session.Session) (interface{}, error)

func (engine *Engine) Transaction(f TxFunc) (result interface{}, err error) {
	s := engine.NewSession()
	if err := s.Begin(); err != nil {
		return nil, err
	}
	defer func() {
		if p := recover(); p != nil {
			_ = s.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			_ = s.Rollback() // err is non-nil; don't change it
		} else {
			err = s.Commit() // err is nil; if Commit returns error update err
		}
	}()

	return f(s)
}
```

> The implementation of Transaction was inspired by [stackoverflow](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback)

Users only need to put all operations into a callback function and pass it as an argument to `engine.Transaction()`. If any error occurs, the transaction is rolled back automatically; if no error occurs, it is committed.

## Testing

Let's add test cases in `geeorm_test.go` to see how Transaction works.

```go
func OpenDB(t *testing.T) *Engine {
	t.Helper()
	engine, err := NewEngine("sqlite3", "gee.db")
	if err != nil {
		t.Fatal("failed to connect", err)
	}
	return engine
}

type User struct {
	Name string `geeorm:"PRIMARY KEY"`
	Age  int
}

func TestEngine_Transaction(t *testing.T) {
	t.Run("rollback", func(t *testing.T) {
		transactionRollback(t)
	})
	t.Run("commit", func(t *testing.T) {
		transactionCommit(t)
	})
}
```

First, the rollback case:

```go
func transactionRollback(t *testing.T) {
	engine := OpenDB(t)
	defer engine.Close()
	s := engine.NewSession()
	_ = s.Model(&User{}).DropTable()
	_, err := engine.Transaction(func(s *session.Session) (result interface{}, err error) {
		_ = s.Model(&User{}).CreateTable()
		_, err = s.Insert(&User{"Tom", 18})
		return nil, errors.New("Error")
	})
	if err == nil || s.HasTable() {
		t.Fatal("failed to rollback")
	}
}
```

- In this test case, if the execution succeeds, a table `User` will be created and a record will be inserted.
- A custom error is deliberately returned, so the transaction is ultimately rolled back and the table creation fails.

Next, the commit case:

```go
func transactionCommit(t *testing.T) {
	engine := OpenDB(t)
	defer engine.Close()
	s := engine.NewSession()
	_ = s.Model(&User{}).DropTable()
	_, err := engine.Transaction(func(s *session.Session) (result interface{}, err error) {
		_ = s.Model(&User{}).CreateTable()
		_, err = s.Insert(&User{"Tom", 18})
		return
	})
	u := &User{}
	_ = s.First(u)
	if err != nil || u.Name != "Tom" {
		t.Fatal("failed to commit")
	}
}
```

- Creating the table and inserting the record both execute successfully, and the inserted record is finally retrieved via the `s.First()` method.

## Appendix: Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [A Concise Go Test Unit Testing Tutorial](https://geektutu.com/post/quick-go-test.html)
- [SQLite Common Commands Cheat Sheet](https://geektutu.com/post/cheat-sheet-sqlite.html)
