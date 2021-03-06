---
title: SQL Builder
layout: page
---

## Raw SQL

Query Raw SQL

```go
Select("name, age, email"). Rows() // (*sql. Rows, error)
defer rows. Close()

for rows. Next() {
  var user User
  // ScanRows scan a row into user
  db. ScanRows(rows, &user)

  // do something
}
```

Exec Raw SQL

```go
db. Exec("DROP TABLE users")
db. Exec("UPDATE orders SET shipped_at=? WHERE id IN ?", time. Now(), []int64{1,2,3})

// SQL Expression
DB. Exec("update users set money=? Where("name = ?", "jinzhu"). + ?", 10000, 1), "jinzhu")
```

**NOTE** GORM allows cache prepared statement to increase performance, checkout [Performance](performance.html) for details

## `Row` & `Rows`

Rows() // (*sql.

```go
Where("name = ?", "jinzhu"). Select("name, age, email"). Rows() // (*sql. Rows, error)
defer rows. Close()

for rows. Next() {
  var user User
  // ScanRows scan a row into user
  db. ScanRows(rows, &user)

  // do something
}
```

Rows() // (*sql.

```go
rows, err := db. Model(&User{}). Where("name = ?", "jinzhu"). Select("name, age, email"). Rows() // (*sql. Rows, error)
defer rows. Close()

for rows. Next() {
  var user User
  // ScanRows scan a row into user
  db. ScanRows(rows, &user)

  // do something
}
```

Checkout [FindInBatches](advanced_query.html) for how to query and process records in batch Checkout [Group Conditions](advanced_query.html#group_conditions) for how to build complicated SQL Query

## <span id="named_argument">Named Argument</span>

GORM supports named arguments with [`sql.NamedArg`](https://tip.golang.org/pkg/database/sql/#NamedArg) or `map[string]interface{}{}`, for example:

```go
DB.Where("name1 = @name OR name2 = @name", sql.Named("name", "jinzhu")).Find(&user)
// SELECT * FROM `named_users` WHERE name1 = "jinzhu" OR name2 = "jinzhu"

DB.Where("name1 = @name OR name2 = @name", map[string]interface{}{"name": "jinzhu2"}).First(&result3)
// SELECT * FROM `named_users` WHERE name1 = "jinzhu2" OR name2 = "jinzhu2" ORDER BY `named_users`.`id` LIMIT 1

DB.Raw("SELECT * FROM named_users WHERE name1 = @name OR name2 = @name2 OR name3 = @name", sql.Named("name", "jinzhu1"), sql.Named("name2", "jinzhu2")).Find(&user)
// SELECT * FROM named_users WHERE name1 = "jinzhu1" OR name2 = "jinzhu2" OR name3 = "jinzhu1"

DB.Exec("UPDATE named_users SET name1 = @name, name2 = @name2, name3 = @name", sql.Named("name", "jinzhunew"), sql.Named("name2", "jinzhunew2"))
// UPDATE named_users SET name1 = "jinzhunew", name2 = "jinzhunew2", name3 = "jinzhunew"

DB.Raw("SELECT * FROM named_users WHERE (name1 = @name AND name3 = @name) AND name2 = @name2", map[string]interface{}{"name": "jinzhu", "name2": "jinzhu2"}).Find(&user)
// SELECT * FROM named_users WHERE (name1 = "jinzhu" AND name3 = "jinzhu") AND name2 = "jinzhu2"
```

## Scan `*sql. Rows` into struct

```go
rows, err := db. Model(&User{}). Where("name = ?", "jinzhu"). Select("name, age, email"). Rows() // (*sql. Rows, error)
defer rows. Close()

for rows. Next() {
  var user User
  // ScanRows scan a row into user
  db. ScanRows(rows, &user)

  // do something
}
```

## DryRun Mode

Generate `SQL` without executing, can be used to prepare or test generated SQL, Checkout [Session](session.html) for details

```go
stmt := DB. Session(&Session{DryRun: true}). First(&user, 1). Statement
stmt.SQL. String() //=> SELECT * FROM `users` WHERE `id` = $1 ORDER BY `id`
stmt. Vars         //=> []interface{}{1}
```

## Advanced

### Clauses

GORM uses SQL builder generates SQL internally, for each operation, GORM creates a `*gorm. Statement` object, all GORM APIs add/change `Clause` for the `Statement`, at last, GORM generated SQL based on those clauses

For example, when querying with `First`, it adds the following clauses to the `Statement`

```go
clause. Select{Columns: "*"}
clause. From{Tables: clause. CurrentTable}
clause. Limit{Limit: 1}
clause. OrderByColumn{
  Column: clause. Column{Table: clause. CurrentTable, Name: clause. PrimaryKey},
}
```

Then GORM build finally querying SQL in callbacks like:

```go
Statement. Build("SELECT", "FROM", "WHERE", "GROUP BY", "ORDER BY", "LIMIT", "FOR")
```

Which generate SQL:

```sql
SELECT * FROM `users` ORDER BY `users`.`id` LIMIT 1
```

You can define your own `Clause` and use it with GORM, it needs to implements [Interface](https://pkg.go.dev/gorm.io/gorm/clause?tab=doc#Interface)

Check out [examples](https://github.com/go-gorm/gorm/tree/master/clause) for reference

### Clause Builder

For different databases, Clauses may generate different SQL, for example:

```go
Find(&User{})
// SELECT * /*+ hint */ FROM `users`
```

Which is supported because GORM allows database driver register Clause Builder to replace the default one, take the [Limit](https://github.com/go-gorm/sqlserver/blob/512546241200023819d2e7f8f2f91d7fb3a52e42/sqlserver.go#L45) as example

### Clause Options

GORM defined [Many Clauses](https://github.com/go-gorm/gorm/tree/master/clause), and some clauses provide advanced options can be used for your application

Although most of them are rarely used, if you find GORM public API can't match your requirements, may be good to check them out, for example:

```go
DB. Clauses(clause. Insert{Modifier: "IGNORE"}). Create(&user)
// INSERT IGNORE INTO users (name,age...) VALUES ("jinzhu",18...);
```

### StatementModifier

GORM provides interface [StatementModifier](https://pkg.go.dev/gorm.io/gorm?tab=doc#StatementModifier) allows you modify statement to match your requirements, take [Hints](hints.html) as example

```go
import "gorm.io/hints"

DB. Clauses(hints. New("hint")). Find(&User{})
// SELECT * /*+ hint */ FROM `users`
```
