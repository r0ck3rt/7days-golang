---
title: Implement an ORM Framework in Go - GeeORM Day 2: Mapping Structs to Tables
description: >-
  A tutorial on implementing the ORM framework GeeORM in Go/golang from scratch in 7 days (7 days implement golang object relational
  mapping framework from scratch tutorial). Build an ORM framework by hand, modeled after gorm and xorm. This article uses reflection
  (reflect) to obtain the name and fields of an arbitrary struct object and map it to a table in the database; uses dialect to
  isolate the differences between databases for easy extension; and creates and drops database tables.
date: '2020-03-08 08:20:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geeorm/geeorm_sm.jpg
lang: en
---

This article is the second part of the [7 Days Go ORM Framework Tutorial Series from scratch](https://geektutu.com/post/geeorm.html).

- Use dialect to isolate the differences between databases, making the framework easy to extend.
- Use reflection (reflect) to obtain the name and fields of an arbitrary struct object and map it to a table in the database.
- Create and drop database tables. **About 150 lines of code**

## 1 Dialect

The types in SQL statements are different from those in Go. For example, Go's `int`, `int8`, `int16` and other types all correspond to the `integer` type in SQLite. Therefore, the first step in implementing ORM mapping is to think about how to map Go types to database types.

Meanwhile, the data types supported by different databases also differ; even for the same feature, the SQL expression may vary. An ORM framework often needs to be compatible with multiple databases, so we need to extract the varying parts and implement them separately for each database, achieving maximum reuse and decoupling. This part of the code is called `dialect`.

Create a new folder dialect in the root directory, and create the file `dialect.go` inside the dialect folder to abstract away the differences between databases.

[day2-reflect-schema/dialect/dialect.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day2-reflect-schema/dialect)

```go
package dialect

import "reflect"

var dialectsMap = map[string]Dialect{}

type Dialect interface {
	DataTypeOf(typ reflect.Value) string
	TableExistSQL(tableName string) (string, []interface{})
}

func RegisterDialect(name string, dialect Dialect) {
	dialectsMap[name] = dialect
}

func GetDialect(name string) (dialect Dialect, ok bool) {
	dialect, ok = dialectsMap[name]
	return
}
```

The `Dialect` interface contains two methods:

- `DataTypeOf` converts a Go type into the data type of the target database.
- `TableExistSQL` returns the SQL statement for checking whether a table exists; the argument is the table name.

Of course, the differences between databases go far beyond these two places. As the ORM framework gains more features, the dialect implementation will gradually grow richer, while the other parts of the framework remain unaffected.

It also declares the two methods `RegisterDialect` and `GetDialect` for registering and retrieving dialect instances. To add support for a new database, just call `RegisterDialect` to register it globally.

Next, create the file `sqlite3.go` in the `dialect` directory to add support for SQLite.

[day2-reflect-schema/dialect/sqlite3.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day2-reflect-schema/dialect)

```go
package dialect

import (
	"fmt"
	"reflect"
	"time"
)

type sqlite3 struct{}

var _ Dialect = (*sqlite3)(nil)

func init() {
	RegisterDialect("sqlite3", &sqlite3{})
}

func (s *sqlite3) DataTypeOf(typ reflect.Value) string {
	switch typ.Kind() {
	case reflect.Bool:
		return "bool"
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32,
		reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uintptr:
		return "integer"
	case reflect.Int64, reflect.Uint64:
		return "bigint"
	case reflect.Float32, reflect.Float64:
		return "real"
	case reflect.String:
		return "text"
	case reflect.Array, reflect.Slice:
		return "blob"
	case reflect.Struct:
		if _, ok := typ.Interface().(time.Time); ok {
			return "datetime"
		}
	}
	panic(fmt.Sprintf("invalid sql type %s (%s)", typ.Type().Name(), typ.Kind()))
}

func (s *sqlite3) TableExistSQL(tableName string) (string, []interface{}) {
	args := []interface{}{tableName}
	return "SELECT name FROM sqlite_master WHERE type='table' and name = ?", args
}
```

- Although the implementation of `sqlite3.go` is somewhat tedious, the overall logic is very clear. `DataTypeOf` maps Go types to SQLite data types. `TableExistSQL` returns the SQL statement for checking whether the table `tableName` exists in SQLite.
- The `init()` function is implemented so that the sqlite3 dialect is automatically registered globally when the package is loaded for the first time.

## 2 Schema

Dialect handles the conversion of some specific SQL statements. Next, we are going to implement the most core conversion in the ORM framework — the conversion between objects and tables. Given an arbitrary object, convert it into a table schema in a relational database.

What elements are needed to create a table in a database?

- Table name — struct name
- Field names and field types — member variables and their types.
- Extra constraints (e.g., not null, primary key, etc.) — the member variables' tags (implemented via tags in Go, and via annotations in languages such as Java and Python).

Here is a concrete example:

```go
type User struct {
    Name string `geeorm:"PRIMARY KEY"`
    Age  int
}
```

The expected schema statement:

```sql
CREATE TABLE `User` (`Name` text PRIMARY KEY, `Age` integer);
```

We put the implementation of this part in a sub-package `schema/schema.go`.

[day2-reflect-schema/schema/schema.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day2-reflect-schema/schema)

```go
package schema

import (
	"geeorm/dialect"
	"go/ast"
	"reflect"
)

// Field represents a column of database
type Field struct {
	Name string
	Type string
	Tag  string
}

// Schema represents a table of database
type Schema struct {
	Model      interface{}
	Name       string
	Fields     []*Field
	FieldNames []string
	fieldMap   map[string]*Field
}

func (schema *Schema) GetField(name string) *Field {
	return schema.fieldMap[name]
}
```

- Field contains three member variables: the field name Name, the type Type, and the constraint Tag.
- Schema mainly contains the object being mapped Model, the table name Name, and the fields Fields.
- FieldNames contains all the field names (column names), and fieldMap records the mapping from field names to Fields, so that they can be used directly later without traversing Fields.

Next, implement the Parse function, which parses an arbitrary object into a Schema instance.

```go
func Parse(dest interface{}, d dialect.Dialect) *Schema {
	modelType := reflect.Indirect(reflect.ValueOf(dest)).Type()
	schema := &Schema{
		Model:    dest,
		Name:     modelType.Name(),
		fieldMap: make(map[string]*Field),
	}

	for i := 0; i < modelType.NumField(); i++ {
		p := modelType.Field(i)
		if !p.Anonymous && ast.IsExported(p.Name) {
			field := &Field{
				Name: p.Name,
				Type: d.DataTypeOf(reflect.Indirect(reflect.New(p.Type))),
			}
			if v, ok := p.Tag.Lookup("geeorm"); ok {
				field.Tag = v
			}
			schema.Fields = append(schema.Fields, field)
			schema.FieldNames = append(schema.FieldNames, p.Name)
			schema.fieldMap[p.Name] = field
		}
	}
	return schema
}
```

- `TypeOf()` and `ValueOf()` are the most fundamental and most important methods in the reflect package, returning the type and value of the argument respectively. Since the argument is designed to be a pointer to an object, `reflect.Indirect()` is needed to obtain the instance the pointer points to.
- `modelType.Name()` obtains the name of the struct as the table name.
- `NumField()` obtains the number of fields of the instance, and then a specific field is obtained by index with `p := modelType.Field(i)`.
- `p.Name` is the field name, `p.Type` is the field type, which is converted to the database field type via `(Dialect).DataTypeOf()`, and `p.Tag` holds the extra constraints.

Write a test case to verify the Parse function.

```go
// schema_test.go
type User struct {
	Name string `geeorm:"PRIMARY KEY"`
	Age  int
}

var TestDial, _ = dialect.GetDialect("sqlite3")

func TestParse(t *testing.T) {
	schema := Parse(&User{}, TestDial)
	if schema.Name != "User" || len(schema.Fields) != 2 {
		t.Fatal("failed to parse User struct")
	}
	if schema.GetField("Name").Tag != "PRIMARY KEY" {
		t.Fatal("failed to parse primary key")
	}
}
```

## 3 Session

Session's core job is to interact with the database. Therefore, we implement the create/drop operations on database tables in the session sub-package. Before that, the Session struct needs some adjustments.

```go
type Session struct {
	db       *sql.DB
	dialect  dialect.Dialect
	refTable *schema.Schema
	sql      strings.Builder
	sqlVars  []interface{}
}

func New(db *sql.DB, dialect dialect.Dialect) *Session {
	return &Session{
		db:      db,
		dialect: dialect,
	}
}
```

- Session gains two new member variables, dialect and refTable.
- The constructor `New` now takes two arguments, db and dialect.

Create `table.go` in the `session` folder for the code related to operating on database tables.

[day2-reflect-schema/session/table.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day2-reflect-schema/session)

```go
func (s *Session) Model(value interface{}) *Session {
	// nil or different model, update refTable
	if s.refTable == nil || reflect.TypeOf(value) != reflect.TypeOf(s.refTable.Model) {
		s.refTable = schema.Parse(value, s.dialect)
	}
	return s
}

func (s *Session) RefTable() *schema.Schema {
	if s.refTable == nil {
		log.Error("Model is not set")
	}
	return s.refTable
}
```

- The `Model()` method is used to set refTable. Parsing is relatively time-consuming, so the parsed result is stored in the member variable refTable. Even if `Model()` is called multiple times, as long as the name of the passed-in struct does not change, the value of refTable is not updated.
- The `RefTable()` method returns the value of refTable; if refTable has not been set, it logs an error.

Next, implement creating, dropping, and checking the existence of database tables. The three methods follow a similar logic: use the table and field information returned by `RefTable()` to assemble an SQL statement, then execute it through the raw SQL interface.

```go
func (s *Session) CreateTable() error {
	table := s.RefTable()
	var columns []string
	for _, field := range table.Fields {
		columns = append(columns, fmt.Sprintf("%s %s %s", field.Name, field.Type, field.Tag))
	}
	desc := strings.Join(columns, ",")
	_, err := s.Raw(fmt.Sprintf("CREATE TABLE %s (%s);", table.Name, desc)).Exec()
	return err
}

func (s *Session) DropTable() error {
	_, err := s.Raw(fmt.Sprintf("DROP TABLE IF EXISTS %s", s.RefTable().Name)).Exec()
	return err
}

func (s *Session) HasTable() bool {
	sql, values := s.dialect.TableExistSQL(s.RefTable().Name)
	row := s.Raw(sql, values...).QueryRow()
	var tmp string
	_ = row.Scan(&tmp)
	return tmp == s.RefTable().Name
}
```

Implement the corresponding test cases in `table_test.go`:

```go
type User struct {
	Name string `geeorm:"PRIMARY KEY"`
	Age  int
}

func TestSession_CreateTable(t *testing.T) {
	s := NewSession().Model(&User{})
	_ = s.DropTable()
	_ = s.CreateTable()
	if !s.HasTable() {
		t.Fatal("Failed to create table User")
	}
}
```

## 4 Engine

Since Session's constructor now has a dependency on dialect, Engine needs a few small adjustments.

[day2-reflect-schema/geeorm.go](https://github.com/geektutu/7days-golang/tree/master/gee-orm/day2-reflect-schema)

```go
type Engine struct {
	db      *sql.DB
	dialect dialect.Dialect
}

func NewEngine(driver, source string) (e *Engine, err error) {
	db, err := sql.Open(driver, source)
	if err != nil {
		log.Error(err)
		return
	}
	// Send a ping to make sure the database connection is alive.
	if err = db.Ping(); err != nil {
		log.Error(err)
		return
	}
	// make sure the specific dialect exists
	dial, ok := dialect.GetDialect(driver)
	if !ok {
		log.Errorf("dialect %s Not Found", driver)
		return
	}
	e = &Engine{db: db, dialect: dial}
	log.Info("Connect database success")
	return
}

func (engine *Engine) NewSession() *session.Session {
	return session.New(engine.db, engine.dialect)
}
```

- When `NewEngine` creates an Engine instance, it obtains the dialect corresponding to the driver.
- When `NewSession` creates a Session instance, it passes the dialect to the constructor New.

That completes the content of day 2. Let's summarize what was accomplished today:

- 1) To adapt to different databases, a Dialect layer was created to shield away database differences, mapping data types and specific SQL statements.
- 2) Designed Schema, using reflection (reflect) to complete the mapping between structs and database table schemas, including the table name, field names, field types, field tags, etc.
- 3) Built the SQL statements for create, drop, and table-exists to perform basic operations on database tables.

## Recommended Reading

- [A Concise Go Tutorial](https://geektutu.com/post/quick-golang.html)
- [A Concise Tutorial on Go Unit Testing](https://geektutu.com/post/quick-go-test.html)
- [Go Reflect: Improving Reflection Performance](https://geektutu.com/post/hpg-reflect.html)
- [SQLite Common Commands Cheat Sheet](https://geektutu.com/post/cheat-sheet-sqlite.html)
