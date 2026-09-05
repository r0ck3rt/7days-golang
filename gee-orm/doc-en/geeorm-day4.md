---
title: Implement an ORM Framework in Go - GeeORM Day 4: Chain Operations, Update and Delete
description: >-
  7 days to implement the ORM framework GeeORM in Go/golang from scratch tutorial (7 days implement golang object relational mapping framework from scratch
  tutorial), write an ORM framework by hand, modeled after gorm and xorm. Support stacking query conditions (where, order by, limit,
  etc.) through chained (chain) operations; implement updating (update), deleting (delete) and counting (count) records.
date: '2020-03-09 00:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geeorm/geeorm_sm.jpg
lang: en
---

This is the fourth article in the [7 Days Go ORM Framework Tutorial Series from scratch](https://geektutu.com/post/geeorm.html).

- Support stacking query conditions (where, order by, limit, etc.) through chained (chain) operations.
- Implement updating (update), deleting (delete) and counting (count) records. **About 100 lines of code**

## 1 Support Update, Delete and Count

### 1.1 Clause Generators

The clause package is responsible for building SQL statements. To add support for updating (update), deleting (delete) and counting (count), the first step is naturally to implement the generators for the update, delete and count clauses in clause.

Step 1: on top of the existing code, add three new enum values of type `Type`: UPDATE, DELETE and COUNT.

[day4-chain-operation/clause/clause.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day4-chain-operation/clause)

```go
// Support types for Clause
const (
	INSERT Type = iota
	VALUES
	SELECT
	LIMIT
	WHERE
	ORDERBY
	UPDATE
	DELETE
	COUNT
)
```

Step 2: implement the generators for the corresponding clauses and register them in the global variable `generators`.

[day4-chain-operation/clause/generator.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day4-chain-operation/clause)

```go
func init() {
	generators = make(map[Type]generator)
	generators[INSERT] = _insert
	generators[VALUES] = _values
	generators[SELECT] = _select
	generators[LIMIT] = _limit
	generators[WHERE] = _where
	generators[ORDERBY] = _orderBy
	generators[UPDATE] = _update
	generators[DELETE] = _delete
	generators[COUNT] = _count
}

func _update(values ...interface{}) (string, []interface{}) {
	tableName := values[0]
	m := values[1].(map[string]interface{})
	var keys []string
	var vars []interface{}
	for k, v := range m {
		keys = append(keys, k+" = ?")
		vars = append(vars, v)
	}
	return fmt.Sprintf("UPDATE %s SET %s", tableName, strings.Join(keys, ", ")), vars
}

func _delete(values ...interface{}) (string, []interface{}) {
	return fmt.Sprintf("DELETE FROM %s", values[0]), []interface{}{}
}

func _count(values ...interface{}) (string, []interface{}) {
	return _select(values[0], []string{"count(*)"})
}
```

- `_update` is designed to take two arguments: the first is the table name (table), and the second is a map representing the key-value pairs to be updated.
- `_delete` takes only one argument, the table name.
- `_count` takes only one argument, the table name, and reuses the `_select` generator.


### 1.2 The Update Method

The clause generators are ready. Next, just like the Insert and Find methods, we assemble the SQL statements in a certain order in `session/record.go` and execute them.

[day4-chain-operation/session/record.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day4-chain-operation/session)

```go
// support map[string]interface{}
// also support kv list: "Name", "Tom", "Age", 18, ....
func (s *Session) Update(kv ...interface{}) (int64, error) {
	m, ok := kv[0].(map[string]interface{})
	if !ok {
		m = make(map[string]interface{})
		for i := 0; i < len(kv); i += 2 {
			m[kv[i].(string)] = kv[i+1]
		}
	}
	s.clause.Set(clause.UPDATE, s.RefTable().Name, m)
	sql, vars := s.clause.Build(clause.UPDATE, clause.WHERE)
	result, err := s.Raw(sql, vars...).Exec()
	if err != nil {
		return 0, err
	}
	return result.RowsAffected()
}
```

What is special about the Update method is that it accepts two kinds of arguments: flattened key-value pairs and key-value pairs of map type. Since the generator expects key-value pairs of map type, the `Update` method dynamically checks the type of the incoming arguments and converts them automatically if they are not a map.


### 1.3 The Delete Method

```go
// Delete records with where clause
func (s *Session) Delete() (int64, error) {
	s.clause.Set(clause.DELETE, s.RefTable().Name)
	sql, vars := s.clause.Build(clause.DELETE, clause.WHERE)
	result, err := s.Raw(sql, vars...).Exec()
	if err != nil {
		return 0, err
	}
	return result.RowsAffected()
}
```

### 1.4 The Count Method

```go
// Count records with where clause
func (s *Session) Count() (int64, error) {
	s.clause.Set(clause.COUNT, s.RefTable().Name)
	sql, vars := s.clause.Build(clause.COUNT, clause.WHERE)
	row := s.Raw(sql, vars...).QueryRow()
	var tmp int64
	if err := row.Scan(&tmp); err != nil {
		return 0, err
	}
	return tmp, nil
}
```

## 2 Chained Calls (Chain)

Chained calls are a programming style that simplifies code, making it more concise and readable. The principle behind chained calls is also very simple: after an object calls a method, the method returns a reference/pointer to the object, so that other methods of the object can be called in turn. Generally speaking, when an object needs multiple method calls at once to configure its properties, it is a very good candidate for chained calls.

Building SQL statements fits this pattern perfectly. A SQL statement is composed of multiple clauses; a typical example is the SELECT statement, which often requires setting query conditions (WHERE), limiting the number of returned rows (LIMIT), and so on. The ideal way of calling it would be like this:

```go
s := geeorm.NewEngine("sqlite3", "gee.db").NewSession()
var users []User
s.Where("Age > 18").Limit(3).Find(&users)
```

From the example above, we can see that query condition statements such as `WHERE`, `LIMIT` and `ORDER BY` are perfect for chained calls. The generators for these clauses were implemented earlier, so next we just need to add the corresponding methods in `session/record.go`.

[day4-chain-operation/session/record.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day4-chain-operation/session)

```go
// Limit adds limit condition to clause
func (s *Session) Limit(num int) *Session {
	s.clause.Set(clause.LIMIT, num)
	return s
}

// Where adds limit condition to clause
func (s *Session) Where(desc string, args ...interface{}) *Session {
	var vars []interface{}
	s.clause.Set(clause.WHERE, append(append(vars, desc), args...)...)
	return s
}

// OrderBy adds order by condition to clause
func (s *Session) OrderBy(desc string) *Session {
	s.clause.Set(clause.ORDERBY, desc)
	return s
}
```

## 3 First: Returning a Single Record

Quite often, we expect a SQL statement to return only one record; for example, looking up a student's information by their student ID should return one and only one result. Combined with chained calls, we can implement the First method very easily.

```go
func (s *Session) First(value interface{}) error {
	dest := reflect.Indirect(reflect.ValueOf(value))
	destSlice := reflect.New(reflect.SliceOf(dest.Type())).Elem()
	if err := s.Limit(1).Find(destSlice.Addr().Interface()); err != nil {
		return err
	}
	if destSlice.Len() == 0 {
		return errors.New("NOT FOUND")
	}
	dest.Set(destSlice.Index(0))
	return nil
}
```

The First method can be used like this:

```go
u := &User{}
_ = s.OrderBy("Age DESC").First(u)
```

> How it works: based on the passed-in type, use reflection to construct a slice, call `Limit(1)` to limit the number of returned rows, and call the `Find` method to get the query results.

## 4 Testing

Next, let's add a few test cases in `record_test.go` to check that the features work correctly.

```go
package session

import "testing"

var (
	user1 = &User{"Tom", 18}
	user2 = &User{"Sam", 25}
	user3 = &User{"Jack", 25}
)

func testRecordInit(t *testing.T) *Session {
	t.Helper()
	s := NewSession().Model(&User{})
	err1 := s.DropTable()
	err2 := s.CreateTable()
	_, err3 := s.Insert(user1, user2)
	if err1 != nil || err2 != nil || err3 != nil {
		t.Fatal("failed init test records")
	}
	return s
}

func TestSession_Limit(t *testing.T) {
	s := testRecordInit(t)
	var users []User
	err := s.Limit(1).Find(&users)
	if err != nil || len(users) != 1 {
		t.Fatal("failed to query with limit condition")
	}
}

func TestSession_Update(t *testing.T) {
	s := testRecordInit(t)
	affected, _ := s.Where("Name = ?", "Tom").Update("Age", 30)
	u := &User{}
	_ = s.OrderBy("Age DESC").First(u)

	if affected != 1 || u.Age != 30 {
		t.Fatal("failed to update")
	}
}

func TestSession_DeleteAndCount(t *testing.T) {
	s := testRecordInit(t)
	affected, _ := s.Where("Name = ?", "Tom").Delete()
	count, _ := s.Count()

	if affected != 1 || count != 1 {
		t.Fatal("failed to delete or count")
	}
}
```

## Further Reading

- [A Concise Tutorial for the Go Language](https://geektutu.com/post/quick-golang.html)
- [A Concise Tutorial for Go Test Unit Testing](https://geektutu.com/post/quick-go-test.html)
- [SQLite Common Commands Cheat Sheet](https://geektutu.com/post/cheat-sheet-sqlite.html)
