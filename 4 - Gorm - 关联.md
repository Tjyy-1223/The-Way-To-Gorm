# 第四章 - Gorm - 关联

参考资料：

+ [Gorm 指南 - 中文](https://learnku.com/docs/gorm/v2/index/9728)
+ [GORM 中文文档](https://jasperxu.github.io/gorm-zh/)



#### 4.1 属于

belongs to 会与另一个模型建立了一对一的连接。 这种模型的每一个实例都 “属于” 另一个模型的一个实例。

例如，您的应用包含 user 和 company，并且每个 user 都可以分配给一个 company

```go
// `User` 属于 `Company`，`CompanyID` 是外键
type User struct {
  gorm.Model
  Name      string
  CompanyID int
  Company   Company
}

type Company struct {
  ID   int
  Name string
}
```

要定义一个 belongs to 关系，必须存在外键，默认的外键使用拥有者的类型名加上主字段名

对于上面例子，定义属于 `Company` 的 `User`，其外键一般是 `CompanyID`

此外，GORM 还提供了一种自定义外键的方法，例如：

```go
type User struct {
  gorm.Model
  Name         string
  CompanyRefer int
  Company      Company `gorm:"foreignKey:CompanyRefer"`
  // 使用 CompanyRefer 作为外键
}

type Company struct {
  ID   int
  Name string
}
```

对于 belongs to 关系，GORM 通常使用拥有者的主字段作为外键的值。 对于上面的例子，它是 Company 的 ID 字段

当您将 user 分配给某个 company 时，GORM 会将 company 的 ID 保存到用户的 CompanyID 字段

此外，您也可以使用标签 references 手动更改它，例如：

```go
type User struct {
  gorm.Model
  Name      string
  CompanyID string
  Company   Company `gorm:"references:Code"` // 使用 Code 作为引用
}

type Company struct {
  ID   int
  Code string
  Name string
}
```

你可以通过为标签 `constraint` 配置 `OnUpdate`、`OnDelete` 实现外键约束，在使用 GORM 进行迁移时它会被创建，例如：

```go
type User struct {
  gorm.Model
  Name      string
  CompanyID int
  Company   Company `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL;"`
}

type Company struct {
  ID   int
  Name string
}
```



#### 4.2 包含一个

has one 与另一个模型建立一对一的关联，但它和一对一关系有些许不同。 这种关联表明一个模型的每个实例都包含或拥有另一个模型的一个实例。

例如，您的应用包含 user 和 credit card 模型，且每个 user 只能有一张 credit card。

```go
// User 有一张 CreditCard，CreditCardID 是外键
type User struct {
  gorm.Model
  CreditCard CreditCard
}

type CreditCard struct {
  gorm.Model
  Number string
  UserID uint
}
```

对于 has one 关系，同样必须存在外键字段。拥有者将把属于它的模型的主键保存到这个字段。

这个字段的名称通常由 has one 模型的类型加上其 主键 生成，对于上面的例子，它是 UserID。

为 user 添加 credit card 时，它会将 user 的 ID 保存到自己的 UserID 字段。

如果你想要使用另一个字段来保存该关系，你同样可以使用标签 foreignKey 来更改它，例如：

```go
type User struct {
  gorm.Model
  CreditCard CreditCard `gorm:"foreignKey:UserName"`
  // 使用 UserName 作为外键
}

type CreditCard struct {
  gorm.Model
  Number   string
  UserName string
}
```

默认情况下，拥有者实体会将 has one 对应模型的主键保存为外键，您也可以修改它，用另一个字段来保存，例如下个这个使用 Name 来保存的例子。

您可以使用标签 references 来更改它，例如：

```go
type User struct {
  gorm.Model
  Name       string     `gorm:"index"`
  CreditCard CreditCard `gorm:"foreignkey:UserName;references:name"`
}

type CreditCard struct {
  gorm.Model
  Number   string
  UserName string
}
```

##### 4.2.1 实践

```go
type Cat struct {
  ID    int
  Name  string
  Toy   Toy `gorm:"polymorphic:Owner;"`
}

type Dog struct {
  ID   int
  Name string
  Toy  Toy `gorm:"polymorphic:Owner;"`
}

type Toy struct {
  ID        int
  Name      string
  OwnerID   int
  OwnerType string
}

db.Create(&Dog{Name: "dog1", Toy: Toy{Name: "toy1"}})
// INSERT INTO `dogs` (`name`) VALUES ("dog1")
// INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","dogs")
```

您可以使用标签 `polymorphicValue` 来更改多态类型的值，例如：

```go
type Dog struct {
  ID   int
  Name string
  Toy  Toy `gorm:"polymorphic:Owner;polymorphicValue:master"`
}

type Toy struct {
  ID        int
  Name      string
  OwnerID   int
  OwnerType string
}

db.Create(&Dog{Name: "dog1", Toy: Toy{Name: "toy1"}})
// INSERT INTO `dogs` (`name`) VALUES ("dog1")
// INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","master")
```



#### 4.3 包含多个

has many 与另一个模型建立了一对多的连接。 不同于 has one，拥有者可以有零或多个关联模型。

例如，您的应用包含 user 和 credit card 模型，且每个 user 可以有多张 credit card。

```go
// User 有多张 CreditCard，UserID 是外键
type User struct {
  gorm.Model
  CreditCards []CreditCard
}

type CreditCard struct {
  gorm.Model
  Number string
  UserID uint
}
```

要定义 has many 关系，同样必须存在外键。 默认的外键名是拥有者的类型名加上其主键字段名

例如，要定义一个属于 User 的模型，则其外键应该是 UserID。

此外，想要使用另一个字段作为外键，您可以使用 foreignKey 标签自定义它：

```go
type User struct {
  gorm.Model
  CreditCards []CreditCard `gorm:"foreignKey:UserRefer"`
}

type CreditCard struct {
  gorm.Model
  Number    string
  UserRefer uint
}
```

GORM 通常使用拥有者的主键作为外键的值。 对于上面的例子，它是 User 的 ID 字段。

为 user 添加 credit card 时，GORM 会将 user 的 ID 字段保存到 credit card 的 UserID 字段。

同样的，您也可以使用标签 references 来更改它，例如：

```go
type User struct {
  gorm.Model
  MemberNumber string
  CreditCards  []CreditCard `gorm:"foreignKey:UserNumber;references:MemberNumber"`
}

type CreditCard struct {
  gorm.Model
  Number     string
  UserNumber string
}
```

##### 4.3.1 实践

GORM 为 `has one` 和 `has many` 提供了多态关联支持，它会将拥有者实体的表名、主键值都保存到多态类型的字段中。

```go
type Dog struct {
  ID   int
  Name string
  Toys []Toy `gorm:"polymorphic:Owner;"`
}

type Toy struct {
  ID        int
  Name      string
  OwnerID   int
  OwnerType string
}

db.Create(&Dog{Name: "dog1", Toy: []Toy{{Name: "toy1"}, {Name: "toy2"}}})
// INSERT INTO `dogs` (`name`) VALUES ("dog1")
// INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","dogs"), ("toy2","1","dogs")
```

您可以使用标签 `polymorphicValue` 来更改多态类型的值，例如：

```go
type Dog struct {
  ID   int
  Name string
  Toys []Toy `gorm:"polymorphic:Owner;polymorphicValue:master"`
}

type Toy struct {
  ID        int
  Name      string
  OwnerID   int
  OwnerType string
}

db.Create(&Dog{Name: "dog1", Toy: []Toy{{Name: "toy1"}, {Name: "toy2"}}})
// INSERT INTO `dogs` (`name`) VALUES ("dog1")
// INSERT INTO `toys` (`name`,`owner_id`,`owner_type`) VALUES ("toy1","1","master"), ("toy2","1","master")
```



#### 4.4 多对多

Many to Many 会在两个 model 中添加一张连接表。

例如，您的应用包含了 user 和 language，且一个 user 可以说多种 language，多个 user 也可以说一种 language。

```go
// User 拥有并属于多种 language，`user_languages` 是连接表
type User struct {
  gorm.Model
  Languages []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
  gorm.Model
  Name string
}
```

当使用 GORM 的 `AutoMigrate` 为 `User` 创建表时，GORM 会自动创建连接表

```go
// User 拥有并属于多种 language，`user_languages` 是连接表
type User struct {
  gorm.Model
  Languages []*Language `gorm:"many2many:user_languages;"`
}

type Language struct {
  gorm.Model
  Name string
  Users []*User `gorm:"many2many:user_languages;"`
}
```

对于 `many2many` 关系，连接表会同时拥有两个模型的外键，例如：

```go
type User struct {
  gorm.Model
  Languages []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
  gorm.Model
  Name string
}

// Join Table: user_languages
//   foreign key: user_id, reference: users.id
//   foreign key: language_id, reference: languages.id
```

若要重写它们，可以使用标签 foreignKey、references、joinforeignKey、joinReferences。当然，您不需要使用全部的标签，你可以仅使用其中的一个重写部分的外键、引用。

```go
type User struct {
    gorm.Model
    Profiles []Profile `gorm:"many2many:user_profiles;foreignKey:Refer;joinForeignKey:UserReferID;References:UserRefer;joinReferences:ProfileRefer"`
    Refer    uint      `gorm:"index:,unique"`
}

type Profile struct {
    gorm.Model
    Name      string
    UserRefer uint `gorm:"index:,unique"`
}

// 这会创建连接表：user_profiles
//   外键: user_refer_id, 引用: users.refer
//   外键: profile_refer, 引用: profiles.user_refer
```



