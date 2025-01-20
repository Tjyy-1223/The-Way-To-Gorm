# 第七章 - Gorm - 高级应用

参考资料：

+ [Gorm 指南 - 中文](https://learnku.com/docs/gorm/v2/index/9728)
+ [GORM 中文文档](https://jasperxu.github.io/gorm-zh/)

### 7.1 多数据源 DBResolver

DBResolver 为 GORM 提供了多个数据库支持，支持以下功能：

- 支持多个 sources、replicas
- 读写分离
- 根据工作表、struct 自动切换连接
- 手动切换连接
- Sources/Replicas 负载均衡
- 适用于原生 SQL

[github.com/go-gorm/dbresolver](https://github.com/go-gorm/dbresolver)

#### 7.1.1 用法示例

```go
import (
  "gorm.io/gorm"
  "gorm.io/plugin/dbresolver"
  "gorm.io/driver/mysql"
)

db, err := gorm.Open(mysql.Open("db1_dsn"), &gorm.Config{})

db.Use(dbresolver.Register(dbresolver.Config{
  // `db2` 作为 sources，`db3`、`db4` 作为 replicas
  Sources:  []gorm.Dialector{mysql.Open("db2_dsn")},
  Replicas: []gorm.Dialector{mysql.Open("db3_dsn"), mysql.Open("db4_dsn")},
  // sources/replicas 负载均衡策略
  Policy: dbresolver.RandomPolicy{},
}).Register(dbresolver.Config{
  // `db1` 作为 sources（DB 的默认连接），对于 `User`、`Address` 使用 `db5` 作为 replicas
  Replicas: []gorm.Dialector{mysql.Open("db5_dsn")},
}, &User{}, &Address{}).Register(dbresolver.Config{
  // `db6`、`db7` 作为 sources，对于 `orders`、`Product` 使用 `db8` 作为 replicas
  Sources:  []gorm.Dialector{mysql.Open("db6_dsn"), mysql.Open("db7_dsn")},
  Replicas: []gorm.Dialector{mysql.Open("db8_dsn")},
}, "orders", &Product{}, "secondary"))
```

这段代码是使用 GORM 的 `dbresolver` 插件配置数据库的负载均衡和读写分离策略。通过 `dbresolver.Register` 注册多个数据库源（`Sources`）和副本（`Replicas`），并配置不同的策略来实现**负载均衡和读写分离** 。接下来是对代码的逐步解析：

**第一段配置**

```go
db.Use(dbresolver.Register(dbresolver.Config{
  // `db2` 作为 sources，`db3`、`db4` 作为 replicas
  Sources:  []gorm.Dialector{mysql.Open("db2_dsn")},
  Replicas: []gorm.Dialector{mysql.Open("db3_dsn"), mysql.Open("db4_dsn")},
  // sources/replicas 负载均衡策略
  Policy: dbresolver.RandomPolicy{},
})
```

+ `Sources: []gorm.Dialector{mysql.Open("db2_dsn")}`: 指定 `db2` 作为主数据库源，所有的写操作将通过这个数据库进行。
+ `Replicas: []gorm.Dialector{mysql.Open("db3_dsn"), mysql.Open("db4_dsn")}`: `db3` 和 `db4` 被配置为副本数据库，这些副本将处理读操作。
+ `Policy: dbresolver.RandomPolicy{}`: 负载均衡策略是 `RandomPolicy`，即读操作在 `db3` 和 `db4` 之间随机选择。

**第二段配置**

```go
}).Register(dbresolver.Config{
  // `db1` 作为 sources（DB 的默认连接），对于 `User`、`Address` 使用 `db5` 作为 replicas
  Replicas: []gorm.Dialector{mysql.Open("db5_dsn")},
}, &User{}, &Address{})
```

+ `Sources: []gorm.Dialector{mysql.Open("db1_dsn")}`（默认连接）: `db1` 是数据库的默认连接，它作为默认的写数据库。
+ `Replicas: []gorm.Dialector{mysql.Open("db5_dsn")}`: 对于 `User` 和 `Address` 表，指定 `db5` 作为它们的副本数据库，处理读操作。
+ `&User{}, &Address{}`: 表示对 `User` 和 `Address` 表使用该配置。换句话说，`User` 和 `Address` 的读操作将由 `db5` 处理，而写操作依然是通过 `db1`。

**第三段配置：**

```go
}).Register(dbresolver.Config{
  // `db6`、`db7` 作为 sources，对于 `orders`、`Product` 使用 `db8` 作为 replicas
  Sources:  []gorm.Dialector{mysql.Open("db6_dsn"), mysql.Open("db7_dsn")},
  Replicas: []gorm.Dialector{mysql.Open("db8_dsn")},
}, "orders", &Product{}, "secondary"))
```

+ `Sources: []gorm.Dialector{mysql.Open("db6_dsn"), mysql.Open("db7_dsn")}`: 指定 `db6` 和 `db7` 作为主数据库源，支持写操作。
+ `Replicas: []gorm.Dialector{mysql.Open("db8_dsn")}`: `db8` 是副本数据库，专门用来处理 `orders` 和 `Product` 表的读操作。
+ `"orders", &Product{}, "secondary"`: 这部分配置指示 `orders` 和 `Product` 表使用 `db8` 作为副本数据库。`"secondary"` 是一个命名标识符，用来区分其他的数据库配置。

**总结：**

这段代码的主要目的是实现基于表的读写分离：

- **第一个配置**: 使得 `db2` 为主数据库，`db3` 和 `db4` 为副本数据库，所有的读操作由副本数据库处理，写操作由主数据库处理。
- **第二个配置**: 使得 `db1` 为默认的主数据库，而 `User` 和 `Address` 表则使用 `db5` 作为副本数据库进行读操作。
- **第三个配置**: `db6` 和 `db7` 是主数据库，`orders` 和 `Product` 表使用 `db8` 作为副本数据库进行读操作。

这种方式通过 `dbresolver` 的配置实现了对不同表的读写分离，并且通过随机负载均衡策略来平衡副本数据库之间的负载。



#### 7.1.2 自动切换连接（读写分离）

DBResolver 会根据工作表、struct 自动切换连接

对于原生 SQL，DBResolver 会从 SQL 中提取表名以匹配 Resolver，除非 SQL 开头为 SELECT（select for update 除外），否则 DBResolver 总是会使用 sources ，例如：

```go
// `User` Resolver 示例
db.Table("users").Rows() // replicas `db5`
db.Model(&User{}).Find(&AdvancedUser{}) // replicas `db5`
db.Exec("update users set name = ?", "jinzhu") // sources `db1`
db.Raw("select name from users").Row().Scan(&name) // replicas `db5`
db.Create(&user) // sources `db1`
db.Delete(&User{}, "name = ?", "jinzhu") // sources `db1`
db.Table("users").Update("name", "jinzhu") // sources `db1`

// Global Resolver 示例
db.Find(&Pet{}) // replicas `db3`/`db4`
db.Save(&Pet{}) // sources `db2`

// Orders Resolver 示例
db.Find(&Order{}) // replicas `db8`
db.Table("orders").Find(&Report{}) // replicas `db8`
```



#### 7.1.3 手动切换连接

```go
// 使用 Write 模式：从 sources db `db1` 读取 user
db.Clauses(dbresolver.Write).First(&user)

// 指定 Resolver：从 `secondary` 的 replicas db `db8` 读取 user
db.Clauses(dbresolver.Use("secondary")).First(&user)

// 指定 Resolver 和 Write 模式：从 `secondary` 的 sources db `db6` 或 `db7` 读取 user
db.Clauses(dbresolver.Use("secondary"), dbresolver.Write).First(&user)
```



#### 7.1.4 负载均衡

GORM 支持基于策略的 sources/replicas 负载均衡，自定义策略应该是一个实现了以下接口的 struct：

```go
type Policy interface {
    Resolve([]gorm.ConnPool) gorm.ConnPool
}
```

当前只实现了一个 `RandomPolicy` 策略，如果没有指定其它策略，它就是默认策略。



#### 7.1.5 连接池

```go
db.Use(
  dbresolver.Register(dbresolver.Config{ /* xxx */ }).
  SetConnMaxIdleTime(time.Hour).
  SetConnMaxLifetime(24 * time.Hour).
  SetMaxIdleConns(100).
  SetMaxOpenConns(200)
)
```



### 7.2 数据库索引

GORM 允许通过 `index`、`uniqueIndex` 标签创建索引，这些索引将在使用 GORM 进行 [AutoMigrate 或 Createtable ](https://learnku.com/docs/gorm/v2/migration)时创建

#### 7.2.1 索引标签

GORM 可以接受很多索引设置，例如 `class`、`type`、`where`、`comment`、`expression`、`sort`、`collate`、`option`

下面的示例演示了如何使用它：

```go
type User struct {
    Name  string `gorm:"index"`
    Name2 string `gorm:"index:idx_name,unique"`
    Name3 string `gorm:"index:,sort:desc,collate:utf8,type:btree,length:10,where:name3 != 'jinzhu'"`
    Name4 string `gorm:"uniqueIndex"`
    Name5 string `gorm:"index:,class:FULLTEXT,option:WITH PARSER ngram"`
    Age   int64  `gorm:"index:,class:FULLTEXT,comment:hello \\, world,where:age > 10"`
    Age2  int64  `gorm:"index:,expression:ABS(age)"`
}
```

#### 7.2.2 唯一索引

`uniqueIndex` 标签的作用与 `index` 类似，它等效于 `index:,unique`

```go
type User struct {
    Name1 string `gorm:"uniqueIndex"`
    Name2 string `gorm:"uniqueIndex:idx_name,sort:desc"`
}
```

#### 7.2.3 复合索引

两个字段使用同一个索引名将创建复合索引，例如：

```go
type User struct {
    Name   string `gorm:"index:idx_member"`
    Number string `gorm:"index:idx_member"`
}
```

#### 7.2.4 字段优先级

复合索引列的顺序会影响其性能，因此必须仔细考虑

您可以使用 `priority` 指定顺序，默认优先级值是 `10`，如果优先级值相同，则顺序取决于模型结构体字段的顺序

```go
type User struct {
    Name   string `gorm:"index:idx_member"`
    Number string `gorm:"index:idx_member"`
}
// column order: name, number

type User struct {
    Name   string `gorm:"index:idx_member,priority:2"`
    Number string `gorm:"index:idx_member,priority:1"`
}
// column order: number, name

type User struct {
    Name   string `gorm:"index:idx_member,priority:12"`
    Number string `gorm:"index:idx_member"`
}
// column order: number, name
```



#### 7.2.5 多索引

一个字段接受多个 `index`、`uniqueIndex` 标签，这会在一个字段上创建多个索引

```go
type UserIndex struct {
    OID          int64  `gorm:"index:idx_id;index:idx_oid,unique"`
    MemberNumber string `gorm:"index:idx_id"`
}
```



### 7.3 约束

GORM 允许通过标签创建数据库约束，约束会在通过 GORM 进行 [AutoMigrate 或创建数据表](https://learnku.com/docs/gorm/v2/migration)时被创建。

#### 7.3.1 检查约束

通过 `check` 标签创建检查约束

```go
type UserIndex struct {
    Name  string `gorm:"check:name_checker,name <> 'jinzhu'"`
    Name2 string `gorm:"check:name <> 'jinzhu'"`
    Name3 string `gorm:"check:,name <> 'jinzhu'"`
}
```

#### 7.3.2 外键约束

GORM 会为关联创建外键约束，您可以在初始化过程中禁用此功能：

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  DisableForeignKeyConstraintWhenMigrating: true,
})
```

GORM 允许您通过 `constraint` 标签的 `Ondelete`、`Ondelete` 选项设置外键约束，例如：

```go
type User struct {
  gorm.Model
  CompanyID  int
  Company    Company    `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"`
  CreditCard CreditCard `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"`
}

type CreditCard struct {
  gorm.Model
  Number string
  UserID uint
}

type Company struct {
  ID   int
  Name string
}
```



### 7.4 复合主键

通过将多个字段设为主键，以创建复合主键，例如：

```go
type Product struct {
  ID           string `gorm:"primaryKey"`
  LanguageCode string `gorm:"primaryKey"`
  Code         string
  Name         string
}
```

**注意：**默认情况下，整型 `PrioritizedPrimaryField` 启用了 `AutoIncrement`，要禁用它，您需要为整型字段关闭 `autoIncrement`：

```go
type Product struct {
  CategoryID uint64 `gorm:"primaryKey;autoIncrement:false"`
  TypeID     uint64 `gorm:"primaryKey;autoIncrement:false"`
}
```



### 7.5 安全

GORM 使用 `database/sql` 的参数占位符来构造 SQL 语句，这可以自动转义参数，避免 SQL 注入数据

**注意** Logger 打印的 SQL 并不像最终执行的 SQL 那样已经转义，复制和运行这些 SQL 时应当注意。



#### 7.5.1 查询条件

用户的输入只能作为参数，例如：

```go
userInput := "jinzhu;drop table users;"

// 安全的，会被转义
db.Where("name = ?", userInput).First(&user)

// SQL 注入
db.Where(fmt.Sprintf("name = %v", userInput)).First(&user)
```

#### 7.5.2 内联条件

```go
// 会被转义
db.First(&user, "name = ?", userInput)

// SQL 注入
db..First(&user, fmt.Sprintf("name = %v", userInput))
```

#### 7.5.3 SQL 注入方法

SQL 注入（SQL Injection）是指攻击者通过恶意构造 SQL 语句来访问、修改、删除或泄露数据库中的敏感数据。攻击者通常通过输入恶意的 SQL 代码来篡改原本的查询逻辑，达到执行任意数据库操作的目的。SQL 注入攻击通常通过不当的用户输入处理、动态拼接 SQL 查询语句等方式发生。

攻击者输入恶意的 SQL 语句片段，使得原本的 SQL 查询变得不安全。

**使用预编译语句（Prepared Statements）**

- **预编译的机制是先编译，再传值，用户传递的参数无法改变SQL语法结构，从根本上解决了SQL注入的问题。**

- 通过使用预编译语句和绑定参数来确保 SQL 语句的结构不会被动态改变。许多现代的数据库库和 ORM（如 GORM）都支持这一功能。

- **示例（Go + GORM）**: 通过预编译，SQL 查询结构得到了保护，任何用户输入的内容都会被安全地处理。

  ```go
  db.Where("username = ? AND password = ?", username, password).First(&user)
  ```

**参数化查询（Parameterized Queries）**

在数据库查询中使用参数占位符（如 `?` 或 `:param`），使得输入值不会直接插入到 SQL 语句中，避免 SQL 注入的发生。

**示例（Python + SQLite）**:

```sql
cursor.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
```

------

为了支持某些功能，一些输入不会被转义，调用方法时要小心用户输入的参数。

```go
db.Select("name; drop table users;").First(&user)
db.Distinct("name; drop table users;").First(&user)

db.Model(&user).Pluck("name; drop table users;", &names)

db.Group("name; drop table users;").First(&user)

db.Group("name").Having("1 = 1;drop table users;").First(&user)

db.Raw("select name from users; drop table users;").First(&user)

db.Exec("select name from users; drop table users;")
```

避免 SQL 注入的一般原则是，不信任用户提交的数据。您可以进行白名单验证来测试用户的输入是否为已知安全的、已批准、已定义的输入，并且在使用用户的输入时，仅将它们作为参数。



### 7.6 GORM 配置

GORM 提供的配置可以在初始化时使用

```go
type Config struct {
  SkipDefaultTransaction bool
  NamingStrategy         schema.Namer
  Logger                 logger.Interface
  NowFunc                func() time.Time
  DryRun                 bool
  PrepareStmt            bool
  AllowGlobalUpdate      bool
  DisableAutomaticPing   bool
  DisableForeignKeyConstraintWhenMigrating bool
}
```

#### 7.6.1 跳动默认事务

为了确保数据一致性，GORM 会在事务里执行写入操作（创建、更新、删除）。如果没有这方面的要求，您可以在初始化时禁用它。

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})
```

#### 7.6.2 命名策略

GORM 允许用户通过覆盖默认的`命名策略`更改默认的命名约定，这需要实现接口 `Namer`

```go
type Namer interface {
    TableName(table string) string
    ColumnName(table, column string) string
    JoinTableName(table string) string
    RelationshipFKName(Relationship) string
    CheckerName(table, column string) string
    IndexName(table, column string) string
}
```

默认 `NamingStrategy` 也提供了几个选项，如：

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  NamingStrategy: schema.NamingStrategy{
    TablePrefix: "t_",   // 表名前缀，`User` 的表名应该是 `t_users`
    SingularTable: true, // 使用单数表名，启用该选项，此时，`User` 的表名应该是 `t_user`
  },
})
```

#### 7.6.3 Logger

允许通过覆盖此选项更改 GORM 的默认 logger，参考 [Logger](https://learnku.com/docs/gorm/v2/logger) 获取详情

#### 7.6.4 NowFunc

更改创建时间使用的函数

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  NowFunc: func() time.Time {
    return time.Now().Local()
  },
})
```

#### 7.6.5 DryRun

生成 `SQL` 但不执行，可以用于准备或测试生成的 SQL，参考 [会话](https://learnku.com/docs/gorm/v2/session) 获取详情

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  DryRun: false,
})
```

#### 7.6.6 PrepareStmt

`PreparedStmt` 在执行任何 SQL 时都会创建一个 prepared statement 并将其缓存，以提高后续的效率，参考 [会话](https://learnku.com/docs/gorm/v2/session) 获取详情

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  PrepareStmt: false,
})
```

