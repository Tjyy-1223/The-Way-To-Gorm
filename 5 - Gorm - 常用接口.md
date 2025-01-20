# 第五章 - Gorm - 常用接口

参考资料：

+ [Gorm 指南 - 中文](https://learnku.com/docs/gorm/v2/index/9728)
+ [GORM 中文文档](https://jasperxu.github.io/gorm-zh/)

### 5.1 Context

GORM 通过 `WithContext` 方法提供了 Context 支持

**5.1.1 单会话模式**

单会话模式通常被用于执行单次操作

```go
db.WithContext(ctx).Find(&users)
```

**5.1.2 持续会话模式**

持续会话模式通常被用于执行一系列操作，例如：

```go
tx := db.WithContext(ctx)
tx.First(&user, 1)
tx.Model(&user).Update("role", "admin")
```

**5.1.3 在 Hooks/Callbacks 中使用 Context**

您可以从当前 `Statement` 中访问 `Context` 对象，例如︰

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  ctx := tx.Statement.Context
  // ...
  return
}
```

**5.1.4 Chi 中间件示例**

在处理 API 请求时持续会话模式会比较有用。例如，您可以在中间件中为 `*gorm.DB` 设置超时 Context，然后使用 `*gorm.DB` 处理所有请求

下面是一个 Chi 中间件的示例：

```go
func SetDBMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    timeoutContext, _ := context.WithTimeout(context.Background(), time.Second)
    ctx := context.WithValue(r.Context(), "DB", db.WithContext(timeoutContext))
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

r := chi.NewRouter()
r.Use(SetDBMiddleware)

r.Get("/", func(w http.ResponseWriter, r *http.Request) {
  db, ok := ctx.Value("DB").(*gorm.DB)

  var users []User
  db.Find(&users)

  // 你的其他 DB 操作...
})

r.Get("/user", func(w http.ResponseWriter, r *http.Request) {
  db, ok := ctx.Value("DB").(*gorm.DB)

  var user User
  db.First(&user)

  // 你的其他 DB 操作...
})
```

**注意** 通过 `WithContext` 设置的 `Context` 是线程安全的，参考[会话](https://learnku.com/docs/gorm/v2/session)获取详情



### 5.2 错误处理

在 Go 中，处理错误是很重要的。

我们鼓励您在调用任何 [Finisher 方法](https://learnku.com/docs/gorm/v2/method_chaining#finisher_method) 后，都进行错误检查

**处理错误**

GORM 的错误处理与常见的 Go 代码不同，因为 GORM 提供的是链式 API。

如果遇到任何错误，GORM 会设置 `*gorm.DB` 的 `Error` 字段，您需要像这样检查它：

```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
  // 处理错误...
}
```

或者

```php
if result := db.Where("name = ?", "jinzhu").First(&user); result.Error != nil {
  // 处理错误...
}
```

**ErrRecordNotFound**
当 First、Last、Take 方法找不到记录时，GORM 会返回 ErrRecordNotFound 错误。如果发生了多个错误，你可以通过 errors.Is 判断错误是否为 ErrRecordNotFound，例如：

```go
// 检查错误是否为 RecordNotFound
err := db.First(&user, 100).Error
errors.Is(err, ErrRecordNotFound)
```



### 5.3 链式调用

GORM 允许进行链式操作，所以您可以像这样写代码：

```php
db.Where("name = ?", "jinzhu").Where("age = ?", 18).First(&user)
```

GORM 中有三种类型的方法： `链式方法`、`Finisher 方法`、`新建会话方法`

**链式方法**

链式方法是将 `Clauses` 修改或添加到当前 `Statement` 的方法，例如：

`Where`, `Select`, `Omit`, `Joins`, `Scopes`, `Preload`, `Raw` (`Raw` can’t be used with other chainable methods to build SQL)…

这是 [完整方法列表](https://github.com/go-gorm/gorm/blob/master/chainable_api.go)

**Finisher 方法**

Finishers 是会立即执行注册回调的方法，然后生成并执行 SQL，比如这些方法：

`Create`, `First`, `Find`, `Take`, `Save`, `Update`, `Delete`, `Scan`, `Row`, `Rows`…

查看[完整方法列表](https://github.com/go-gorm/gorm/blob/master/finisher_api.go)

**新建会话方法**

在初始化了 `*gorm.DB` 或 `新建会话方法` 后， 调用下面的方法会创建一个新的 `Statement` 实例而不是使用当前的。

**GROM 定义了 `Session`、`WithContext`、`Debug` 方法做为 `新建会话方法`**，查看[会话](https://learnku.com/docs/gorm/v2/session) 获取详情

让我们用一些例子来解释它：

示例 1：

```go
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
// db 是一个刚完成初始化的 *gorm.DB 实例，这是一个 `新建会话`

