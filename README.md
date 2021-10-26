# Sequelize 总结

## Why Sequelize？
Sequelize 是一个基于 promise 的 Node.js ORM (Object-Relational Mapping，把关系数据库的表结构映射到对象上)  
目前支持 Postgres, MySQL, MariaDB, SQLite 以及 Microsoft SQL Server.   
它具有强大的事务支持, 关联关系, 预读和延迟加载,读取复制等功能。  
Sequelize 遵从 语义版本控制。 支持 Node v10 及更高版本以便使用 ES6 功能。  

## 文档

### English (OFFICIAL)
https://sequelize.org/master

### 中文文档 (UNOFFICIAL)
https://github.com/demopark/sequelize-docs-Zh-CN

## Installation

$ npm i sequelize # This will install v6

### And one of the following:

$ npm i pg pg-hstore # Postgres  
$ npm i mysql2  
$ npm i mariadb  
$ npm i sqlite3  
$ npm i tedious # Microsoft SQL Server  

## 术语

#### 模型
模型是 Sequelize 的本质. 模型是代表数据库中表的抽象. 在 Sequelize 中,它是一个 Model 的扩展类.
#### 模型实例
模型是 ES6 类. 类的实例表示该模型中的一个对象(该对象映射到数据库中表的一行).
#### 关联
模型（表）间的关联关系。标准关联关系: 一对一, 一对多 和 多对多.

## 配置

```typescript
const { host, port, user, password, database, schema, pool }: dbConfig =
  config.get("dbConfig");
const sequelize = new Sequelize.Sequelize(database, user, password, {
  host: host, //数据库host，默认localhost
  port: port, //mysql默认端口3306
  dialect: "mysql", //使用的数据库，默认mysql
  timezone: "+08:00", //时区, 北京时间，东八区
  //模型定义的默认选项
  define: {
    charset: "utf8mb4", //字符编码， utf8mb4 是 utf8 的超集，能够用四个字节存储更多的字符
    collate: "utf8mb4_general_ci", //字符排序或比较大小，比utf8mb4_unicode_ci相对快
    underscored: true, //以下划线命名字段
    freezeTableName: true, //冻结表名，否则会自动变复数
    schema: schema, //schema（数据库对象集合）名
    timestamps: false, //默认true，false时不添加时间戳属性 (updatedAt, createdAt)
    paranoid: true, // paranoid 只有在启用时间戳时才能工作..使用方法参见“偏执表”。
    version: true, //是否启用乐观锁定。默认false。设置为true或具有要用于启用的属性名称的字符串。
  },
  pool: {
    min: pool.min, //最小连接数
    max: pool.max, //最大连接数
    acquire: pool.acquire, //该池在抛出错误之前尝试获取连接的最长时间（以毫秒为单位）
    idle: pool.idle, //连接被释放前可以空闲的最长时间，以毫秒为单位
  },
  logQueryParameters: process.env.NODE_ENV === "development", //是否在log中显示参数
  logging: (query, time) => {
    logger.info(time + "ms" + " " + query);
  },
  benchmark: true, //上面logging第二个参数传时间（毫秒）
});
```

### 测试连接

你可以使用 .authenticate() 函数测试连接是否正常：

```typescript
try {
  await sequelize.authenticate();
  console.log("Connection has been established successfully.");
} catch (error) {
  console.error("Unable to connect to the database:", error);
}
```

## 模型

### 模型定义

#### define

```typescript
const { Sequelize, DataTypes } = require("sequelize");
const sequelize = new Sequelize("sqlite::memory:");

const User = sequelize.define(
  "User",
  {
    // 在这里定义模型属性
    firstName: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    lastName: {
      type: DataTypes.STRING,
      // allowNull 默认为 true
    },
  },
  {
    // 这是其他模型参数
  }
);
```

#### init

```typescript
const { Sequelize, DataTypes, Model } = require("sequelize");
const sequelize = new Sequelize("sqlite::memory");

class User extends Model {}

User.init(
  {
    // 在这里定义模型属性
    firstName: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    lastName: {
      type: DataTypes.STRING,
      // allowNull 默认为 true
    },
  },
  {
    // 这是其他模型参数
    sequelize, // 我们需要传递连接实例
    modelName: "User", // 我们需要选择模型名称
  }
);
```

### 模型同步

- `User.sync()` - 如果表不存在,则创建该表(如果已经存在,则不执行任何操作)
- `User.sync({ force: true })` - 将创建表,如果表已经存在,则将其首先删除
- `User.sync({ alter: true })` - 这将检查数据库中表的当前状态(它具有哪些列,它们的数据类型等),然后在表中进行必要的更改以使其与模型匹配.

## 关联

Sequelize 提供了 **四种** 关联类型,并将它们组合起来以创建关联：

