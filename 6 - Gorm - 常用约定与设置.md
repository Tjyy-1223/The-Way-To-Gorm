# 第六章 - Gorm - 常用约定与设置

参考资料：

+ [Gorm 指南 - 中文](https://learnku.com/docs/gorm/v2/index/9728)
+ [GORM 中文文档](https://jasperxu.github.io/gorm-zh/)

### 6.1 约定

#### 6.1.1 使用 ID 作为主键

默认情况下，GORM 会使用 `ID` 作为表的主键。

```go
type User struct {
  ID   string // 默认情况下，名为 `ID` 的字段会作为表的主键
  Name string
}
```

你可以通过标签 `primaryKey` 将其它字段设为主键

```go
// 将 `UUID` 设为主键
type Animal struct {
  ID     int64
  UUID   string `gorm:"primaryKey"`
  Name   string
  Age    int64
}
```

此外，您还可以看看 [复合主键](https://learnku.com/docs/gorm/v2/composite_primary_key)



#### 6.1.2 表名

GORM 使用结构体名的 `蛇形命名` 作为表名。对于结构体 `User`，根据约定，其表名为 `users`

您可以实现 `Tabler` 接口来更改默认表名，例如：

```go
type Tabler interface {
    TableName() string
}

// TableName 会将 User 的表名重写为 `profiles`
func (User) TableName() string {
  return "profiles"
}
```

**注意：** `TableName` 不支持动态变化，它会被缓存下来以便后续使用。想要使用动态表名，你可以使用 `Scopes`，例如：

```go
func UserTable(user User) func (tx *gorm.DB) *gorm.DB {
  return func (tx *gorm.DB) *gorm.DB {
    if user.Admin {
      return tx.Table("admin_users")
    }

    return tx.Table("users")
  }
}

db.Scopes(UserTable(user)).Create(&user)
```



#### 6.1.3 临时指定表名

您可以使用 `Table` 方法临时指定表名，例如：

```go
// 根据 User 的字段创建 `deleted_users` 表
db.Table("deleted_users").AutoMigrate(&User{})

// 从另一张表查询数据
var deletedUsers []User
db.Table("deleted_users").Find(&deletedUsers)
// SELECT * FROM deleted_users;

db.Table("deleted_users").Where("name = ?", "jinzhu").Delete(&User{})
// DELETE FROM deleted_users WHERE name = 'jinzhu';
```

查看 [from 子查询](https://learnku.com/docs/gorm/v2/advanced_query#from_subquery) 了解如何在 FROM 子句中使用子查询



#### 6.1.4 命名策略

GORM 允许用户通过覆盖默认的`命名策略`更改默认的命名约定，命名策略被用于构建：`TableName`、`ColumnName`、`JoinTableName`、`RelationshipFKName`、`CheckerName`、`IndexName`。查看 [GORM 配置](https://learnku.com/docs/gorm/v2/gorm_config#naming_strategy) 获取详情。



#### 6.1.5 列名

 根据约定，数据表的列名使用的是 struct 字段名的 `蛇形命名`

```go
type User struct {
  ID        uint      // 列名是 `id`
  Name      string    // 列名是 `name`
  Birthday  time.Time // 列名是 `birthday`
  CreatedAt time.Time // 列名是 `created_at`
}
```

您可以使用 `column` 标签或 [`命名策略`](https://learnku.com/docs/gorm/v2/conventions#naming_strategy) 来覆盖列名

```go
type Animal struct {
  AnimalID int64     `gorm:"column:beast_id"`         // 将列名设为 `beast_id`
  Birthday time.Time `gorm:"column:day_of_the_beast"` // 将列名设为 `day_of_the_beast`
  Age      int64     `gorm:"column:age_of_the_beast"` // 将列名设为 `age_of_the_beast`
}
```



#### 6.1.6 时间戳追踪

对于有 `CreatedAt` 字段的模型，创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

```go
db.Create(&user) // 将 `CreatedAt` 设为当前时间

user2 := User{Name: "jinzhu", CreatedAt: time.Now()}
db.Create(&user2) // user2 的 `CreatedAt` 不会被修改

// 想要修改该值，您可以使用 `Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

对于有 `UpdatedAt` 字段的模型，更新记录时，将该字段的值设为当前时间。创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

```go
db.Save(&user) // 将 `UpdatedAt` 设为当前时间

db.Model(&user).Update("name", "jinzhu") // 会将 `UpdatedAt` 设为当前时间

db.Model(&user).UpdateColumn("name", "jinzhu") // `UpdatedAt` 不会被修改

user2 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Create(&user2) // 创建记录时，user2 的 `UpdatedAt` 不会被修改

user3 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Save(&user3) // 更新世，user3 的 `UpdatedAt` 会修改为当前时间
```



### 6.2 设置

#### 6.2.1 表后缀

GORM 提供了 `Set`, `Get`, `InstanceSet`, `InstanceGet` 方法来允许用户传值给 [勾子](https://learnku.com/docs/gorm/v2/hooks) 或其他方法

Gorm 中有一些特性用到了这种机制，如迁移表时传递表选项。

```go
// 创建表时添加表后缀
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```



#### 6.2.2 Set/Get

使用 `Set` / `Get` 传递设置到钩子方法，例如：

```go
type User struct {
  gorm.Model
  CreditCard CreditCard
  // ...
}

func (u *User) BeforeCreate(tx *gorm.DB) error {
  myValue, ok := tx.Get("my_value")
  // ok => true
  // myValue => 123
}

type CreditCard struct {
  gorm.Model
  // ...
}

func (card *CreditCard) BeforeCreate(tx *gorm.DB) error {
  myValue, ok := tx.Get("my_value")
  // ok => true
  // myValue => 123
}

myValue := 123
db.Set("my_value", myValue).Create(&User{})
```



#### 6.2.3 InstanceSet / InstanceGet

使用 `InstanceSet` / `InstanceGet` 传递设置到 `*Statement` 的钩子方法，例如：

```go
type User struct {
  gorm.Model
  CreditCard CreditCard
  // ...
}

func (u *User) BeforeCreate(tx *gorm.DB) error {
  myValue, ok := tx.InstanceGet("my_value")
  // ok => true
  // myValue => 123
}

type CreditCard struct {
  gorm.Model
  // ...
}

// 在创建关联时，GORM 创建了一个新 `*Statement`，所以它不能读取到其它实例的设置
func (card *CreditCard) BeforeCreate(tx *gorm.DB) error {
  myValue, ok := tx.InstanceGet("my_value")
  // ok => false
  // myValue => nil
}

myValue := 123
db.InstanceSet("my_value", myValue).Create(&User{})
```