db.Where("name = ?", "jinzhu").Where("age = ?", 18).Find(&users)
// `Where("name = ?", "jinzhu")` 是调用的第一个方法，它会创建一个新 `Statement`
// `Where("age = ?", 18)` 会复用 `Statement`，并将条件添加至这个 `Statement`
// `Find(&users)` 是一个 finisher 方法，它运行注册的查询回调，生成并运行下面这条 SQL：
// SELECT * FROM users WHERE name = 'jinzhu' AND age = 18;

db.Where("name = ?", "jinzhu2").Where("age = ?", 20).Find(&users)
// `Where("name = ?", "jinzhu2")` 也是调用的第一个方法，也会创建一个新 `Statement`
// `Where("age = ?", 20)` 会复用 `Statement`，并将条件添加至这个 `Statement`
// `Find(&users)` 是一个 finisher 方法，它运行注册的查询回调，生成并运行下面这条 SQL：
// SELECT * FROM users WHERE name = 'jinzhu2' AND age = 20;

db.Find(&users)
// 对于这个 `新建会话模式` 的 `*gorm.DB` 实例来说，`Find(&users)` 是一个 finisher 方法也是第一个调用的方法。 
// 它创建了一个新的 `Statement` 运行注册的查询回调，生成并运行下面这条 SQL：
// SELECT * FROM users;
```

示例 2：

```go
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
// db 是一个刚完成初始化的 *gorm.DB 实例，这是一个 `新建会话`

tx := db.Where("name = ?", "jinzhu")
// `Where("name = ?", "jinzhu")` 是第一个被调用的方法，它创建了一个新的 `Statement` 并添加条件

tx.Where("age = ?", 18).Find(&users)
// `tx.Where("age = ?", 18)` 会复用上面的那个 `Statement`，并向其添加条件
// `Find(&users)` 是一个 finisher 方法，它运行注册的查询回调，生成并运行下面这条 SQL：
// SELECT * FROM users WHERE name = 'jinzhu' AND age = 18

tx.Where("age = ?", 28).Find(&users)
// `tx.Where("age = ?", 18)` 同样会复用上面的那个 `Statement`，并向其添加条件
// `Find(&users)` 是一个 finisher 方法，它运行注册的查询回调，生成并运行下面这条 SQL：
// SELECT * FROM users WHERE name = 'jinzhu' AND age = 18 AND age = 28;
```

**注意** 在示例 2 中，第一个查询会影响第二个查询生成的 SQL ，因为 GORM 复用 `Statement` 这可能会引发预期之外的问题，请参考 [线程安全](https://learnku.com/docs/gorm/v2/method_chaining#goroutine_safe) 了解如何避免该问题。

**方法链和协程安全**

新初始化的 `*gorm.DB` 或调用 `新建会话方法` 后，GORM 会创建新的 `Statement` 实例。因此想要复用 `*gorm.DB`，您需要确保它们处于 `新建会话模式`，例如：

```go
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})

// 安全的使用新初始化的 *gorm.DB
for i := 0; i < 100; i++ {
  go db.Where(...).First(&user)
}