- `HasOne` 关联类型
- `BelongsTo` 关联类型
- `HasMany` 关联类型
- `BelongsToMany` 关联类型

```typescript
const A = sequelize.define("A" /* ... */);
const B = sequelize.define("B" /* ... */);

A.hasOne(B); // A 有一个 B
A.belongsTo(B); // A 属于 B
A.hasMany(B); // A 有多个 B
A.belongsToMany(B, { through: "C" }); // A 属于多个 B , 通过联结表 C
```

关联的定义顺序是有关系的. 换句话说,对于这四种情况,定义顺序很重要. 在上述所有示例中,A 称为 源 模型,而 B 称为 目标 模型. 此术语很重要.

- A.hasOne(B) 关联意味着 A 和 B 之间存在一对一的关系,外键在目标模型(B)中定义.
- A.belongsTo(B)关联意味着 A 和 B 之间存在一对一的关系,外键在源模型中定义(A).
- A.hasMany(B) 关联意味着 A 和 B 之间存在一对多关系,外键在目标模型(B)中定义.
- A.belongsToMany(B, { through: 'C' }) 关联意味着将表 C 用作联结表,在 A 和 B 之间存在多对多关系. 具有外键(例如,aId 和 bId). Sequelize 将自动创建此模型 C(除非已经存在),并在其上定义适当的外键.

### 强制性与可选性关联

默认情况下,该关联被视为可选. 换句话说,在我们的示例中,fooId 允许为空,这意味着一个 Bar 可以不存在 Foo 而存在. 只需在外键选项中指定 allowNull: false 即可更改此设置：

```typescript
Foo.hasOne(Bar, {
  foreignKey: {
    allowNull: false,
  },
});
// "fooId" INTEGER NOT NULL REFERENCES "foos" ("id") ON DELETE RESTRICT ON UPDATE RESTRICT
```

### 关联查询

#### 别名

```typescript
products.findAll({
  attributes: ["prdName", "price"],
  include: [
    {
      model: user,
      as: "u",
      attributes: ["userName"],
    },
  ],
});

//result
// [
//   {
//     prdName: "ipad",
//     price: 4.99,
//     u: { userName: "张三" },
//   },
// ];
```

#### 原始查询

```typescript
products.findAll({
  attributes: ["prdName", "price"],
  include: [
    {
      model: user,
      as: "u",
      attributes: ["userName"],
    },
  ],
  raw: true,
});

//result
// [
//   {
//     prdName: "ipad",
//     price: 4.99,
//     "u.userName": "张三",
//   },
// ];
```

#### 合并数据

```typescript
products.findAll({
  attributes: [Sequelize.col("u.userName"), "prdName", "price"],
  include: [
    {
      model: user,
      as: "u",
      attributes: [],
    },
  ],
  raw: true,
});

//result
// [
//   {
//     prdName: "ipad",
//     price: 4.99,
//     userName: "张三",
//   },
// ];
```

#### where条件

```typescript
products.findAll({
  attributes: [Sequelize.col("u.userName"), "prdName", "price"],
  include: [
    {
      model: user,
      as: "u",
      attributes: [],
    },
  ],
  where: {
    prdName: "ipad",
    "$u.userId$": 1,
  },
  raw: true,
});

//result
// [
//   {
//     prdName: "ipad",
//     price: 4.99,
//     userName: "张三",
//   },
// ];
```

#### 多表嵌套

```typescript
sequelize.sync().then(function () {
  User.findAll({
    attributes: [[Sequelize.col("User.name"), "username"]],
    include: {
      model: Foo,
      attributes: [[Sequelize.col("User->Foo.name"), "fooname"]],
    },
  });
});
```

## 事务

