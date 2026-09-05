---
title: Implement an ORM Framework in Go - GeeORM Day 3: Record Operations
description: >-
  7 days to implement the ORM framework GeeORM in Go/golang from scratch tutorial (7 days implement golang object relational mapping framework from scratch
  tutorial), write an ORM framework by hand, modeled after gorm and xorm. Implement inserting (insert) records; use reflection (reflect) to convert database records into the corresponding struct instances, implementing the query (select) feature.
date: '2020-03-08 09:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geeorm/geeorm_sm.jpg
lang: en
---

This is the third article in the [7 Days Go ORM Framework Tutorial Series from scratch](https://geektutu.com/post/geeorm.html).

- Implement the feature of inserting (insert) records.
- Use reflection (reflect) to convert database records into the corresponding struct instances, implementing the query (select) feature. **About 150 lines of code**

## 1 Building SQL Statements with Clause

Starting from day 3, GeeORM needs to handle some more complex operations, such as queries. A query statement is generally composed of many clauses. A SELECT statement is usually structured like this:

```sql
SELECT col1, col2, ...
    FROM table_name
    WHERE [ conditions ]
    GROUP BY col1
    HAVING [ conditions ]
```

In other words, building a complete SQL statement in one go is rather difficult, so we extract the SQL-building part into a separate sub-package, `clause`.

First, implement the generation rules for each clause in `clause/generator.go`.

[day3-save-query/clause/generator.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day3-save-query/clause)


```go
package clause

import (
	"fmt"
	"strings"
)

type generator func(values ...interface{}) (string, []interface{})

var generators map[Type]generator

func init() {
	generators = make(map[Type]generator)
	generators[INSERT] = _insert
	generators[VALUES] = _values
	generators[SELECT] = _select
	generators[LIMIT] = _limit
	generators[WHERE] = _where
	generators[ORDERBY] = _orderBy
}

func genBindVars(num int) string {
	var vars []string
	for i := 0; i < num; i++ {
		vars = append(vars, "?")
	}
	return strings.Join(vars, ", ")
}

func _insert(values ...interface{}) (string, []interface{}) {
	// INSERT INTO $tableName ($fields)
	tableName := values[0]
	fields := strings.Join(values[1].([]string), ",")
	return fmt.Sprintf("INSERT INTO %s (%v)", tableName, fields), []interface{}{}
}

func _values(values ...interface{}) (string, []interface{}) {
	// VALUES ($v1), ($v2), ...
	var bindStr string
	var sql strings.Builder
	var vars []interface{}
	sql.WriteString("VALUES ")
	for i, value := range values {
		v := value.([]interface{})
		if bindStr == "" {
			bindStr = genBindVars(len(v))
		}
		sql.WriteString(fmt.Sprintf("(%v)", bindStr))
		if i+1 != len(values) {
			sql.WriteString(", ")
		}
		vars = append(vars, v...)
	}
	return sql.String(), vars

}

func _select(values ...interface{}) (string, []interface{}) {
	// SELECT $fields FROM $tableName
	tableName := values[0]
	fields := strings.Join(values[1].([]string), ",")
	return fmt.Sprintf("SELECT %v FROM %s", fields, tableName), []interface{}{}
}

func _limit(values ...interface{}) (string, []interface{}) {
	// LIMIT $num
	return "LIMIT ?", values
}

func _where(values ...interface{}) (string, []interface{}) {
	// WHERE $desc
	desc, vars := values[0], values[1:]
	return fmt.Sprintf("WHERE %s", desc), vars
}

func _orderBy(values ...interface{}) (string, []interface{}) {
	return fmt.Sprintf("ORDER BY %s", values[0]), []interface{}{}
}
```

Then, implement the `Clause` struct in `clause/clause.go` to assemble the individual clauses.

[day3-save-query/clause/clause.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day3-save-query/clause)

```go
package clause

import "strings"

type Clause struct {
	sql     map[Type]string
	sqlVars map[Type][]interface{}
}

type Type int
const (
	INSERT Type = iota
	VALUES
	SELECT
	LIMIT
	WHERE
	ORDERBY
)

func (c *Clause) Set(name Type, vars ...interface{}) {
	if c.sql == nil {
		c.sql = make(map[Type]string)
		c.sqlVars = make(map[Type][]interface{})
	}
	sql, vars := generators[name](vars...)
	c.sql[name] = sql
	c.sqlVars[name] = vars
}

func (c *Clause) Build(orders ...Type) (string, []interface{}) {
	var sqls []string
	var vars []interface{}
	for _, order := range orders {
		if sql, ok := c.sql[order]; ok {
			sqls = append(sqls, sql)
			vars = append(vars, c.sqlVars[order]...)
		}
	}
	return strings.Join(sqls, " "), vars
}
```

- The `Set` method calls the corresponding generator based on `Type`, generating the SQL statement for that clause.
- The `Build` method constructs the final SQL statement according to the order of the passed-in `Type` values.

Implement the corresponding test cases in `clause_test.go`:

```go
func testSelect(t *testing.T) {
	var clause Clause
	clause.Set(LIMIT, 3)
	clause.Set(SELECT, "User", []string{"*"})
	clause.Set(WHERE, "Name = ?", "Tom")
	clause.Set(ORDERBY, "Age ASC")
	sql, vars := clause.Build(SELECT, WHERE, ORDERBY, LIMIT)
	t.Log(sql, vars)
	if sql != "SELECT * FROM User WHERE Name = ? ORDER BY Age ASC LIMIT ?" {
		t.Fatal("failed to build SQL")
	}
	if !reflect.DeepEqual(vars, []interface{}{"Tom", 3}) {
		t.Fatal("failed to build SQLVars")
	}
}

func TestClause_Build(t *testing.T) {
	t.Run("select", func(t *testing.T) {
		testSelect(t)
	})
}
```

## 2 Implementing Insert

First, add a `clause` member variable to Session:

```go
// session/raw.go
type Session struct {
	db       *sql.DB
	dialect  dialect.Dialect
	refTable *schema.Schema
	clause   clause.Clause
	sql      strings.Builder
	sqlVars  []interface{}
}

func (s *Session) Clear() {
	s.sql.Reset()
	s.sqlVars = nil
	s.clause = clause.Clause{}
}
```

The clause already supports generating simple INSERT and SELECT SQL statements, so next we can implement the corresponding features in session.

The SQL statement for INSERT generally looks like this:

```sql
INSERT INTO table_name(col1, col2, col3, ...) VALUES
    (A1, A2, A3, ...),
    (B1, B2, B3, ...),
    ...
```

In an ORM framework, the expected way of calling Insert is as follows:

```go
s := geeorm.NewEngine("sqlite3", "gee.db").NewSession()
u1 := &User{Name: "Tom", Age: 18}
u2 := &User{Name: "Sam", Age: 25}
s.Insert(u1, u2, ...)
```

That is to say, we still need one more step: find the corresponding values from the object according to the order of the columns in the database and flatten them in order, i.e. converting `u1` and `u2` into a format like `("Tom", 18), ("Same", 25)`.

Therefore, before implementing the Insert feature, we need to add a function `RecordValues` to `Schema` to perform the conversion above.

[day3-save-query/schema/schema.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day3-save-query/schema)

```go
func (schema *Schema) RecordValues(dest interface{}) []interface{} {
	destValue := reflect.Indirect(reflect.ValueOf(dest))
	var fieldValues []interface{}
	for _, field := range schema.Fields {
		fieldValues = append(fieldValues, destValue.FieldByName(field.Name).Interface())
	}
	return fieldValues
}
```

Create a new file record.go in the session folder to implement the code related to creating, deleting, querying and updating records.

[day3-save-query/session/record.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day3-save-query/session)

```go
package session

import (
	"geeorm/clause"
	"reflect"
)

func (s *Session) Insert(values ...interface{}) (int64, error) {
	recordValues := make([]interface{}, 0)
	for _, value := range values {
		table := s.Model(value).RefTable()
		s.clause.Set(clause.INSERT, table.Name, table.FieldNames)
		recordValues = append(recordValues, table.RecordValues(value))
	}

	s.clause.Set(clause.VALUES, recordValues...)
	sql, vars := s.clause.Build(clause.INSERT, clause.VALUES)
	result, err := s.Raw(sql, vars...).Exec()
	if err != nil {
		return 0, err
	}

	return result.RowsAffected()
}
```

From now on, all SQL statements will be constructed in the same way as in `Insert`, in two steps:

- 1) Call `clause.Set()` multiple times to build each clause.
- 2) Call `clause.Build()` once to construct the final SQL statement in the given order.