tx := db.Where("name = ?", "jinzhu")
// 不安全的复用 Statement
for i := 0; i < 100; i++ {
  go tx.Where(...).First(&user)
}

// GROM 定义了 `Session`、`WithContext`、`Debug` 方法做为 `新建会话方法`
ctx, _ := context.WithTimeout(context.Background(), time.Second)
ctxDB := db.WithContext(ctx)
// 在 `新建会话方法` 之后是安全的
for i := 0; i < 100; i++ {
  go ctxDB.Where(...).First(&user)
}

ctx, _ := context.WithTimeout(context.Background(), time.Second)
ctxDB := db.Where("name = ?", "jinzhu").WithContext(ctx)
// 在 `新建会话方法` 之后是安全的
for i := 0; i < 100; i++ {
  go ctxDB.Where(...).First(&user) // `name = 'jinzhu'` 会应用到查询中
}

tx := db.Where("name = ?", "jinzhu").Session(&gorm.Session{})
// 在 `新建会话方法` 之后是安全的
for i := 0; i < 100; i++ {
  go tx.Where(...).First(&user) // `name = 'jinzhu'` 会应用到查询中
}
```



### 5.4 会话

GORM 提供了 `Session` 方法，这是一个 [`New Session Method`](https://learnku.com/docs/gorm/v2/method_chaining)，它允许创建带配置的新建会话模式：

```go
// Session 配置
type Session struct {
  DryRun                 bool
  PrepareStmt            bool
  NewDB                  bool
  SkipHooks              bool
  SkipDefaultTransaction bool
  AllowGlobalUpdate      bool
  FullSaveAssociations   bool
  Context                context.Context
  Logger                 logger.Interface
  NowFunc                func() time.Time
}
```

**DryRun**

DryRun 模式会生成但不执行 `SQL`，可以用于准备或测试生成的 SQL，详情请参考 Session：

```go
// 新建会话模式
stmt := db.Session(&Session{DryRun: true}).First(&user, 1).Statement
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 ORDER BY `id`
stmt.Vars         //=> []interface{}{1}

// 全局 DryRun 模式
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{DryRun: true})

// 不同的数据库生成不同的 SQL
stmt := db.Find(&user, 1).Statement
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = $1 // PostgreSQL
stmt.SQL.String() //=> SELECT * FROM `users` WHERE `id` = ?  // MySQL
stmt.Vars         //=> []interface{}{1}
```

译者注：运行不了？`&Session{DryRun: true}` 改为 `&gorm.Session{DryRun: true}`

你可以使用下面的代码生成最终的 SQL：

```go
// 注意：SQL 并不总是能安全地执行，GORM 仅将其用于日志，它可能导致会 SQL 注入
db.Dialector.Explain(stmt.SQL.String(), stmt.Vars...)
// SELECT * FROM `users` WHERE `id` = 1
```

**预编译**

`PreparedStmt` 在执行任何 SQL 时都会创建一个 prepared statement 并将其缓存，以提高后续的效率，例如：

```go
// 全局模式，所有 DB 操作都会 创建并缓存预编译语句
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  PrepareStmt: true,
})

