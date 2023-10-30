---
title: FlinkSQL - Operation
subtitle: 
date: 2023-07-15T16:24:09+08:00
draft: true
author: toxi
license:
toc: true
tags: ["flink",  "sql"]
blog: ["code"]
categories: ["Code"]


# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

<!--more-->
SQL验证完成后，需要将SQL语法树转换成Operation

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