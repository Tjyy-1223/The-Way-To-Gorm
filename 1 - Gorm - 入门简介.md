# 第一章 - Gorm - 入门简介

参考资料：

+ [Gorm 指南 - 中文](https://learnku.com/docs/gorm/v2/index/9728)
+ [GORM 中文文档](https://jasperxu.github.io/gorm-zh/)

## 1 GORM 指南

GORM 是 Go 语言的一个 ORM（对象关系映射）库，用于简化数据库操作。它提供了一个简洁的接口，让开发者可以通过 Go 的结构体来映射数据库表，执行常见的数据库操作，如增、删、改、查，而不需要编写过多的 SQL 语句。

主要特点：

1. **结构体与数据库表映射**：通过定义 Go 结构体，可以自动生成与数据库表对应的字段映射关系。
2. **简洁的 CRUD 操作**：支持常见的增、删、改、查操作，且接口简洁直观。
3. **自动迁移**：支持自动同步结构体变化到数据库（如创建、更新表结构等）。
4. **支持多种数据库**：支持 MySQL、PostgreSQL、SQLite、SQL Server 等多种数据库。
5. **复杂查询支持**：支持链式调用和条件查询，可以轻松构建复杂的 SQL 查询。

除此之外，GORM 还提供了如下功能：

- 全功能ORM（几乎）
- 关联（包含一个，包含多个，属于，多对多，多种包含）
- Callbacks（创建/保存/更新/删除/查找之前/之后）
- 预加载（急加载）
- 事务
- 复合主键
- SQL Builder
- 自动迁移
- 日志
- 可扩展，编写基于GORM回调的插件
- 每个功能都有测试
- 开发人员友好

GORM 的安装：

```
go get -u github.com/jinzhu/gorm
```

**示例代码：**

```go
package main

import (
  "gorm.io/gorm"
  "gorm.io/driver/sqlite"
)

type Product struct {
  gorm.Model // 包含 ID、CreatedAt、UpdatedAt、DeletedAt 字段
  Code  string 
  Price uint
}

func main() {
  db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
  if err != nil {
    panic("failed to connect database")
  }

  // 迁移 schema
  db.AutoMigrate(&Product{})

  // Create
  db.Create(&Product{Code: "D42", Price: 100})

  // Read
  var product Product
  db.First(&product, 1) // 根据整型主键查找
  db.First(&product, "code = ?", "D42") // 查找 code 字段值为 D42 的记录

  // Update - 将 product 的 price 更新为 200
  db.Model(&product).Update("Price", 200)
  // Update - 更新多个字段
  db.Model(&product).Updates(Product{Price: 200, Code: "F42"}) // 仅更新非零值字段
  db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})

  // Delete - 删除 product
  db.Delete(&product, 1)
}
```

GORM 会根据结构体字段的名称自动推导出数据库表中的字段名。默认情况下，字段名会采用结构体字段的名字，并将其转换为小写蛇形命名（snake_case）格式。具体来说，`Code` 会自动对应数据库中的 `code` 字段。

默认的对应规则：

1. **字段名称**：结构体中的字段名称会被转换为小写的蛇形命名（snake_case）格式。比如：

- `Code` → `code`
- `Price` → `price`

2. **表名**：表名会根据结构体类型的名称来推导，通常是结构体名称的小写形式。如果结构体名称是 `Product`，那么默认的表名就是 `products`（小写复数形式）。你可以通过 `db.Table("custom_table_name")` 来手动指定表名。

3. **外键字段**：如果结构体中有与其他表相关联的外键，GORM 会自动推导外键字段。例如，`User` 和 `Product` 之间的关系，GORM 会自动为外键生成字段，格式通常是 `user_id`。



## 2 模型定义

模型是标准的 struct，由 Go 的基本数据类型、实现了 [Scanner](https://pkg.go.dev/database/sql/?tab=doc#Scanner) 和 [Valuer](https://pkg.go.dev/database/sql/driver#Valuer) 接口的自定义类型及其指针或别名组成

例如：

```go
type User struct {
  ID           uint
  Name         string
  Email        *string
  Age          uint8
  Birthday     *time.Time
  MemberNumber sql.NullString
  ActivedAt    sql.NullTime
  CreatedAt    time.Time
  UpdatedAt    time.Time
}
```

GORM 倾向于约定，而不是配置。默认情况下，GORM 使用 ID 作为主键

+ 使用结构体名的 蛇形复数 作为表名
+ 字段名的 蛇形 作为列名
+ 使用 CreatedAt、UpdatedAt 字段追踪创建、更新时间

遵循 GORM 已有的约定，可以减少您的配置和代码量。如果约定不符合您的需求，GORM 允许您自定义配置它们。

### 2.1 gorm.Model

GORM 定义一个 `gorm.Model` 结构体，其包括字段 `ID`、`CreatedAt`、`UpdatedAt`、`DeletedAt`

```go
// gorm.Model 的定义
type Model struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

您可以将它嵌入到您的结构体中，以包含这几个字段，详情请参考 [嵌入结构体](https://learnku.com/docs/gorm/v2/models/9729#9b91fc)

### 2.2 高级选项

#### 2.2.1 字段级权限控制

可导出的字段在使用 GORM 进行 CRUD 时拥有全部的权限，此外，GORM 允许您用标签控制字段级别的权限。这样您就可以让一个字段的权限是只读、只写、只创建、只更新或者被忽略

注意： 使用 GORM Migrator 创建表时，不会创建被忽略的字段

```go
type User struct {
  Name string `gorm:"<-:create"` // 允许读和创建
  Name string `gorm:"<-:update"` // 允许读和更新
  Name string `gorm:"<-"`        // 允许读和写（创建和更新）
  Name string `gorm:"<-:false"`  // 允许读，禁止写
  Name string `gorm:"->"`        // 只读（除非有自定义配置，否则禁止写）
  Name string `gorm:"->;<-:create"` // 允许读和写
  Name string `gorm:"->:false;<-:create"` // 仅创建（禁止从 db 读）
  Name string `gorm:"-"`  // 读写操作均会忽略该字段
}
```



#### 2.2.2 创建/更新时间追踪（纳秒、毫秒、秒、Time）

GORM 约定使用 `CreatedAt`、`UpdatedAt` 追踪创建 / 更新时间。如果您定义了这种字段，GORM 在创建、更新时会自动填充 [当前时间](https://learnku.com/docs/gorm/v2/gorm_config#now_func)

要使用不同名称的字段，您可以配置 `autoCreateTime`、`autoUpdateTime` 标签

如果您想要保存 UNIX（毫 / 纳）秒时间戳，而不是 time，您只需简单地将 `time.Time` 修改为 `int` 即可

```go
type User struct {
  CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
  UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
  Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳纳秒数填充更新时间
  Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
  Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳秒数填充创建时间
}
```



#### 2.2.3 嵌入结构体

对于匿名字段，GORM 会将其字段包含在父结构体中，例如：

```go
type User struct {
  gorm.Model
  Name string
}
// 等效于
type User struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`
  Name string
}
```

对于正常的结构体字段，你也可以通过标签 `embedded` 将其嵌入，例如：

```go
type Author struct {
    Name  string
    Email string
}

type Blog struct {
  ID      int
  Author  Author `gorm:"embedded"`
  Upvotes int32
}
// 等效于
type Blog struct {
  ID    int64
    Name  string
    Email string
  Upvotes  int32
}
```

并且，您可以使用标签 `embeddedPrefix` 来为 db 中的字段名添加前缀，例如：

```go
type Blog struct {
  ID      int
  Author  Author `gorm:"embedded;embeddedPrefix:author_"`
  Upvotes int32
}
// 等效于
type Blog struct {
  ID          int64
    AuthorName  string
    AuthorEmail string
  Upvotes     int32
}
```



### 2.3 字段标签

声明 model 时，tag 是可选的，GORM 支持以下 tag： tag 名大小写不敏感，但建议使用 `camelCase` 风格

| 标签名         | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| column         | 指定 db 列名                                                 |
| type           | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：not null、size, autoIncrement… 像 varbinary(8) 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：MEDIUMINT UNSIGNED not NULL AUTO_INSTREMENT |
| size           | 指定列大小，例如：`size:256`                                 |
| primaryKey     | 指定列为主键                                                 |
| unique         | 指定列为唯一                                                 |
| default        | 指定列的默认值                                               |
| precision      | 指定列的精度                                                 |
| scale          | 指定列大小                                                   |
| not null       | 指定列为 NOT NULL                                            |
| autoIncrement  | 指定列为自动增长                                             |
| embedded       | 嵌套字段                                                     |
| embeddedPrefix | 嵌入字段的列名前缀                                           |
| autoCreateTime | 创建时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano` |
| autoUpdateTime | 创建 / 更新时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli` |
| index          | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 [索引](https://learnku.com/docs/gorm/v2/indexes) 获取详情 |
| uniqueIndex    | 与 `index` 相同，但创建的是唯一索引                          |
| check          | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://learnku.com/docs/gorm/v2/constraints) 获取详情 |
| <-             | 设置字段写入的权限， `<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限 |
| ->             | 设置字段读的权限，`->:false` 无读权限                        |
| -              | 忽略该字段，`-` 无读写权限                                   |

GORM 允许通过标签为关联配置外键、约束、many2many 表，详情请参考 [关联部分](https://learnku.com/docs/gorm/v2/associations#tags)



## 3 连接数据库

GORM 官方支持的数据库类型有： MySQL, PostgreSQL, SQlite, SQL Server

### 3.1 Mysql

```go
import (
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)

func main() {
  // 参考 https://github.com/go-sql-driver/mysql#dsn-data-source-name 获取详情
  dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
  db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
}
```

**注意：**想要正确的处理 `time.Time` ，您需要带上 `parseTime` 参数， ([更多参数](https://github.com/go-sql-driver/mysql#parameters)) 要支持完整的 UTF-8 编码，您需要将 `charset=utf8` 更改为 `charset=utf8mb4` 查看 [此文章](https://mathiasbynens.be/notes/mysql-utf8mb4) 获取详情

MySQl 驱动程序提供了 [一些高级配置](https://github.com/go-gorm/mysql) 可以在初始化过程中使用，例如：

```go
db, err := gorm.Open(mysql.New(mysql.Config{
  DSN: "gorm:gorm@tcp(127.0.0.1:3306)/gorm?charset=utf8&parseTime=True&loc=Local", // DSN data source name
  DefaultStringSize: 256, // string 类型字段的默认长度
  DisableDatetimePrecision: true, // 禁用 datetime 精度，MySQL 5.6 之前的数据库不支持
  DontSupportRenameIndex: true, // 重命名索引时采用删除并新建的方式，MySQL 5.7 之前的数据库和 MariaDB 不支持重命名索引
  DontSupportRenameColumn: true, // 用 `change` 重命名列，MySQL 8 之前的数据库和 MariaDB 不支持重命名列
  SkipInitializeWithVersion: false, // 根据当前 MySQL 版本自动配置
}), &gorm.Config{})
```

GORM 允许通过 `DriverName` 选项自定义 MySQL 驱动，例如：

```go
import (
  _ "example.com/my_mysql_driver"
  "gorm.io/gorm"
)

db, err := gorm.Open(mysql.New(mysql.Config{
  DriverName: "my_mysql_driver",
  DSN: "gorm:gorm@tcp(localhost:9910)/gorm?charset=utf8&parseTime=True&loc=Local", // Data Source Name，参考 https://github.com/go-sql-driver/mysql#dsn-data-source-name
}), &gorm.Config{})
```

**GORM 允许通过一个现有的数据库连接来初始化 `*gorm.DB`**

```go
import (
  "database/sql"
  "gorm.io/gorm"
)

sqlDB, err := sql.Open("mysql", "mydb_dsn")
gormDB, err := gorm.Open(mysql.New(mysql.Config{
  Conn: sqlDB,
}), &gorm.Config{})
```



### 3.2 PostgreSQL

```go
import (
  "gorm.io/driver/postgres"
  "gorm.io/gorm"
)

dsn := "user=gorm password=gorm dbname=gorm port=9920 sslmode=disable TimeZone=Asia/Shanghai"
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
```

我们使用 [pgx](https://github.com/jackc/pgx) 作为 postgres 的 database/sql 驱动，默认情况下，它会启用 prepared statement 缓存，你可以这样禁用它：

```go
// https://github.com/go-gorm/postgres
db, err := gorm.Open(postgres.New(postgres.Config{
  DSN: "user=gorm password=gorm dbname=gorm port=9920 sslmode=disable TimeZone=Asia/Shanghai",
  PreferSimpleProtocol: true, // disables implicit prepared statement usage
}), &gorm.Config{})
```

GORM 允许通过 `DriverName` 选项自定义 PostgreSQL 驱动，例如：

```go
import (
  _ "github.com/GoogleCloudPlatform/cloudsql-proxy/proxy/dialers/postgres"
  "gorm.io/gorm"
)

db, err := gorm.Open(postgres.New(postgres.Config{
  DriverName: "cloudsqlpostgres",
  DSN: "host=project:region:instance user=postgres dbname=postgres password=password sslmode=disable",
})
```

**GORM 允许通过一个现有的数据库连接来初始化 `*gorm.DB`**

```go
import (
  "database/sql"
  "gorm.io/gorm"
)

sqlDB, err := sql.Open("postgres", "mydb_dsn")
gormDB, err := gorm.Open(postgres.New(postgres.Config{
  Conn: sqlDB,
}), &gorm.Config{})
```



### 3.3 SQLite

```go
import (
  "gorm.io/driver/sqlite"
  "gorm.io/gorm"
)

// github.com/mattn/go-sqlite3
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{})
```



### 3.4 SQL Server

```go
import (
  "gorm.io/driver/sqlserver"
  "gorm.io/gorm"
)

// github.com/denisenkom/go-mssqldb
dsn := "sqlserver://gorm:LoremIpsum86@localhost:9930?database=gorm"
db, err := gorm.Open(sqlserver.Open(dsn), &gorm.Config{})
```



### 3.5 ClickHouse

```go
package main

import (
    "gorm.io/driver/clickhouse"
    "gorm.io/gorm"
)

type ExampleModel struct {
    ID   uint   `gorm:"column:id"`
    Name string `gorm:"column:name"`
    Age  uint   `gorm:"column:age"`
}

func main() {
    dsn := "clickhouse://username:password@host:port/database?dial_timeout=10s&read_timeout=20s"
    db, err := gorm.Open(clickhouse.Open(dsn), &gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }

    // 创建数据表
    if err := db.AutoMigrate(&ExampleModel{}); err != nil {
        log.Fatal(err)
    }

    // 写入数据
    example := &ExampleModel{ID: 1, Name: "John Doe", Age: 25}
    if err := db.Create(example).Error; err != nil {
        log.Fatal(err)
    }

    // 查询单个数据
    var result ExampleModel
    if err := db.First(&result, "id = ?", 1).Error; err != nil {
      log.Fatal(err)
    }
    fmt.Printf("ID: %d, Name: %s, Age: %d\n", result.ID, result.Name, result.Age)

    // 查询所有数据
    var exas []ExampleModel
    db.Find(&exas, "age > ? ", 25)
    fmt.Println(exas)

    //批量保存
    e1 := ExampleModel{ID: 4, Name: "name4", Age: 31}
    e2 := ExampleModel{ID: 5, Name: "name5", Age: 32}
    e3 := ExampleModel{ID: 6, Name: "name6", Age: 33}
    es := []ExampleModel{e1, e2, e3}
    db.Create(&es)

}
```



### 3.6 连接池

GORM 使用 [database/sql](https://pkg.go.dev/database/sql) 维护连接池

```go
sqlDB, err := db.DB()

// SetMaxIdleConns 设置空闲连接池中连接的最大数量
sqlDB.SetMaxIdleConns(10)

// SetMaxOpenConns 设置打开数据库连接的最大数量。
sqlDB.SetMaxOpenConns(100)

// SetConnMaxLifetime 设置了连接可复用的最大时间。
sqlDB.SetConnMaxLifetime(time.Hour)
```

查看 [通用接口](https://learnku.com/docs/gorm/v2/generic_interface) 获取详情。