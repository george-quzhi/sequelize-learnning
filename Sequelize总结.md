# Sequelize 总结

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
    version: true, //是否启用乐观锁定。启用时，sequelize将向模型添加版本计数属性,并在保存过时的实例时引发OptimisticLockingError错误。设置为true或具有要用于启用的属性名称的字符串。默认false。
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

## 事务

默认情况下,Sequelize 不使用[事务](https://en.wikipedia.org/wiki/Database_transaction). 但是,对于 Sequelize 的生产环境使用,你绝对应该将 Sequelize 配置为使用事务.

Sequelize 支持两种使用事务的方式：

1. 非托管事务:提交和回滚事务应由用户手动完成(通过调用适当的 Sequelize 方法).
2. **托管事务**: 如果引发任何错误,Sequelize 将自动回滚事务,否则将提交事务. 另外,如果启用了 CLS(连续本地存储),则事务回调中的所有查询将自动接收事务对象.

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
    id: 1
  },
  force: true
});
// DELETE FROM "posts" WHERE "id" = 1
```

要恢复软删除的记录,可以使用 restore 方法,该方法在静态版本和实例版本中都提供：

```typescript
// 展示实例 `restore` 方法的示例
// 我们创建一个帖子,对其进行软删除,然后将其还原
const post = await Post.create({ title: 'test' });
console.log(post instanceof Post); // true
await post.destroy();
console.log('soft-deleted!');
await post.restore();
console.log('restored!');

// 展示静态 `restore` 方法的示例.
// 恢复每个 likes 大于 100 的软删除的帖子
await Post.restore({
  where: {
    likes: {
      [Op.gt]: 100
    }
  }
});
```

Sequelize 执行的每个查询将自动忽略软删除的记录(当然,原始查询除外).
例如,findAll,findByPk,结果也将是 null,就好像该记录不存在一样.
如果你真的想让查询看到被软删除的记录,可以将 paranoid: false 参数传递给查询方法. 例如：

```typescript
await Post.findByPk(123); // 如果 ID 123 的记录被软删除,则将返回 `null`
await Post.findByPk(123, { paranoid: false }); // 这将检索记录

await Post.findAll({
  where: { foo: 'bar' }
}); // 这将不会检索软删除的记录

await Post.findAll({
  where: { foo: 'bar' },
  paranoid: false
}); // 这还将检索软删除的记录
```