After construction, call the `Raw().Exec()` method to execute it.

## 3 Implementing Find

The expected usage is to pass in a pointer to a slice, and the query results are stored in the slice.

```go
s := geeorm.NewEngine("sqlite3", "gee.db").NewSession()
var users []User
s.Find(&users);
```

The difficulty of Find is the exact opposite of Insert. Insert needs to flatten the values of every field of an existing object, while Find needs to construct objects from the flattened field values. Again, reflection (reflect) is needed.

```go
func (s *Session) Find(values interface{}) error {
	destSlice := reflect.Indirect(reflect.ValueOf(values))
	destType := destSlice.Type().Elem()
	table := s.Model(reflect.New(destType).Elem().Interface()).RefTable()

	s.clause.Set(clause.SELECT, table.Name, table.FieldNames)
	sql, vars := s.clause.Build(clause.SELECT, clause.WHERE, clause.ORDERBY, clause.LIMIT)
	rows, err := s.Raw(sql, vars...).QueryRows()
	if err != nil {
		return err
	}

	for rows.Next() {
		dest := reflect.New(destType).Elem()
		var values []interface{}
		for _, name := range table.FieldNames {
			values = append(values, dest.FieldByName(name).Addr().Interface())
		}
		if err := rows.Scan(values...); err != nil {
			return err
		}
		destSlice.Set(reflect.Append(destSlice, dest))
	}
	return rows.Close()
}
```