// 会话模式
tx := db.Session(&Session{PrepareStmt: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)

// returns prepared statements manager
stmtManger, ok := tx.ConnPool.(*PreparedStmtDB)

// 关闭 *当前会话* 的预编译模式
stmtManger.Close()

// 为 *当前会话* 预编译 SQL
stmtManger.PreparedSQL // => []string{}

// 为当前数据库连接池的（所有会话）开启预编译模式
stmtManger.Stmts // map[string]*sql.Stmt

for sql, stmt := range stmtManger.Stmts {
  sql  // 预编译 SQL
  stmt // 预编译模式
  stmt.Close() // 关闭预编译模式
}
```

**NewDB**

通过 `NewDB` 选项创建一个不带之前条件的新 DB，例如：

```go
tx := db.Where("name = ?", "jinzhu").Session(&gorm.Session{NewDB: true})

tx.First(&user)
// SELECT * FROM users ORDER BY id LIMIT 1

tx.First(&user, "id = ?", 10)
// SELECT * FROM users WHERE id = 10 ORDER BY id

// 不带 `NewDB` 选项
tx2 := db.Where("name = ?", "jinzhu").Session(&gorm.Session{})
tx2.First(&user)
// SELECT * FROM users WHERE name = "jinzhu" ORDER BY id
```

**跳过钩子**

如果您想跳过 `钩子` 方法，您可以使用 `SkipHooks` 会话模式，例如：

```go
DB.Session(&gorm.Session{SkipHooks: true}).Create(&user)

DB.Session(&gorm.Session{SkipHooks: true}).Create(&users)

DB.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)

DB.Session(&gorm.Session{SkipHooks: true}).Find(&user)

DB.Session(&gorm.Session{SkipHooks: true}).Delete(&user)

DB.Session(&gorm.Session{SkipHooks: true}).Model(User{}).Where("age > ?", 18).Updates(&user)
```

**AllowGlobalUpdate**

默认情况下，GORM 不允许全局 update/delete，它会返回 `ErrMissingWhereClause` 错误，你可以将该选项置为 true 以允许全局操作，例如：

```go
db.Session(&gorm.Session{
  AllowGlobalUpdate: true,
}).Model(&User{}).Update("name", "jinzhu")
// UPDATE users SET `name` = "jinzhu"
```

**FullSaveAssociations**

当创建 / 更新记录时，GORM 将使用 [Upsert](https://learnku.com/docs/gorm/v2/create#upsert) 自动保存关联和其引用。 如果您想要更新关联的数据，您应该使用 `FullSaveAssociations` 模式，例如：

```go
db.Session(&gorm.Session{FullSaveAssociations: true}).Updates(&user)
// ...
// INSERT INTO "addresses" (address1) VALUES ("Billing Address - Address 1"), ("Shipping Address - Address 1") ON DUPLICATE KEY SET address1=VALUES(address1);
// INSERT INTO "users" (name,billing_address_id,shipping_address_id) VALUES ("jinzhu", 1, 2);
// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu@example.com"), (111, "jinzhu-2@example.com") ON DUPLICATE KEY SET email=VALUES(email);
// ...
```

**Context**

通过 `Context` 选项，您可以传入 `Context` 来追踪 SQL 操作，例如：

```go
timeoutCtx, _ := context.WithTimeout(context.Background(), time.Second)
tx := db.Session(&Session{Context: timeoutCtx})

tx.First(&user) // 带 timeoutCtx 的查询
tx.Model(&user).Update("role", "admin") // 带 timeoutCtx 的更新
```

GORM 也提供了快捷调用方法 `WithContext`，其实现如下：

```go
func (db *DB) WithContext(ctx context.Context) *DB {
  return db.Session(&Session{Context: ctx})
}
```

**Logger**

Gorm 允许使用 `Logger` 选项自定义内建 Logger，例如：

```go
newLogger := logger.New(log.New(os.Stdout, "\r\n", log.LstdFlags),
              logger.Config{
                SlowThreshold: time.Second,
                LogLevel:      logger.Silent,
                Colorful:      false,
              })
db.Session(&Session{Logger: newLogger})