默认情况下,Sequelize 不使用[事务](https://en.wikipedia.org/wiki/Database_transaction). 但是,对于 Sequelize 的生产环境使用,你绝对应该将 Sequelize 配置为使用事务.

Sequelize 支持两种使用事务的方式：

1. 非托管事务:提交和回滚事务应由用户手动完成(通过调用适当的 Sequelize 方法).
2. 托管事务: 如果引发任何错误,Sequelize 将自动回滚事务,否则将提交事务. 另外,如果启用了 CLS(连续本地存储),则事务回调中的所有查询将自动接收事务对象.

### 非托管事务

非托管事务方法要求你在必要时手动提交和回滚事务.

```typescript
// 首先,我们开始一个事务并将其保存到变量中
const t = await sequelize.transaction();
try {
  // 然后,我们进行一些调用以将此事务作为参数传递:
  const user = await User.create(
    {
      firstName: "Bart",
      lastName: "Simpson",
    },
    { transaction: t }
  );
  await user.addSibling(
    {
      firstName: "Lisa",
      lastName: "Simpson",
    },
    { transaction: t }
  );
  // 如果执行到此行,且没有引发任何错误.
  // 我们提交事务.
  await t.commit();
} catch (error) {
  // 如果执行到达此行,则抛出错误.
  // 我们回滚事务.
  await t.rollback();
}
```

### 托管事务

托管事务会自动处理提交或回滚事务. 通过将回调传递给 sequelize.transaction 来启动托管事务. 这个回调可以是 async(通常是)的.

```typescript
try {
  const result = await sequelize.transaction(async (t) => {
    const user = await User.create(
      {
        firstName: "Abraham",
        lastName: "Lincoln",
      },
      { transaction: t }
    );
    await user.setShooter(
      {
        firstName: "John",
        lastName: "Boothe",
      },
      { transaction: t }
    );
    return user;
  });
  // 如果执行到此行,则表示事务已成功提交,`result`是事务返回的结果
  // `result` 就是从事务回调中返回的结果(在这种情况下为 `user`)
} catch (error) {
  // 如果执行到此,则发生错误.
  // 该事务已由 Sequelize 自动回滚！
}
```

### 自动将事务传递给所有查询

要将事务自动传递给所有查询,你必须安装 cls-hooked (CLS) 模块,并在自己的代码中实例化命名空间：

```typescript
const cls = require("cls-hooked");
const namespace = cls.createNamespace("my-very-own-namespace");
```

要启用 CLS,你必须通过使用 sequelize 构造函数的静态方法来告诉 sequelize 使用哪个命名空间：

```typescript
const Sequelize = require('sequelize');
Sequelize.useCLS(namespace);

new Sequelize(....);
```

注意,useCLS() 方法在 构建器 上,而不在 sequelize 实例上. 这意味着所有实例将共享相同的命名空间,并且 CLS 是全有或全无 - 你不能仅对某些实例启用它.

```typescript
sequelize.transaction((t1) => {
  // 启用 CLS 后,将在事务内部创建用户
  return User.create({ name: "Alice" });
});
```

## 偏执表

Sequelize 支持 _paranoid_ 表的概念. 一个 _paranoid_ 表是一个被告知删除记录时不会真正删除它的表.反而一个名为 `deletedAt` 的特殊列会将其值设置为该删除请求的时间戳.

这意味着偏执表会执行记录的 _软删除_,而不是 _硬删除_.

要定义 paranoid 模型,必须将 `paranoid: true` 参数传递给模型定义. Paranoid 需要时间戳才能起作用(即,如果你传递 `timestamps: false` 了,paranoid 将不起作用).

你还可以将默认的列名(默认是 `deletedAt`)更改为其他名称.

```typescript
class Post extends Model {}
Post.init(
  {
    /* 这是属性 */
  },
  {
    sequelize,
    paranoid: true,
    // 如果要为 deletedAt 列指定自定义名称
    deletedAt: "destroyTime",
  }
);
```

如果你确实想要硬删除,并且模型是 paranoid,则可以使用 force: true 参数强制执行：

```typescript
await Post.destroy({
  where: {
    id: 1,
  },
  force: true,
});
// DELETE FROM "posts" WHERE "id" = 1
```

要恢复软删除的记录,可以使用 restore 方法,该方法在静态版本和实例版本中都提供：

```typescript
// 展示实例 `restore` 方法的示例
// 我们创建一个帖子,对其进行软删除,然后将其还原
const post = await Post.create({ title: "test" });
console.log(post instanceof Post); // true
await post.destroy();
console.log("soft-deleted!");
await post.restore();
console.log("restored!");

// 展示静态 `restore` 方法的示例.
// 恢复每个 likes 大于 100 的软删除的帖子
await Post.restore({
  where: {
    likes: {
      [Op.gt]: 100,
    },
  },
});
```

Sequelize 执行的每个查询将自动忽略软删除的记录(当然,原始查询除外).
例如,findAll,findByPk,结果也将是 null,就好像该记录不存在一样.
如果你真的想让查询看到被软删除的记录,可以将 paranoid: false 参数传递给查询方法. 例如：

```typescript
await Post.findByPk(123); // 如果 ID 123 的记录被软删除,则将返回 `null`
await Post.findByPk(123, { paranoid: false }); // 这将检索记录

await Post.findAll({
  where: { foo: "bar" },
}); // 这将不会检索软删除的记录

await Post.findAll({
  where: { foo: "bar" },
  paranoid: false,
}); // 这还将检索软删除的记录
```

## 乐观锁

乐观锁定允许并发访问模型记录以进行编辑,并防止冲突覆盖数据. 它通过检查自从读取以来另一个进程是否对记录进行了更改,并在检测到冲突时抛出 OptimisticLockError 来执行此操作.

乐观锁定默认情况下处于禁用状态,可以通过在特定模型定义或全局模型配置中将 `version` 属性设置为 true 来启用.
