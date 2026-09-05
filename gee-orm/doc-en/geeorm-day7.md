---
title: Implement an ORM Framework in Go - GeeORM Day 7 Database Migration
description: >-
  A 7-day tutorial on implementing an ORM framework GeeORM in Go/golang from scratch (7 days implement golang object relational mapping framework from scratch
  tutorial). Build an ORM framework modeled after the implementations of gorm and xorm. When a struct changes, the fields of the database table are migrated
  automatically; only adding and deleting fields is supported, changing field types is not.
date: '2020-03-09 07:00:00'
tags:
  - Go
book: 7days-golang
status: done
draft: false
cover: geeorm/geeorm_sm.jpg
lang: en
---

This article is part of the [7 Days Go ORM Framework Tutorial Series from scratch](https://geektutu.com/post/geeorm.html).

- When a struct changes, the fields of the database table are migrated automatically.
- Only adding and deleting fields is supported, changing field types is not. **About 70 lines of code**

## Migrating with SQL Statements

Database migration has always been one of the biggest headaches for database administrators. Adding or deleting columns in a single table is fairly easy, but once foreign keys and other complex relationships are involved, database migration becomes very difficult.

GeeORM's Migrate operation targets only the simplest scenario: it supports adding and deleting fields, but does not support changing field types.

Before implementing Migrate, let's first look at how to add and delete fields using native SQL statements.

### Adding Fields

```sql
ALTER TABLE table_name ADD COLUMN col_name, col_type;
```

Most databases support using the `ALTER` keyword to add fields, or to rename fields.

### Deleting Fields

> Reference: [sqlite delete or add column - stackoverflow](https://stackoverflow.com/questions/8442147/how-to-delete-or-add-column-in-sqlite)

For SQLite, deleting a field is not as easy as adding one. A viable approach requires the following steps:

```sql
CREATE TABLE new_table AS SELECT col1, col2, ... from old_table
DROP TABLE old_table
ALTER TABLE new_table RENAME TO old_table;
```

- Step 1: Select the fields to keep from `old_table` into `new_table`.
- Step 2: Drop `old_table`.
- Step 3: Rename `new_table` back to `old_table`.

## Implementing Migrate in GeeORM

Following the native SQL commands and making use of the transaction implemented earlier, implement the Migrate method in `geeorm.go`.

```go
// difference returns a - b
func difference(a []string, b []string) (diff []string) {
	mapB := make(map[string]bool)
	for _, v := range b {
		mapB[v] = true
	}
	for _, v := range a {
		if _, ok := mapB[v]; !ok {
			diff = append(diff, v)
		}
	}
	return
}

// Migrate table
func (engine *Engine) Migrate(value interface{}) error {
	_, err := engine.Transaction(func(s *session.Session) (result interface{}, err error) {
		if !s.Model(value).HasTable() {
			log.Infof("table %s doesn't exist", s.RefTable().Name)
			return nil, s.CreateTable()
		}
		table := s.RefTable()
		rows, _ := s.Raw(fmt.Sprintf("SELECT * FROM %s LIMIT 1", table.Name)).QueryRows()
		columns, _ := rows.Columns()
		addCols := difference(table.FieldNames, columns)
		delCols := difference(columns, table.FieldNames)
		log.Infof("added cols %v, deleted cols %v", addCols, delCols)

		for _, col := range addCols {
			f := table.GetField(col)
			sqlStr := fmt.Sprintf("ALTER TABLE %s ADD COLUMN %s %s;", table.Name, f.Name, f.Type)
			if _, err = s.Raw(sqlStr).Exec(); err != nil {
				return
			}
		}

		if len(delCols) == 0 {
			return
		}
		tmp := "tmp_" + table.Name
		fieldStr := strings.Join(table.FieldNames, ", ")
		s.Raw(fmt.Sprintf("CREATE TABLE %s AS SELECT %s from %s;", tmp, fieldStr, table.Name))
		s.Raw(fmt.Sprintf("DROP TABLE %s;", table.Name))
		s.Raw(fmt.Sprintf("ALTER TABLE %s RENAME TO %s;", tmp, table.Name))
		_, err = s.Exec()
		return
	})
	return err
}
```

- `difference` computes the difference between two field slices. New table - old table = added fields; old table - new table = deleted fields.
- Uses `ALTER` statements to add fields.
- Deletes fields by creating a new table and renaming it.

## Testing

Add a test case for Migrate in `geeorm_test.go`:

```go
type User struct {
	Name string `geeorm:"PRIMARY KEY"`
	Age  int
}

func TestEngine_Migrate(t *testing.T) {
	engine := OpenDB(t)
	defer engine.Close()
	s := engine.NewSession()
	_, _ = s.Raw("DROP TABLE IF EXISTS User;").Exec()
	_, _ = s.Raw("CREATE TABLE User(Name text PRIMARY KEY, XXX integer);").Exec()
	_, _ = s.Raw("INSERT INTO User(`Name`) values (?), (?)", "Tom", "Sam").Exec()
	engine.Migrate(&User{})

	rows, _ := s.Raw("SELECT * FROM User").QueryRows()
	columns, _ := rows.Columns()
	if !reflect.DeepEqual(columns, []string{"Name", "Age"}) {
		t.Fatal("Failed to migrate table User, got columns", columns)
	}
}
```

- First, assume the original `User` contains two fields, `Name` and `XXX`. After a business change, the fields of the `User` struct are changed to `Name` and `Age`.
- That is, the old field `XXX` needs to be deleted and the field `Age` needs to be added.
- After calling `Migrate(&User{})`, the structure of the new table is `Name`, `Age`

## Summary

GeeORM's overall implementation is fairly rough; for example, database migration only considers the simplest scenario. The implemented features are also few; scenarios such as nested structs, foreign keys, and composite primary keys are not covered. ORM frameworks are generally large in code size. First, to get as close to the database as possible, a large amount of code is needed to implement the related features. Second, databases differ considerably from one another; the more features implemented, the more prominent these differences become, and sometimes, to achieve good performance, special handling has to be done for each database. In addition, some ORM frameworks support both relational and non-relational databases, which requires the framework itself to have a higher level of abstraction and not be confined to the SQL layer.

It is impossible to achieve all of this with only about 800 lines of code in GeeORM. However, the goal of GeeORM is not to implement an ORM framework that can be used in production, but rather to introduce as much as possible the general implementation principles of ORM frameworks, for example:

- How to shield the differences between different databases within the framework;
- How the table structures in a database are mapped to objects in a programming language;
- How to elegantly emulate query conditions — chained calls are a good choice;
- Why ORM frameworks usually provide the ability to extend with hooks;
- The principles of transactions and how an ORM framework integrates support for transactions;
- Some difficult problems, such as database migration.
- ...

Based on these points, I think GeeORM has achieved its goal.

## Appendix: Recommended Reading

- [A Concise Go Test Unit Testing Tutorial](https://geektutu.com/post/quick-go-test.html)
- [SQLite Common Commands Cheat Sheet](https://geektutu.com/post/cheat-sheet-sqlite.html)
- [sqlite delete or add column - stackoverflow](https://stackoverflow.com/questions/8442147/how-to-delete-or-add-column-in-sqlite)