db.Session(&Session{Logger: logger.Default.LogMode(logger.Silent)})
```

**NowFunc**

`NowFunc` 允许改变 GORM 获取当前时间的实现，例如：

```go
db.Session(&Session{
  NowFunc: func() time.Time {
    return time.Now().Local()
  },
})
```

**Debug**

`Debug` 只是将会话的 `Logger` 修改为调试模式的快捷方法，其实现如下：

```go
func (db *DB) Debug() (tx *DB) {
  return db.Session(&Session{
    Logger:         db.Logger.LogMode(logger.Info),
  })
}
```



### 5.5 钩子

**对象生命周期**

Hook 是在创建、查询、更新、删除等操作之前、之后调用的函数。

如果您已经为模型定义了指定的方法，它会在创建、更新、查询、删除时自动被调用。如果任何回调返回错误，GORM 将停止后续的操作并回滚事务。

钩子方法的函数签名应该是 func(*gorm.DB) error

**Hook** 

**创建对象**

创建时可用的 hook

```go
// 开始事务
BeforeSave
BeforeCreate
// 关联前的 save
// 插入记录至 db
// 关联后的 save
AfterCreate
AfterSave
// 提交或回滚事务
```

代码示例：

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
  u.UUID = uuid.New()

  if !u.IsValid() {
    err = errors.New("can't save invalid data")
  }
  return
}

func (u *User) AfterCreate(tx *gorm.DB) (err error) {
  if u.ID == 1 {
    tx.Model(u).Update("role", "admin")
  }
  return
}
```

**注意** 在 GORM 中保存、删除操作会默认运行在事务上， 因此在事务完成之前该事务中所作的更改是不可见的，如果您的钩子返回了任何错误，则修改将被回滚。

```go
func (u *User) AfterCreate(tx *gorm.DB) (err error) {
  if !u.IsValid() {
    return errors.New("rollback invalid user")
  }
  return nil
}
```

**更新对象**

代码示例：

```go
func (u *User) BeforeUpdate(tx *gorm.DB) (err error) {
  if u.readonly() {
    err = errors.New("read only user")
  }
  return
}

// 在同一个事务中更新数据
func (u *User) AfterUpdate(tx *gorm.DB) (err error) {
  if u.Confirmed {
    tx.Model(&Address{}).Where("user_id = ?", u.ID).Update("verfied", true)
  }
  return
}
```

**删除对象**

```go
// 开始事务
BeforeDelete
// 删除 db 中的数据
AfterDelete
// 提交或回滚事务
```

代码示例：

```go
// 在同一个事务中更新数据
func (u *User) AfterDelete(tx *gorm.DB) (err error) {
  if u.Confirmed {
    tx.Model(&Address{}).Where("user_id = ?", u.ID).Update("invalid", false)
  }
  return // 回滚
}
```

**查询对象**

查询时可用的 hook

```php
// 从 db 中加载数据
// Preloading (eager loading)
AfterFind
```

代码示例：

```go
func (u *User) AfterFind(tx *gorm.DB) (err error) {
  if u.MemberShip == "" {
    u.MemberShip = "user"
  }
  return
}
```

**修改当前操作**

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
  // 通过 tx.Statement 修改当前操作，例如：
  tx.Statement.Select("Name", "Age")
  tx.Statement.AddClause(clause.OnConflict{DoNothing: true})

  // tx 是带有 `NewDB` 选项的新会话模式 
  // 基于 tx 的操作会在同一个事务中，但不会带上任何当前的条件
  err := tx.First(&role, "name = ?", user.Role).Error
  // SELECT * FROM roles WHERE name = "admin"
  // ...
  return err
}
```



### 5.6 事务

**禁用默认事务**

为了确保数据一致性，GORM 会在事务里执行写入操作（创建、更新、删除）。如果没有这方面的要求，您可以在初始化时禁用它，这将获得大约 30%+ 性能提升。

```go
// 全局禁用
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})

// 持续会话模式
tx := db.Session(&Session{SkipDefaultTransaction: true})
tx.First(&user, 1)
tx.Find(&users)
tx.Model(&user).Update("Age", 18)
```

**事务**

**要在事务中执行一系列操作，一般流程如下：**

```go
db.Transaction(func(tx *gorm.DB) error {
  // 在事务中执行一些 db 操作（从这里开始，您应该使用 'tx' 而不是 'db'）
  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
    // 返回任何错误都会回滚事务
    return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
    return err
  }

  // 返回 nil 提交事务
  return nil
})
```

**嵌套事务**

GORM 支持嵌套事务，您可以回滚较大事务内执行的一部分操作，例如：

```go
db.Transaction(func(tx *gorm.DB) error {
  tx.Create(&user1)

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx2.Create(&user2)
    return errors.New("rollback user2") // Rollback user2
  })

  tx.Transaction(func(tx2 *gorm.DB) error {
    tx2.Create(&user3)
    return nil
  })

  return nil
})