The implementation of Find is rather complex and mainly consists of the following steps:

- 1) `destSlice.Type().Elem()` gets the type of a single element of the slice `destType`; use the `reflect.New()` method to create an instance of `destType` as the argument of `Model()`, mapping out the table structure `RefTable()`.
- 2) Based on the table structure, use clause to build a SELECT statement and query all matching records `rows`.
- 3) Iterate over each row of records, use reflection to create an instance `dest` of `destType`, flatten all fields of `dest` and build the slice `values`.
- 4) Call `rows.Scan()` to assign the value of each column of the row to each field in values in turn.
- 5) Append `dest` to the slice `destSlice`. Loop until all records have been appended to the slice `destSlice`.

## 4 Testing

Create a new `record_test.go` in the session folder with the test cases.

> The definitions of `User` and `NewSession()` are located in raw_test.go.

[day3-save-query/session/record_test.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day3-save-query/session)

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

func TestSession_Insert(t *testing.T) {
	s := testRecordInit(t)
	affected, err := s.Insert(user3)
	if err != nil || affected != 1 {
		t.Fatal("failed to create record")
	}
}

func TestSession_Find(t *testing.T) {
	s := testRecordInit(t)
	var users []User
	if err := s.Find(&users); err != nil || len(users) != 2 {
		t.Fatal("failed to query all")
	}
}
```

## Further Reading

- [A Concise Tutorial for the Go Language](https://geektutu.com/post/quick-golang.html)
- [A Concise Tutorial for Go Test Unit Testing](https://geektutu.com/post/quick-go-test.html)
- [SQLite Common Commands Cheat Sheet](https://geektutu.com/post/cheat-sheet-sqlite.html)
- [Laws Of Reflection - golang.org](https://blog.golang.org/laws-of-reflection)
