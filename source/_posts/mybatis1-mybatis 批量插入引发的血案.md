id: mybatis1
title: Mybatis 批量插入引发的血案
categories: web java
tags: [db,mybatis,java]
---

## 故事

今天下午公司技术分享，一个伙伴提到他踩过坑：mybatis批量插入时动态sql允许的最大参数数量是2100个。即下面代码中“#{...}”的数量。

```xml
<insert id="batchInsert" parameterType="list">
  insert into Adv_permeability values
  <foreach collection="permeabilityList" separator="," item="permeability">
    (#{permeability.areaId}, #{permeability.areaName}, #{permeability.flag})
  </foreach>
</insert>
```

当时就有一个小伙伴不开心了，他明确表示自己曾经成功插入超过2100个参数，并当场拉出了代码证明之。由此猜测`mybatis`批量插入的上限是按**对象**数量限制的，不是**参数**，双方争执不下，一片欢乐。

## 探究

`mybatis` 真的对动态 **SQL** 的 **参数** 或者 **对象** 数量设定了限制么？如果有那么原因何在？如果没有，那么为什么双方都那么确定呢？难道是 **数据库** 不同导致的？

本着最小投入的原则，笔者开始了测试。手头现有 `SQLServer` 和 `Mysql` 就测这两个吧。脚本使用上述的简单脚本。

### SQLServer

对 `SQLServer` 进行2000次批量插入，实体参数是3个，即本次插入参数数量是 2000\*3=6000，结果是 **报错** 了！

> com.microsoft.sqlserver.jdbc.SQLServerException: 传入的请求具有过多的参数。该服务器支持最多 2100 个参数。请减少参数的数目，然后重新发送该请求。

可以清楚看到，错误是 `SQLServer` 的JDBC包抛出的。

不过，笔者记得在用 `SqlServer Management Studio` 批量插入式，最大条数有 **1000** 的限制， 那么在用 `Mybatis` 时还会有么？这里将插入参数变为一个 `{permeability.areaId}` ，插入 **1315** 条，结果 **报错** 了！

> com.microsoft.sqlserver.jdbc.SQLServerException: INSERT 语句中行值表达式的数目超出了 1000 行值的最大允许值。

果然，是 `SQLServer` 数据库自己的鬼，不是 `Mybatis` 的锅！

### Mysql

对 `Mysql` 进行同样的2000次批量插入，结果是完美执行。笔者一咬牙，来了次 200000 的亲密接触，结果 **报错** 了！

> com.mysql.jdbc.PacketTooBigException: Packet for query is too large (8346602 > 4194304). You can change this value on the server by setting the max_allowed_packet' variable.

看来 `Mysql` 也有限制，跟 `SQLServer` “数参数个数” 的限制方法不同， `Mysql` 对执行的SQL语句大小进行限制，相当于对字符串进行限制。根据报错内容，默认允许最大SQL是 **4M** 。这个错误是 `Mysql` 的JDBC包抛出的。

可以使用语句查询这个值并改变：

```shell
# 查询当前大小
mysql> select @@max_allowed_packet;
+----------------------+
| @@max_allowed_packet |
+----------------------+
|              4194304 |
+----------------------+
1 row in set (0.00 sec)

# 修改为256M
mysql> SET GLOBAL max_allowed_packet=268435456;
Query OK, 0 rows affected (0.00 sec)

# 退出客户端重新登录
mysql> select @@max_allowed_packet;
+----------------------+
| @@max_allowed_packet |
+----------------------+
|            268435456 |
+----------------------+
1 row in set (0.00 sec)
```

### Mybatis

简单看一下 `Mybatis` 解析动态SQL的源码。

```java
// 开始解析
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
        configurationElement(parser.evalNode("/mapper"));
        configuration.addLoadedResource(resource);
        bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
}
// 解析mapper
private void configurationElement(XNode context) {
    try {
        String namespace = context.getStringAttribute("namespace");
        if (namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        builderAssistant.setCurrentNamespace(namespace);
        cacheRefElement(context.evalNode("cache-ref"));
        cacheElement(context.evalNode("cache"));
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        sqlElement(context.evalNodes("/mapper/sql"));
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    }
}

// 创建 select|insert|update|delete 语句
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            configuration.addIncompleteStatement(statementParser);
        }
    }
}

// 填充参数，创建语句
public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
        boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
}
```

从开始到结束， `Mybatis` 都没有对填充的条数和参数的数量做限制。

## 结语
- `SqlServer` 对语句的条数和参数的数量都有限制，分别是 1000 和 2100。
- `Mysql`     对语句的长度有限制，默认是 4M。
- `Mybatis`   对动态语句没有数量上的限制。