// Commit user1, user3
```

**手动事务**

```go
// 开始事务
tx := db.Begin()

// 在事务中执行一些 db 操作（从这里开始，您应该使用 'tx' 而不是 'db'）
tx.Create(...)

// ...

// 遇到错误时回滚事务
tx.Rollback()

// 否则，提交事务
tx.Commit()
```

一个特殊的示例：

```go
func CreateAnimals(db *gorm.DB) error {
  // 再唠叨一下，事务一旦开始，你就应该使用 tx 处理数据
  tx := db.Begin()
  defer func() {
    if r := recover(); r != nil {
      tx.Rollback()
    }
  }()

  if err := tx.Error; err != nil {
    return err
  }

  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
     tx.Rollback()
     return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
     tx.Rollback()
     return err
  }

  return tx.Commit().Error
}
```

**GORM 提供了 `SavePoint`、`Rollbackto` 来提供保存点以及回滚至保存点，例如：**

```go
tx := db.Begin()
tx.Create(&user1)

tx.SavePoint("sp1")
tx.Create(&user2)
tx.RollbackTo("sp1") // Rollback user2

tx.Commit() // Commit user1
```



### 5.7 数据库迁移

**AutoMigrate** 

AutoMigrate 用于自动迁移您的 schema，保持您的 schema 是最新的。

**注意：** AutoMigrate 会创建表，缺少的外键，约束，列和索引，并且会更改现有列的类型（如果其大小、精度、是否为空可更改）。但 **不会** 删除未使用的列，以保护您的数据。

```go
db.AutoMigrate(&User{})

db.AutoMigrate(&User{}, &Product{}, &Order{})

// 创建表时添加后缀
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```

> **注意** AutoMigrate 会自动创建数据库外键约束，您可以在初始化时禁用此功能，例如：
>
> ```go
> db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
>   DisableForeignKeyConstraintWhenMigrating: true,
> })
> ```

**Migrator 接口**

GORM 提供了 Migrator 接口，该接口为每个数据库提供了统一的 API 接口，可用来为您的数据库构建独立迁移，例如：

SQLite 不支持 ALTER COLUMN、DROP COLUMN，当你试图修改表结构，GORM 将创建一个新表、复制所有数据、删除旧表、重命名新表。

```go
type Migrator interface {
  // AutoMigrate
  AutoMigrate(dst ...interface{}) error

  // Database
  CurrentDatabase() string
  FullDataTypeOf(*schema.Field) clause.Expr

  // Tables
  CreateTable(dst ...interface{}) error
  DropTable(dst ...interface{}) error
  HasTable(dst interface{}) bool
  RenameTable(oldName, newName interface{}) error

  // Columns
  AddColumn(dst interface{}, field string) error
  DropColumn(dst interface{}, field string) error
  AlterColumn(dst interface{}, field string) error
  HasColumn(dst interface{}, field string) bool
  RenameColumn(dst interface{}, oldName, field string) error
  MigrateColumn(dst interface{}, field *schema.Field, columnType *sql.ColumnType) error
  ColumnTypes(dst interface{}) ([]*sql.ColumnType, error)

  // Constraints
  CreateConstraint(dst interface{}, name string) error
  DropConstraint(dst interface{}, name string) error
  HasConstraint(dst interface{}, name string) bool

  // Indexes
  CreateIndex(dst interface{}, name string) error
  DropIndex(dst interface{}, name string) error
  HasIndex(dst interface{}, name string) bool
  RenameIndex(dst interface{}, oldName, newName string) error
}
```

表相关操作：

```go
// 为 `User` 创建表
db.Migrator().CreateTable(&User{})

