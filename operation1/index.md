# FlinkSQL - Operation


<!--more-->
SQL验证完成后，需要将SQL语法树转换成Operation的集合

```java
public List<Operation> parse(String statement) {
    ...
        return Collections.singletonList(
                SqlNodeToOperationConversion.convert(planner, catalogManager, parsed.get(0))
                        .orElseThrow(() -> new TableException("Unsupported query: " + statement)));
    }
```
在这里将已经验证好的SQL 转换成Oper的过程，就会使用到SqlNodeToOperationConversion.convert 


## SqlNodeToOperationConversion.convert 

```java
    public static Optional<Operation> convert(
            FlinkPlannerImpl flinkPlanner, CatalogManager catalogManager, SqlNode sqlNode) {
        // validate the query
        final SqlNode validated = flinkPlanner.validate(sqlNode);
        return convertValidatedSqlNode(flinkPlanner, catalogManager, validated);
    }
```

convertValidatedSqlNode 就是转换Operation的入口，一共有三个参数
- flinkPlanner: FlinkPlannerImpl 优化器，将SQL语法树生成flink的执行计划
- catalogManager: CatalogManager 当前作业的库表等信息的管理
- validated: SqlNode  已经检验好的SQL语法树


### convert 参数
#### FlinkPlannerImpl
在实际运行中, flinkPlanner 是通过 FlinkPlannerImpl 实例实现的，具体是在 PlannerContext进行实现的

```java
package org.apache.flink.table.planner.delegation;

public class PlannerContext {
...
public FlinkPlannerImpl createFlinkPlanner() {
        return new FlinkPlannerImpl(
                createFrameworkConfig(), this::createCatalogReader, typeFactory, cluster);
    }
...
}
```
- 一套flink 使用的schema，类型系统以及SQL优化器等组件的framework配置
- 一个读取flink的catalog的入口实例
- 生成flink类型的工厂实例
- 专门为Flink服务的 calcite 关系优化cluster 

planner 的作用就是用于将 FlinkSQL 转换成Flink 可以识别的DAG作业，运行到Flink 环境中

#### CatalogManager

CatalogManager 隐藏的很深，在最初创建的时候他就已经生成了。如果没有特殊要求，catalog  回创建一个名为 "default_catalog" 的catalog，在此之下，还会创建一个 名为 “default_database”的数据库  
通常情况下，这里的catalog是不需要我们修改的，在flink UI 界面里我们也经常可以看到，所有的表都是放在这个“default_catlog.default_database"下的
流式任务的创建的代码是在 org.apache.flink.table.api.bridge.scala.internal.StreamTableEnvironmentImpl 下的
```scala
package org.apache.flink.table.api.bridge.scala.internal
...
object StreamTableEnvironmentImpl {
    ...
    val catalogManager = CatalogManager.newBuilder
      .classLoader(userClassLoader)
      .config(tableConfig)
      .defaultCatalog(
        settings.getBuiltInCatalogName,
        new GenericInMemoryCatalog(settings.getBuiltInCatalogName, settings.getBuiltInDatabaseName))
      .executionConfig(executionEnvironment.getConfig)
      .catalogModificationListeners(TableFactoryUtil
        .findCatalogModificationListenerList(tableConfig.getConfiguration, userClassLoader))
      .catalogStoreHolder(
        CatalogStoreHolder
          .newBuilder()
          .catalogStore(catalogStore)
          .factory(catalogStoreFactory)
          .config(tableConfig)
          .classloader(userClassLoader)
          .build())
      .build
    ...
}

```
这里的所有参数都是默认值

## SQL验证
写入的SQL语句 会先进行验证，通过Flink自己写的Planner，设置自己的规则然后借助Calicte 进行验证  

> 具体验证过程: [FlinkSQL - 验证过程]("/flink-sql-validate/#validate")

## convertValidatedSqlNode

从这里开始SqlNode进入后就要转换成Operation了  
```java
package org.apache.flink.table.planner.operations;
public class SqlNodeToOperationConversion {
...
    private static Optional<Operation> convertValidatedSqlNode(
            FlinkPlannerImpl flinkPlanner, CatalogManager catalogManager, SqlNode validated) {
        ...
    }
...
}
```
该方法接收3个参数：
- flinkPlanner: 就是从上层传进来的执行计划
- catalogManager: 用来管理数据的schema
- validated: 验证后的Sql语句

```java
    private static Optional<Operation> convertValidatedSqlNode(
            FlinkPlannerImpl flinkPlanner, CatalogManager catalogManager, SqlNode validated) {
        beforeConversion();
        // delegate conversion to the registered converters first
        SqlNodeConvertContext context = new SqlNodeConvertContext(flinkPlanner, catalogManager);
        Optional<Operation> operation = SqlNodeConverters.convertSqlNode(validated, context);
        if (operation.isPresent()) {
            return operation;
        }
        ...
}
```
首先做一些前置工作  
1. beforeConversion： 清除行级修改上下文，在普通的SQL作业中这一步没什么变化
2. 如果SQL之前注册过，那么就返回之前的SQL，这里因为第一次启动，之前也没有任务，所以也不会直接返回



## Q&A 
1. 为什么 parse 方法返回的是 Operation的集合呢？

---

> 作者: toxi  
> URL: https://example.com/operation1/  