#### 7.6.7 AllowGlobalUpdate

启用全局 update/delete，查看 [Session](https://learnku.com/docs/gorm/v2/session) 获取详情

#### 7.6.8 DisableAutomaticPing

在完成初始化后，GORM 会自动 ping 数据库以检查数据库的可用性，若要禁用该特性，可将其设置为 `true`

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  DisableAutomaticPing: true,
})
```

#### 7.6.9 DisableForeignKeyConstraintWhenMigrating

在 `AutoMigrate` 或 `CreateTable` 时，GORM 会自动创建外键约束，若要禁用该特性，可将其设置为 `true`，参考 [迁移](https://learnku.com/docs/gorm/v2/migration) 获取详情。

```go
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  DisableForeignKeyConstraintWhenMigrating: true,
})
```



### 7.7 插件

#### 7.7.1 Callbacks

GORM 自身也是基于 `Callbacks` 的，包括 `Create`、`Query`、`Update`、`Delete`、`Row`、`Raw`。此外，您也完全可以根据自己的意愿自定义 GORM。

+ 回调会注册到全局 `*gorm.DB`，而不是会话级别。
+ 如果您想要 `*gorm.DB` 具有不同的回调，您需要初始化另一个 `*gorm.DB`。

#### 7.7.2 注册 Callback

注册 callback 至 callbacks

```go
func cropImage(db *gorm.DB) {
  if db.Statement.Schema != nil {
    // 伪代码：裁剪图片并上传至 CDN
    for _, field := range db.Statement.Schema.Fields {
      switch db.Statement.ReflectValue.Kind() {
      case reflect.Slice, reflect.Array:
        for i := 0; i < db.Statement.ReflectValue.Len(); i++ {
          // 从字段获取值
          if fieldValue, isZero := field.ValueOf(db.Statement.ReflectValue.Index(i)); !isZero {
            if crop, ok := fieldValue.(CropInterface); ok {
              crop.Crop()
            }
          }
        }
      case reflect.Struct:
        // 从字段获取值
        if fieldValue, isZero := field.ValueOf(db.Statement.ReflectValue); isZero {
          if crop, ok := fieldValue.(CropInterface); ok {
            crop.Crop()
          }
        }

        // 设置字段值
        err := field.Set(db.Statement.ReflectValue, "newValue")
      }
    }

    // 当前 Model 的所有字段
    db.Statement.Schema.Fields

    // 当前 Model 的所有主键字段
    db.Statement.Schema.PrimaryFields

    // 优先主键字段：带有 db 名为 `id` 或定义的第一个主键字段。
    db.Statement.Schema.PrioritizedPrimaryField

    // 当前 Model 的所有关系
    db.Statement.Schema.Relationships

    // 根据 db 名或字段名查找字段
    field := db.Statement.Schema.LookUpField("Name")

    // 处理...
  }
}