// 将 "ENGINE=InnoDB" 添加到创建 `User` 的 SQL 里去
db.Set("gorm:table_options", "ENGINE=InnoDB").CreateTable(&User{})

// 检查 `User` 对应的表是否存在
db.Migrator().HasTable(&User{})
db.Migrator().HasTable("users")

// 如果存在表则删除（删除时会忽略、删除外键约束)
db.Migrator().DropTable(&User{})
db.Migrator().DropTable("users")

// 重命名表
db.Migrator().RenameTable(&User{}, &UserInfo{})
db.Migrator().RenameTable("users", "user_infos")
```

列相关操作：

```go
type User struct {
  Name string
}

// 添加 name 字段
db.Migrator().AddColumn(&User{}, "Name")
// 删除 name 字段
db.Migrator().DropColumn(&User{}, "Name")
// 修改 name 字段
db.Migrator().AlterColumn(&User{}, "Name")
// 检查字段是否存在
db.Migrator().HasColumn(&User{}, "Name")

type User struct {
  Name    string
  NewName string
}

// 重命名字段
db.Migrator().RenameColumn(&User{}, "Name", "NewName")
db.Migrator().RenameColumn(&User{}, "name", "new_name")

// 获取字段类型
db.Migrator().ColumnTypes(&User{}) ([]*sql.ColumnType, error)
```

**约束：**

```go
type UserIndex struct {
  Name  string `gorm:"check:name_checker,name <> 'jinzhu'"`
}

// 创建约束
db.Migrator().CreateConstraint(&User{}, "name_checker")

// 删除约束
db.Migrator().DropConstraint(&User{}, "name_checker")

// 检查约束是否存在
db.Migrator().HasConstraint(&User{}, "name_checker")
```

**索引：**

```go
type User struct {
  gorm.Model
  Name string `gorm:"size:255;index:idx_name,unique"`
}

// 为 Name 字段创建索引
db.Migrator().CreateIndex(&User{}, "Name")
db.Migrator().CreateIndex(&User{}, "idx_name")

// 为 Name 字段删除索引
db.Migrator().DropIndex(&User{}, "Name")
db.Migrator().DropIndex(&User{}, "idx_name")

// 检查索引是否存在
db.Migrator().HasIndex(&User{}, "Name")
db.Migrator().HasIndex(&User{}, "idx_name")

type User struct {
  gorm.Model
  Name  string `gorm:"size:255;index:idx_name,unique"`
  Name2 string `gorm:"size:255;index:idx_name_2,unique"`
}
// 修改索引名
db.Migrator().RenameIndex(&User{}, "Name", "Name2")
db.Migrator().RenameIndex(&User{}, "idx_name", "idx_name_2")
```





### 5.8 记录日志

Gorm 有一个 [默认 logger 实现](https://github.com/go-gorm/gorm/blob/master/logger/logger.go)，默认情况下，它会打印慢 SQL 和错误

Logger 接受的选项不多，您可以在初始化时自定义它，例如：

```go
newLogger := logger.New(
  log.New(os.Stdout, "\r\n", log.LstdFlags), // io writer
  logger.Config{
    SlowThreshold: time.Second,   // 慢 SQL 阈值
    LogLevel:      logger.Silent, // Log level
    Colorful:      false,         // 禁用彩色打印
  },
)

// 全局模式
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  Logger: newLogger,
})

