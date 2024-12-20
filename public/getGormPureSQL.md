---
title: GORMで実行した生SQLを無理やり取得する方法
tags:
  - Go
  - GORM
private: false
updated_at: '2024-12-21T02:59:13+09:00'
id: c520679a2663b1b6b5f1
organization_url_name: null
slide: false
ignorePublish: false
---

# 目的、方針
SQLを文字列として受け取ってログ出力に使いたい！（GORM自体に実行したSQLのログ出力はあるけど、個人的に融通が効かなくて。。。）
パッケージのソースに手を加える形で修正。いい感じに取得できればいいな。

# 環境
- Go    1.23.2
- GORM  1.25.10

# ソース修正

:::note alert
**注意**
修正する際は自己責任でお願いします！
:::

```go:callbacks.go
func (p *processor) Execute(db *DB) *DB {
// ...省略...
	if stmt.SQL.Len() > 0 {
		db.Logger.Trace(stmt.Context, curTime, func() (string, int64) {
			sql, vars := stmt.SQL.String(), stmt.Vars
			if filter, ok := db.Logger.(ParamsFilter); ok {
				sql, vars = filter.ParamsFilter(stmt.Context, stmt.SQL.String(), stmt.Vars...)
			}
            // ↓↓ココ！！！↓↓
			sql = db.Dialector.Explain(sql, vars...)
			db.Sql = sql
			return sql, db.RowsAffected
            // ↑↑↑↑
		}, db.Error)
	}

// ...省略...

	return db
}
```

```go:gorm.go
// DB GORM DB definition
type DB struct {
	*Config
	Error        error
	RowsAffected int64
	Statement    *Statement
	clone        int
  // ↓追加
	Sql          string
}
```

```go:db.go
	db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
    // ↓ロガーの設定これ大事
		Logger: logger.Default.LogMode(logger.Info),
	})
```

```go:select.go
	result := db.Limit(10).Find(&pokemons)
	fmt.Println(result.Sql)
  // ↓出力結果↓
  // SELECT * FROM "pokemons" WHERE "pokemons"."deleted_at" IS NULL LIMIT 10
```

# 解説

callbacks.goのExecuteメソッドですが、SQLを実行する間際になります。
db.Dialector.Explainの返り値はそのまま実行できる生SQLになります。

処理を一通り見たところ、GORMの利用者に返却されるものはDB構造体であることがわかります。
なのでDB構造体にSqlフィールドを追加して格納してあげます。

パッケージのソースはこの二つのファイルを修正して終わりになります。（意外と簡単）

```go:callbacks.go
func (p *processor) Execute(db *DB) *DB {
// ...省略...
	if stmt.SQL.Len() > 0 {
		db.Logger.Trace(stmt.Context, curTime, func() (string, int64) {
			sql, vars := stmt.SQL.String(), stmt.Vars
			if filter, ok := db.Logger.(ParamsFilter); ok {
				sql, vars = filter.ParamsFilter(stmt.Context, stmt.SQL.String(), stmt.Vars...)
			}
            // ↓↓ココ！！！↓↓
			sql = db.Dialector.Explain(sql, vars...)
			db.Sql = sql
			return sql, db.RowsAffected
            // ↑↑↑↑
		}, db.Error)
	}

// ...省略...

	return db
}
```

```go:gorm.go
// DB GORM DB definition
type DB struct {
	*Config
	Error        error
	RowsAffected int64
	Statement    *Statement
	clone        int
  // ↓追加
	Sql          string
}
```

ここからはGORMを利用するプロジェクト側になります。
DB接続する際にログの設定をします。
この設定をすることでログにSQLが出力されるようになります。
今回は生SQLをログではなく内部で文字列として受け取りたいので、上記でこねくり回しています。

GORMの中でどのようにログ出力しているか気になる方はlogger.goのTrace()をみるといいかもしれません。

```go:db.go
	db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
    // ↓ロガーの設定これ大事
		Logger: logger.Default.LogMode(logger.Info),
	})
```

最後に使い方になります。
FindやFirstの返り値にはDB構造体のデータが入っています。
Executeメソッドを通っているはずなので、db.Sqlには値が入っているはずです。
なのでresult.Sqlで取得することができます。

```go:select.go
	result := db.Limit(10).Find(&pokemons)
	fmt.Println(result.Sql)
  // ↓出力結果↓
  // SELECT * FROM "pokemons" WHERE "pokemons"."deleted_at" IS NULL LIMIT 10
```

# まとめ

こんなやり方じゃなくてもスマートなやり方あったかもなー
デバッグでここまで処理がたどり着いたけど、正直途中の処理全然意味分かってないからまずい

:::note alert
**注意**
筆者はFind関数でしか確かめておりません！
他の関数では動かない可能性があります！
:::

:::note info
変なこと言ってたら指摘などあれば嬉しいです！
:::