db.Callback().Create().Register("crop_image", cropImage)
// 为 Create 流程注册一个 callback
```

#### 7.7.3 删除 Callback

从 callbacks 中删除回调

```go
db.Callback().Create().Remove("gorm:create")
// 从 Create 的 callbacks 中删除 `gorm:create`
```

#### 7.7.4 替换 Callback

用一个新的回调替换已有的同名回调

```go
db.Callback().Create().Replace("gorm:create", newCreateFunction)
// 用新函数 `newCreateFunction` 替换 Create 流程目前的 `gorm:create`
```

#### 7.7.5 注册带顺序的 Callback

注册带顺序的 Callback

```go
// gorm:create 之前
db.Callback().Create().Before("gorm:create").Register("update_created_at", updateCreated)

// gorm:create 之后
db.Callback().Create().After("gorm:create").Register("update_created_at", updateCreated)

// gorm:query 之后
db.Callback().Query().After("gorm:query").Register("my_plugin:after_query", afterQuery)

// gorm:delete 之后
db.Callback().Delete().After("gorm:delete").Register("my_plugin:after_delete", afterDelete)

// gorm:update 之前
db.Callback().Update().Before("gorm:update").Register("my_plugin:before_update", beforeUpdate)

// 位于 gorm:before_create 之后 gorm:create 之前
db.Callback().Create().Before("gorm:create").After("gorm:before_create").Register("my_plugin:before_create", beforeCreate)

