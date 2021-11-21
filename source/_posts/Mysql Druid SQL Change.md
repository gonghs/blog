---
title: Mysql使用Druid访问器修改SQL
date: 2021-11-22 00:00:00
categories: 
- java
tags:
- java
- mysql
- druid
description: SQL的统一处理是我们在Java开发中经常需要用到的功能，阿里的Druid以访问器的形式为我们提供了一种可行性方案。
---

本文[相关测试代码](https://gitee.com/gonghs/blog-code/tree/master/druid-visitor)。

## 访问器介绍

Druid官方为我们提供了解析Sql为抽象AST（抽象语法树）的功能，简单的来说就是将sql解析为一个个节点对象，允许我们访问并操作修改。

下面以一条简单的mysql数据库sql语句为例：

```sql
select * from test where name = 'maple';
```

下图简单描述了解析后的节点细节，详细的对象介绍参照[官方wiki](https://github.com/alibaba/druid/wiki/Druid_SQL_AST)：

![节点树图](https://gitee.com/gonghs/image/raw/master/img/20201129143018.png)

可以看到语句被解析为select，from，where几个部分，并最终形成各类型的Expr对象，而druid访问器做的事情便是为我们提供了一种机制，用于访问以上的所有节点对象。

## 访问器示例

还是刚刚那条语句，我们使用一个简单的示例介绍druid访问器的使用。

访问器：

```java
@Getter
public class TestVisitor extends MySqlASTVisitorAdapter {
    private final List<String> tableNameList = new ArrayList<>();

    @Override
    public boolean visit(SQLExprTableSource x) {
        tableNameList.add(x.getName().getSimpleName());
        return super.visit(x);
    }
}
```

测试类：

```java
@Test
public void testSql() {
    String sql = "select * from test where name = 'maple';";
    List<SQLStatement> sqlStatements = SQLUtils.parseStatements(sql, JdbcConstants.MYSQL);
    SQLStatement sqlStatement = sqlStatements.get(0);
    TestVisitor testVisitor = new TestVisitor();
    sqlStatement.accept(testVisitor);
    System.out.println(testVisitor.getTableNameList());
}
/**
 * out: 
 * [test]
 */
```

上面的测试类我们使用访问器访问语树并存储了访问所得是所有表名，Druid为我们提供了很多visit方法的相关重载方法，利用这些方法我们可以访问到上节图示中除了SQLBinaryOperator之外的所有节点。

```java
@Getter
public class TestVisitor extends MySqlASTVisitorAdapter {
    private final List<String> tableNameList = new ArrayList<>();

    @Override
    public boolean visit(SQLSelectStatement x) {
        System.out.println("SQLSelectStatement");
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLSelect x) {
        System.out.println("SQLSelect");
        return super.visit(x);
    }

    @Override
    public boolean visit(MySqlSelectQueryBlock x) {
        System.out.println("MySqlSelectQueryBlock");
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLSelectItem x) {
        System.out.println("SQLSelectItem");
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLExprTableSource x) {
        System.out.println("SQLExprTableSource");
        tableNameList.add(x.getName().getSimpleName());
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLBinaryOpExpr x) {
        System.out.println("SQLBinaryOpExpr");
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLAllColumnExpr x) {
        System.out.println("SQLAllColumnExpr");
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLIdentifierExpr x) {
        System.out.println("SQLIdentifierExpr");
        return super.visit(x);
    }

    @Override
    public boolean visit(SQLCharExpr x) {
        System.out.println("SQLCharExpr");
        return super.visit(x);
    }
}
```

测试输出：

```java
/**
 * out:
 * SQLSelectStatement
 * SQLSelect
 * MySqlSelectQueryBlock
 * SQLSelectItem
 * SQLAllColumnExpr
 * SQLExprTableSource
 * SQLIdentifierExpr
 * SQLBinaryOpExpr
 * SQLIdentifierExpr
 * SQLCharExpr
 * [test]
 */
```

这里需要提及的是，visit的返回值代表访问器是否还需要继续向下遍历，若返回值为false则访问器将停止于此visit方法。

```java
@Getter
public class TestVisitor extends MySqlASTVisitorAdapter {
    @Override
    public boolean visit(SQLSelectStatement x) {
        System.out.println("SQLSelectStatement");
        return false;
    }

    @Override
    public boolean visit(SQLSelect x) {
        // 不会被访问
        System.out.println("SQLSelect");
        return super.visit(x);
    }
}
```

测试：

```java
@Test
public void testSql() {
    String sql = "select * from test where name = 'maple';";
    List<SQLStatement> sqlStatements = SQLUtils.parseStatements(sql, JdbcConstants.MYSQL);
    SQLStatement sqlStatement = sqlStatements.get(0);
    TestVisitor testVisitor = new TestVisitor();
    sqlStatement.accept(testVisitor);
}
/**
 * out:
 * SQLSelectStatement
 */
```

## 使用访问器修改查询语句

### 修改查询字段

这里以一个简单的逻辑删除字段为例，假设我们要为系统里所有的表指定一个字段为布尔值true。

这里有几个可能需要注意的地方：

- 在关联查询时，需要在on上追加条件，但在where条件中只追加左表的条件

举例来说，这里有一条语句：

```sql
SELECT *
FROM test a
	LEFT JOIN test1 b ON a.id = b.id
WHERE a.name = 'maple';
```

假设处理之后的语句变为：

```sql
SELECT *
FROM test a
	LEFT JOIN test1 b
	ON a.id = b.id
		AND a.is_delete IS true
		AND b.is_delete IS true
WHERE a.name = 'maple'
	AND a.is_delete IS true
	AND b.is_delete IS true;
```

那么在右表的条件将会影响整体的查询结果，而对我们来说我们只需要关联不到b表的数据即可（即b表信息为空）。

因此正确的处理应该为：

```sql
SELECT *
FROM test a
	LEFT JOIN test1 b
	ON a.id = b.id
		AND a.is_delete IS true
		AND b.is_delete IS true
WHERE a.name = 'maple'
	AND a.is_delete IS true;
```

- 在语句中已经指定了的条件，需要去除，即若源语句中指定了is_delete is false，则访问器不应再处理此字段（即以开发者语句为准）。

如下图所示，不论是on条件还是where条件在访问器中最终都会访问至到SQLBinaryOpExpr节点。

![join语句节点解析信息](https://gitee.com/gonghs/image/raw/master/img/20211119210947.png)

语句处理的整体思路为在访问器遍历到SQLBinaryOpExpr节点时向上找到当前层级的表别名等信息并进行统一条件追加。

以下为访问器的具体实现：

```java
public class SelectVisitor extends MySqlASTVisitorAdapter {
    private final String EMPTY_TABLE_KEY = StrUtil.EMPTY;
    protected final ColumnInfo BASE_COLUMN_INFO =
            ColumnInfo.builder().columnName("is_delete").columnValue(new SQLBooleanExpr(Boolean.TRUE)).build();

    @Override
    public boolean visit(SQLBinaryOpExpr x) {
        // 在此阶段判断上一个节点是什么 并进行各种判断
        SQLObject parent = x.getParent();
        // on块
        if (parent instanceof SQLJoinTableSource) {
            // 拼接要加入的条件
            List<TableInfo> tableInfoList = getTableInfoList(((SQLJoinTableSource) parent),
                    new ArrayList<>());
            filterCustomColumnTable(x, tableInfoList);
            SQLExpr newExpr = getNewExpr(tableInfoList, x);
            ((SQLJoinTableSource) parent).setCondition(newExpr);
            return false;
        }
        // where块
        if (parent instanceof MySqlSelectQueryBlock) {
            // 获取表信息时忽略右表
            List<TableInfo> tableInfoList = getTableInfoList(((MySqlSelectQueryBlock) parent).getFrom(),
                    new ArrayList<>(), Boolean.FALSE);
            filterCustomColumnTable(x, tableInfoList);
            SQLExpr newExpr = getNewExpr(tableInfoList, x);
            ((MySqlSelectQueryBlock) parent).setWhere(newExpr);
            return false;
        }
        return false;
    }

    /**
     * 遍历并获得当前层级下的表别名
     *
     * @param tableSource   表信息 由form块获取
     * @param tableInfoList 用于迭代的表信息集合
     * @return 表信息集合
     */
    protected List<TableInfo> getTableInfoList(SQLTableSource tableSource, List<TableInfo> tableInfoList) {
        return getTableInfoList(tableSource, tableInfoList, Boolean.TRUE);
    }

    /**
     * 遍历并获得当前层级下的表别名
     *
     * @param tableSource   表信息 由form块获取
     * @param tableInfoList 用于迭代的表信息集合
     * @param isGetRight    关联查询是是否获取右表信息
     * @return 表信息集合
     */
    private List<TableInfo> getTableInfoList(SQLTableSource tableSource, List<TableInfo> tableInfoList,
                                             Boolean isGetRight) {
        if (tableSource instanceof SQLSubqueryTableSource) {
            tableInfoList.add(new TableInfo(EMPTY_TABLE_KEY, tableSource.getAlias()));
        }

        if (tableSource instanceof SQLJoinTableSource) {
            SQLJoinTableSource joinSource = (SQLJoinTableSource) tableSource;
            getTableInfoList(joinSource.getLeft(), tableInfoList, isGetRight);
            // 这里如果是join语句在where条件中是不需要加入右表的 因为关联查询关联表不应该影响数据条数 应该只影响关联结果
            if (isGetRight) {
                getTableInfoList(joinSource.getRight(), tableInfoList, true);
            }
        }

        if (tableSource instanceof SQLExprTableSource) {
            tableInfoList.add(new TableInfo(String.valueOf(tableSource), tableSource.getAlias()));
        }
        return tableInfoList;
    }

    /**
     * 过滤已经自定义了字段的表信息, 例如当前条件中已经追加了 is_delete is false 则不需要再在语句中加入 is_delete is true
     *
     * @param where         条件
     * @param tableInfoList 表信息
     */
    private void filterCustomColumnTable(SQLExpr where, List<TableInfo> tableInfoList) {
        if (Objects.isNull(where)) {
            return;
        }

        // 遍历左右两端
        SQLExpr sqlExpr = where;
        while (sqlExpr instanceof SQLBinaryOpExpr && Objects.nonNull(sqlExpr = ((SQLBinaryOpExpr) sqlExpr).getLeft())) {
            filterCustomColumnTable(sqlExpr, tableInfoList);
        }

        sqlExpr = where;
        while (sqlExpr instanceof SQLBinaryOpExpr && Objects.nonNull(sqlExpr = ((SQLBinaryOpExpr) sqlExpr).getRight())) {
            filterCustomColumnTable(sqlExpr, tableInfoList);
        }

        // 未设置别名 则直接判断是否当前字段
        if (where instanceof SQLInSubQueryExpr) {
            filterCustomColumnTable(((SQLInSubQueryExpr) where).getExpr(), tableInfoList);
        }
        // 处理形如 A = 1 存在字段名相同的条件则去掉
        if (where instanceof SQLIdentifierExpr) {
            tableInfoList.removeIf(item -> Objects.isNull(item.getAlias()) && Objects.equals(BASE_COLUMN_INFO.getColumnName(),
                    ((SQLIdentifierExpr) where).getName()));
            return;
        }

        // 处理形如 a.A = 1 存在别名和字段名都相同的条件则去掉
        if (where instanceof SQLPropertyExpr) {
            SQLPropertyExpr propertyExpr = (SQLPropertyExpr) where;
            tableInfoList.removeIf(item -> StringUtils.equalsIgnoreCase(propertyExpr.getOwner().toString(),
                    StrUtil.blankToDefault(item.getAlias(), item.getTableName())) && StringUtils.equalsIgnoreCase(propertyExpr.getName(),
                    BASE_COLUMN_INFO.getColumnName()));
        }

    }

    /**
     * 获取新的条件语句（若需要进行配置之类的处理，在这里统一处理即可）
     *
     * @param tableInfoList 表信息
     * @param x             原始表达式
     * @return 处理后的表达式
     */
    private SQLExpr getNewExpr(List<TableInfo> tableInfoList, SQLBinaryOpExpr x) {
        SQLExpr allOpExpr = x;
        for (TableInfo item : tableInfoList) {
            SQLBinaryOpExpr addOpExpr = new SQLBinaryOpExpr();
            // 如果别名为空则将表名做别名处理
            addOpExpr.setLeft(new SQLPropertyExpr(StrUtil.blankToDefault(item.getAlias(), item.getTableName()),
                    BASE_COLUMN_INFO.getColumnName()));
            addOpExpr.setOperator(SQLBinaryOperator.Is);
            addOpExpr.setRight(BASE_COLUMN_INFO.getColumnValue());
            allOpExpr = SQLBinaryOpExpr.and(allOpExpr, addOpExpr);
        }
        return allOpExpr;
    }
}
```

相关测试（测试不全处烦请评论指出）：

```java
@Test
public void selectVisitor() {
    String sql = "select * from test left join test1 on 1 = 1 where name = 'maple';";
    String sql5 = "select * from test a left join test1 b on a.id = b.id where a.name = 'maple';";
    String sql2 = "select * from (select * from test a left join test1 b on 1 = 1 where name = 'maple') x where x" +
            ".sex = 'man';";
    String sql3 = "select * from (select * from test a left join test1 b on 1 = 1 and a.is_delete is false where " +
            "name = 'maple') x where x.sex = 'man' and x.is_delete is false;";
    String sql4 = "select * from (select * from test a left join test1 b on 1 = 1 and a.is_delete is false where " +
            "name = 'maple') x where x.sex = 'man' and x.is_delete is false " +
            "union all select * from (select * from test a left join test1 b on 1 = 1 and a.is_delete is false " +
            "where name = 'maple') x where x.sex = 'woman' and x.is_delete is false;";
    SelectVisitor selectVisitor = new SelectVisitor();
    System.out.println("--------测试无别名关联查询-------");
    accept(selectVisitor, sql);
    System.out.println("--------测试有别名关联查询-------");
    accept(selectVisitor, sql5);
    System.out.println("--------测试子查询-------");
    accept(selectVisitor, sql2);
    System.out.println("--------测试已经自定义条件的情况下是否会覆盖条件-------");
    accept(selectVisitor, sql3);
    System.out.println("--------测试union-------");
    accept(selectVisitor, sql4);
}

 private void accept(MySqlASTVisitorAdapter visitorAdapterSupplier, String sql) {
    List<SQLStatement> sqlStatements = SQLUtils.parseStatements(sql, JdbcConstants.MYSQL);
    for (SQLStatement sqlStatement : sqlStatements) {
        System.out.println("原始语句： " + sqlStatement.toString());
        sqlStatement.accept(visitorAdapterSupplier);
        System.out.println("处理后语句： " + sqlStatement.toString());
    }
}

/**
 * out:
 * --------测试无别名关联查询-------
 * 原始语句： SELECT *
 * FROM test
 * 	LEFT JOIN test1 ON 1 = 1
 * WHERE name = 'maple';
 * 处理后语句： SELECT *
 * FROM test
 * 	LEFT JOIN test1
 * 	ON 1 = 1
 * 		AND test.is_delete IS true
 * 		AND test1.is_delete IS true
 * WHERE name = 'maple'
 * 	AND test.is_delete IS true;
 * --------测试有别名关联查询-------
 * 原始语句： SELECT *
 * FROM test a
 * 	LEFT JOIN test1 b ON a.id = b.id
 * WHERE a.name = 'maple';
 * 处理后语句： SELECT *
 * FROM test a
 * 	LEFT JOIN test1 b
 * 	ON a.id = b.id
 * 		AND a.is_delete IS true
 * 		AND b.is_delete IS true
 * WHERE a.name = 'maple'
 * 	AND a.is_delete IS true;
 * --------测试子查询-------
 * 原始语句： SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b ON 1 = 1
 * 	WHERE name = 'maple'
 * ) x
 * WHERE x.sex = 'man';
 * 处理后语句： SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS true
 * 			AND b.is_delete IS true
 * 	WHERE name = 'maple'
 * 		AND a.is_delete IS true
 * ) x
 * WHERE x.sex = 'man'
 * 	AND x.is_delete IS true;
 * --------测试已经自定义条件的情况下是否会覆盖条件-------
 * 原始语句： SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS false
 * 	WHERE name = 'maple'
 * ) x
 * WHERE x.sex = 'man'
 * 	AND x.is_delete IS false;
 * 处理后语句： SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS false
 * 			AND b.is_delete IS true
 * 	WHERE name = 'maple'
 * 		AND a.is_delete IS true
 * ) x
 * WHERE x.sex = 'man'
 * 	AND x.is_delete IS false;
 * --------测试union-------
 * 原始语句： SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS false
 * 	WHERE name = 'maple'
 * ) x
 * WHERE x.sex = 'man'
 * 	AND x.is_delete IS false
 * UNION ALL
 * SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS false
 * 	WHERE name = 'maple'
 * ) x
 * WHERE x.sex = 'woman'
 * 	AND x.is_delete IS false;
 * 处理后语句： SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS false
 * 			AND b.is_delete IS true
 * 	WHERE name = 'maple'
 * 		AND a.is_delete IS true
 * ) x
 * WHERE x.sex = 'man'
 * 	AND x.is_delete IS false
 * UNION ALL
 * SELECT *
 * FROM (
 * 	SELECT *
 * 	FROM test a
 * 		LEFT JOIN test1 b
 * 		ON 1 = 1
 * 			AND a.is_delete IS false
 * 			AND b.is_delete IS true
 * 	WHERE name = 'maple'
 * 		AND a.is_delete IS true
 * ) x
 * WHERE x.sex = 'woman'
 * 	AND x.is_delete IS false;
 */
```

### 修改插入字段

同样以逻辑删除字段为例，我们要为表的is_delete字段默认增加一个为true的值。可能需要注意的地方：

- 与查询相同，需要做用户语句优先的处理，即若插入字段中存在此字段则不再处理这个字段。

以下为两个常规插入语句的解析节点图：

```sql
insert into test(id, name) values(1, 'maple');
```

![插入语句节点图](https://gitee.com/gonghs/image/raw/master/img/20211120155415.png)

存在子查询时结构上有一些不同：

```sql
insert into test(name) select name from test where name = 'maple';
```

![image-20211120160324117](https://gitee.com/gonghs/image/raw/master/img/20211120160324.png)

以上的节点解析结果可以看出，当不存在子查询时，访问器需要做的处理包含columns与valueList两部分，分别需要加上我们需要添加的字段名与字段值。

存在子查询是，需要处理columns与query两部分，其中columns加上我们需要添加的字段名，query块中的selectList部分需要加上我们需要添加的字段名，并且如果需要的话子查询部分可以继承查询部分的处理。

代码实现：

```java
public class InsertVisitor extends SelectVisitor {

    @Override
    public boolean visit(MySqlInsertStatement x) {
        List<SQLExpr> exprList = x.getColumns();
        SQLIdentifierExpr sqlIdentifierExpr = new SQLIdentifierExpr(BASE_COLUMN_INFO.getColumnName());
        // 存在此字段则忽略
        if (exprList.contains(sqlIdentifierExpr)) {
            return super.visit(x);
        }
        // 加入插入字段和默认值
        exprList.add(sqlIdentifierExpr);

        // 如果为空 则可能为子查询 如果子查询的条件不足 则加上
        if (CollUtil.isEmpty(x.getValuesList()) && Objects.nonNull(x.getQuery())
                && x.getQuery().getQuery() instanceof MySqlSelectQueryBlock) {
            List<SQLSelectItem> childSelectList = ((MySqlSelectQueryBlock) x.getQuery().getQuery()).getSelectList();
            if (childSelectList.size() < exprList.size()) {
                SQLSelectItem sqlSelectItem = new SQLSelectItem();
                sqlSelectItem.setExpr(BASE_COLUMN_INFO.getColumnValue());
                childSelectList.add(sqlSelectItem);
            }
            return super.visit(x);
        }
        x.getValuesList().forEach(valueClause -> valueClause.addValue(BASE_COLUMN_INFO.getColumnValue()));
        return super.visit(x);
    }
}
```

测试与测试结果：

```java
@Test
public void insertVisitor() {
    String sql = "insert into test(id, name) values(1, 'maple');";
    String sql2 = "insert into test(name) select name from test where name = 'maple';";
    String sql3 = "insert into test(name,is_delete) select name,false from test where name = 'maple';";
    InsertVisitor insertVisitor = new InsertVisitor();
    System.out.println("--------测试普通插入语句-------");
    accept(insertVisitor, sql);
    System.out.println("--------测试子查询插入语句-------");
    accept(insertVisitor, sql2);
    System.out.println("--------测试已经存在对应字段的插入语句-------");
    accept(insertVisitor, sql3);
}
/**
 * out:
 * --------测试普通插入语句-------
 * 原始语句： INSERT INTO test (id, name)
 * VALUES (1, 'maple');
 * 处理后语句： INSERT INTO test (id, name, is_delete)
 * VALUES (1, 'maple', true);
 * --------测试子查询插入语句-------
 * 原始语句： INSERT INTO test (name)
 * SELECT name
 * FROM test
 * WHERE name = 'maple';
 * 处理后语句： INSERT INTO test (name, is_delete)
 * SELECT name, true
 * FROM test
 * WHERE name = 'maple'
 * 	AND test.is_delete IS true;
 * --------测试已经存在对应字段时插入语句-------
 * 原始语句： INSERT INTO test (name, is_delete)
 * SELECT name, false
 * FROM test
 * WHERE name = 'maple';
 * 处理后语句： INSERT INTO test (name, is_delete)
 * SELECT name, false
 * FROM test
 * WHERE name = 'maple'
 * 	AND test.is_delete IS true;
 */
```

### 修改更新字段

同样以逻辑删除字段为例，我们要为表的is_delete字段默认更新为true的值。可能需要注意的地方与插入语句一致。

update语句的基本节点结构如下:

![image-20211122003820823](https://gitee.com/gonghs/image/raw/master/img/20211122003820.png)

当语句存在关联更新时，例如语句为如下语句时：

```sql
update test left join test t on test.username = t.username 
set test.is_delete = 1, test.username = 'maple' 
where test.username = '张三';
```

tableSource部分将会变为SQLJoinTableSource与查询类似，继承查询部分即可完成替换。

基本实现如下：

```java
public class UpdateVisitor extends SelectVisitor {

    @Override
    public boolean visit(MySqlUpdateStatement x) {
        List<SQLUpdateSetItem> updateItemList = x.getItems();
        // 如果已经包含指定字段则忽略
        List<TableInfo> tableInfoList = getTableInfoList(x.getTableSource(), new ArrayList<>());
        filterCustomColumnTable(updateItemList, tableInfoList);
        for (TableInfo item : tableInfoList) {
            SQLUpdateSetItem sqlUpdateSetItem = new SQLUpdateSetItem();
            sqlUpdateSetItem.setColumn(new SQLPropertyExpr(StrUtil.blankToDefault(item.getAlias(),
                    item.getTableName()), BASE_COLUMN_INFO.getColumnName()));
            sqlUpdateSetItem.setValue(BASE_COLUMN_INFO.getColumnValue());
            updateItemList.add(sqlUpdateSetItem);
        }
        return super.visit(x);
    }

    private void filterCustomColumnTable(List<SQLUpdateSetItem> updateItemList, List<TableInfo> tableInfoList) {
        for (SQLUpdateSetItem sqlUpdateSetItem : updateItemList) {
            // 处理形如 A = 1 存在字段名相同的条件则去掉
            if (sqlUpdateSetItem.getColumn() instanceof SQLIdentifierExpr) {
                tableInfoList.removeIf(item -> Objects.isNull(item.getAlias()) && Objects.equals("is_delete",
                        ((SQLIdentifierExpr) sqlUpdateSetItem.getColumn()).getName()));
                continue;
            }

            // 处理形如 a.A = 1 存在别名和字段名都相同的条件则去掉
            if (sqlUpdateSetItem.getColumn() instanceof SQLPropertyExpr) {
                SQLPropertyExpr propertyExpr = (SQLPropertyExpr) sqlUpdateSetItem.getColumn();
                tableInfoList.removeIf(item -> StrUtil.equalsIgnoreCase(propertyExpr.getOwner().toString(),
                        StrUtil.blankToDefault(item.getAlias(), item.getTableName())) && StrUtil.equalsIgnoreCase(propertyExpr.getName(), BASE_COLUMN_INFO.getColumnName()));
            }
        }
    }
}
```

测试与测试结果：

```java
@Test
public void updateVisitor() {
    String sql = "update test set name = 'maple' where name = '张三'";
    String sql2 = "update test set name = 'maple',test.is_delete = true where name = '张三'";
    String sql3 = "update test left join test t on test.username = t.username set test.is_delete = 1, test" +
            ".username = 'maple' where test.username = 'maple';";
    UpdateVisitor updateVisitor = new UpdateVisitor();
    System.out.println("--------测试普通更新语句-------");
    accept(updateVisitor, sql);
    System.out.println("--------测试已经存在对应字段的更新语句-------");
    accept(updateVisitor, sql2);
    System.out.println("--------测试联表的更新语句-------");
    accept(updateVisitor, sql3);
}
/**
 * out:
 * --------测试普通更新语句-------
 * 原始语句： UPDATE test
 * SET name = 'maple'
 * WHERE name = '张三'
 * 处理后语句： UPDATE test
 * SET name = 'maple', test.is_delete = true
 * WHERE name = '张三'
 * --------测试已经存在对应字段的更新语句-------
 * 原始语句： UPDATE test
 * SET name = 'maple', test.is_delete = true
 * WHERE name = '张三'
 * 处理后语句： UPDATE test
 * SET name = 'maple', test.is_delete = true
 * WHERE name = '张三'
 * --------测试联表的更新语句-------
 * 原始语句： UPDATE test
 * 	LEFT JOIN test t ON test.username = t.username
 * SET test.is_delete = 1, test.username = 'maple'
 * WHERE test.username = 'maple';
 * 处理后语句： UPDATE test
 * 	LEFT JOIN test t
 * 	ON test.username = t.username
 * 		AND test.is_delete IS true
 * 		AND t.is_delete IS true
 * SET test.is_delete = 1, test.username = 'maple', t.is_delete = true
 * WHERE test.username = 'maple';
 */
```

### 修改删除字段

删除整体与查询类似，只是需要在查询的基础上加上部分逻辑判断。

```java
// where块
if (parent instanceof MySqlDeleteStatement) {
    // 获取表信息时忽略右表
    List<TableInfo> tableInfoList = getTableInfoList(((MySqlDeleteStatement) parent).getTableSource(),
            new ArrayList<>(), Boolean.FALSE);
    filterCustomColumnTable(x, tableInfoList);
    SQLExpr newExpr = getNewExpr(tableInfoList, x);
    ((MySqlDeleteStatement) parent).setWhere(newExpr);
    return false;
}
```

测试与测试结果：

```java
@Test
public void deleteVisitor() {
    String sql = "delete from test where name = '张三'";
    String sql2 = "delete test,test1\n" +
            "from test\n" +
            "         left join test1 on test1.username = test.username and test1.is_delete is false\n" +
            "where test.username = '张三'";
    DeleteVisitor deleteVisitor = new DeleteVisitor();
    System.out.println("--------测试普通删除语句-------");
    accept(deleteVisitor, sql);
    System.out.println("--------测试联表删除语句-------");
    accept(deleteVisitor, sql2);

}
/**
 * out:
 * --------测试普通删除语句-------
 * 原始语句： DELETE FROM test
 * WHERE name = '张三'
 * 处理后语句： DELETE FROM test
 * WHERE name = '张三'
 * 	AND test.is_delete IS true
 * --------测试联表删除语句-------
 * 原始语句： DELETE test, test1
 * FROM test
 * 	LEFT JOIN test1
 * 	ON test1.username = test.username
 * 		AND test1.is_delete IS false
 * WHERE test.username = '张三'
 * 处理后语句： DELETE test, test1
 * FROM test
 * 	LEFT JOIN test1
 * 	ON test1.username = test.username
 * 		AND test1.is_delete IS false
 * 		AND test.is_delete IS true
 * WHERE test.username = '张三'
 * 	AND test.is_delete IS true
 */
```