// 新建会话模式
tx := db.Session(&Session{Logger: newLogger})
tx.First(&user)
tx.Model(&user).Update("Age", 18)
```

GORM 定义了这些日志级别：`Silent`、`Error`、`Warn`、`Info`

```go
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{
  Logger: logger.Default.LogMode(logger.Silent),
})
```

Debug 单个操作，将当前操作的 log 级别调整为 logger.Info

```go
db.Debug().Where("name = ?", "jinzhu").First(&User{})
```

Logger 需要实现以下接口，它接受 `context`，所以你可以用它来追踪日志

```go
type Interface interface {
    LogMode(LogLevel) Interface
    Info(context.Context, string, ...interface{})
    Warn(context.Context, string, ...interface{})
    Error(context.Context, string, ...interface{})
    Trace(ctx context.Context, begin time.Time, fc func() (string, int64), err error)
}
```



### 5.9 通用数据库接口 sql.DB

GORM 提供了 `DB` 方法，可用于从当前 `*gorm.DB` 返回一个通用的数据库接口 [*sql.DB](https://pkg.go.dev/database/sql#DB)

```go
// 获取通用数据库对象 sql.DB，然后使用其提供的功能
sqlDB, err := db.DB()

// Ping
sqlDB.Ping()

// Close
sqlDB.Close()

// 返回数据库统计信息
sqlDB.Stats()
```

**注意** 如果底层连接的数据库不是 `*sql.DB`，它会返回错误

**连接池设置：**

```go
// 获取通用数据库对象 sql.DB ，然后使用其提供的功能
sqlDB, err := db.DB()

// SetMaxIdleConns 用于设置连接池中空闲连接的最大数量。
sqlDB.SetMaxIdleConns(10)

// SetMaxOpenConns 设置打开数据库连接的最大数量。
sqlDB.SetMaxOpenConns(100)

// SetConnMaxLifetime 设置了连接可复用的最大时间。
sqlDB.SetConnMaxLifetime(time.Hour)
```



### 5.10 查询范围

Scopes 使你可以复用通用的逻辑，共享的逻辑需要定义为 `func(*gorm.DB) *gorm.DB` 类型

**查询**

Scope 查询示例：

```go
func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
  return db.Where("amount > ?", 1000)
}

func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
  return db.Where("pay_mode_sign = ?", "C")
}

func PaidWithCod(db *gorm.DB) *gorm.DB {
  return db.Where("pay_mode_sign = ?", "C")
}

func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
  return func (db *gorm.DB) *gorm.DB {
    return db.Where("status IN (?)", status)
  }
}

db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)
// 查找所有金额大于 1000 的信用卡订单

db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
// 查找所有金额大于 1000 的 COD 订单

db.Scopes(AmountGreaterThan1000, OrderStatus([]string{"paid", "shipped"})).Find(&orders)
// 查找所有金额大于1000 的已付款或已发货订单
```

**分页**

```go
func Paginate(r *http.Request) func(db *gorm.DB) *gorm.DB {
  return func (db *gorm.DB) *gorm.DB {
    page, _ := strconv.Atoi(r.Query("page"))
    if page == 0 {
      page = 1
    }

    pageSize, _ := strconv.Atoi(r.Query("page_size"))
    switch {
    case pageSize > 100:
      pageSize = 100
    case pageSize <= 0:
      pageSize = 10
    }

    offset := (page - 1) * pageSize
    return db.Offset(offset).Limit(pageSize)
  }
}

db.Scopes(Paginate(r)).Find(&users)
db.Scopes(Paginate(r)).Find(&articles)
```

**更新**

Scope 更新、删除示例：

```go
func CurOrganization(r *http.Request) func(db *gorm.DB) *gorm.DB {
  return func (db *gorm.DB) *gorm.DB {
    org := r.Query("org")

    if org != "" {
      var organization Organization
      if db.Session(&Session{}).First(&organization, "name = ?", org).Error == nil {
        return db.Where("org_id = ?", org.ID)
      }
    }

    db.AddError("invalid organization")
    return db
  }
}

db.Model(&article).Scopes(CurOrganization(r)).Update("Name", "name 1")
// UPDATE articles SET name = "name 1" WHERE org_id = 111
db.Scopes(CurOrganization(r)).Delete(&Article{})
// DELETE FROM articles WHERE org_id = 111
```