// 所有其它 callback 之前
db.Callback().Create().Before("*").Register("update_created_at", updateCreated)

// 所有其它 callback 之后
db.Callback().Create().After("*").Register("update_created_at", updateCreated)
```

#### 7.7.6 预定义 Callback

GORM 已经定义了 [一些 callback](https://github.com/go-gorm/gorm/blob/master/callbacks/callbacks.go) 来支持当前的 GORM 功能，在启动您的插件之前可以先看看这些 callback

#### 7.7.7 插件

GORM 提供了 `Use` 方法来注册插件，插件需要实现 `Plugin` 接口

```go
type Plugin interface {
  Name() string
  Initialize(*gorm.DB) error
}
```

当插件首次注册到 GORM 时将调用 `Initialize` 方法，且 GORM 会保存已注册的插件，你可以这样访问访问：

```go
db.Config.Plugins[pluginName]
```



### 7.8 驱动

GORM 官方支持 sqlite、mysql、postgres、sqlserver。

有些数据库可能兼容 mysql、postgres 的方言，在这种情况下，你可以直接使用这些数据库的方言。

对于其它不兼容的情况，您可以自行编写一个新驱动，这需要实现 方言接口。

```go
type Dialector interface {
    Name() string
    Initialize(*DB) error
    Migrator(db *DB) Migrator
    DataTypeOf(*schema.Field) string
    DefaultValueOf(*schema.Field) clause.Expression
    BindVarTo(writer clause.Writer, stmt *Statement, v interface{})
    QuoteTo(clause.Writer, string)
    Explain(sql string, vars ...interface{}) string
}
```



### 7.9 更新日志

GORM 2.0 是基于用户过去几年中的反馈进行思考后的重写，在该发行版本中将会引入不兼容 API 改动。

- 性能优化
- 代码模块化
- Context、批量插入、Prepared Statment、DryRun 模式、Join Preload, Find 到 Map, FindInBatches 支持
- SavePoint/RollbackTo/Nested Transaction 支持
- 命名参数、Group 条件、Upsert、锁定、优化 / 索引 / 评论提示支持、SubQuery 改进
- 完整的自引用支持，连接表改进，批量数据的关联模式
- 插入时间、更新时间可支持多个字段，加入了对 unix (nano) second 的支持
- 字段级权限控制：只读、只写、只创建、只更新、忽略
- 全新的插件系统：多数据库，由 Database Resolver 提供的读写分离支持，Prometheus 集成，以及更多…
- 全新的 Hook API：带插件的统一接口
- 全新的 Migrator：允许为关系创建数据库外键，约束、检查其支持，增强索引支持
- 全新的 Logger：context 支持、提高可扩展性
- 统一命名策略（表名、字段名、连接表名、外键、检查器、索引名称规则）
- 更好的数据类型定义支持（例如 JSON）





