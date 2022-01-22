# ShardingSphere 语句解析生成初探
***
## 简介
在上篇文章中，我们找到了一个逻辑SQL转换到真实SQL的关键路径代码，本篇文中，我们就上篇基础上，来探索语句解析生成的一些细节

## 源码解析
语句的关键解析生成的代码如下：

```java
@RequiredArgsConstructor
public abstract class AbstractSQLBuilder implements SQLBuilder {
    
    private final SQLRewriteContext context;
    
    @Override
    public final String toSQL() {
        if (context.getSqlTokens().isEmpty()) {
            return context.getSql();
        }
        Collections.sort(context.getSqlTokens());
        StringBuilder result = new StringBuilder();
        result.append(context.getSql(), 0, context.getSqlTokens().get(0).getStartIndex());
        for (SQLToken each : context.getSqlTokens()) {
            result.append(each instanceof ComposableSQLToken ? getComposableSQLTokenText((ComposableSQLToken) each) : getSQLTokenText(each));
            result.append(getConjunctionText(each));
        }
        return result.toString();
    }
    
    protected abstract String getSQLTokenText(SQLToken sqlToken);

    private String getComposableSQLTokenText(final ComposableSQLToken composableSQLToken) {
        StringBuilder result = new StringBuilder();
        for (SQLToken each : composableSQLToken.getSqlTokens()) {
            result.append(getSQLTokenText(each));
            result.append(getConjunctionText(each));
        }
        return result.toString();
    }

    private String getConjunctionText(final SQLToken sqlToken) {
        return context.getSql().substring(getStartIndex(sqlToken), getStopIndex(sqlToken));
    }
    
    private int getStartIndex(final SQLToken sqlToken) {
        int startIndex = sqlToken instanceof Substitutable ? ((Substitutable) sqlToken).getStopIndex() + 1 : sqlToken.getStartIndex();
        return Math.min(startIndex, context.getSql().length());
    }
    
    private int getStopIndex(final SQLToken sqlToken) {
        int currentSQLTokenIndex = context.getSqlTokens().indexOf(sqlToken);
        return context.getSqlTokens().size() - 1 == currentSQLTokenIndex ? context.getSql().length() : context.getSqlTokens().get(currentSQLTokenIndex + 1).getStartIndex();
    }
}
```

从toSQL函数可以看到，基本都死用context变量里面去获取生成真实SQL的，其内容大致如下：


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b6434f202d84280bd66cb016bfb0e14~tplv-k3u1fbpfcp-watermark.image)

其他的TableToken比较丰富，包含了很多的信息，可能是逻辑表名需要转成真实表名，所有需要这么多的信息

我们看到context.getSqlTokens()的长度是2，那result的就会经过三次添加：

- 未循环遍历处理前的初始添加：result.append(context.getSql(), 0, context.getSqlTokens().get(0).getStartIndex());
- 第一次循环添加： TableToken
- 第二次循环添加： 
- 第三次循环添加

下面我们就仔细跟下这三次添加

### 初始添加
```java
result.append(context.getSql(), 0, context.getSqlTokens().get(0).getStartIndex());
```

上面的语句就是从字符串中截取子字符串添加，从原始的SQL截取的，起点是0，结束点是第一个SQLTokens的startIndex

相应的变化如下：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO 

这步感觉对于所有的SQL来说确实是通用的，前面这种语句应该都是不用进行解析变换的（如何获取截取的结束点，这个是SQLToken的生成部分，这次先不看，先跟一下处理流程）

### 第一次循环添加： TableToken
```java
        for (SQLToken each : context.getSqlTokens()) {
            result.append(each instanceof ComposableSQLToken ? getComposableSQLTokenText((ComposableSQLToken) each) : getSQLTokenText(each));
            result.append(getConjunctionText(each));
        }
```

从上面的语句看出，目前有两种处理方式：

- 组合SQLToken的处理： getComposableSQLTokenText
- 非组合SQLToken的处理： getSQLTokenText

我们跟踪得到直接走的： getSQLTokenText

跟踪进入下面的函数，如果是 RouteUnitAware,还需要进行处理

```java
public final class RouteSQLBuilder extends AbstractSQLBuilder {
    
    @Override
    protected String getSQLTokenText(final SQLToken sqlToken) {
        if (sqlToken instanceof RouteUnitAware) {
            return ((RouteUnitAware) sqlToken).toString(routeUnit);
        }
        return sqlToken.toString();
    }
}
```

看看TableToken是如何ToString的，相关的细节如下

```java
public final class TableToken extends SQLToken implements Substitutable, RouteUnitAware {
    
    @Override
    public String toString(final RouteUnit routeUnit) {
	// 得到了真实的表名，真实表名从逻辑表到真实表的转换Map中获取的
        String actualTableName = getLogicAndActualTables(routeUnit).get(tableName.getValue().toLowerCase());
	// 如果真实表名是null，则获取tableName的值（还能这么写成一句，学到了）
	// tableName是value值是原始的SQL的 t_order
        actualTableName = null == actualTableName ? tableName.getValue().toLowerCase() : actualTableName;
        return tableName.getQuoteCharacter().wrap(actualTableName);
    }
    
    // 从下面的函数大意就可以看出是构建逻辑表名到真实表的映射转换Map 
    private Map<String, String> getLogicAndActualTables(final RouteUnit routeUnit) {
        Collection<String> tableNames = sqlStatementContext.getTablesContext().getTableNames();
        Map<String, String> result = new HashMap<>(tableNames.size(), 1);
        for (RouteMapper each : routeUnit.getTableMappers()) {
            result.put(each.getLogicName().toLowerCase(), each.getActualName());
	    // 为啥还要再加一次，这里没有理解
            result.putAll(shardingRule.getLogicAndActualTablesFromBindingTable(routeUnit.getDataSourceMapper().getLogicName(), each.getLogicName(), each.getActualName(), tableNames));
        }
        return result;
    }
}
```

return tableName.getQuoteCharacter().wrap(actualTableName) 核心代码大致如下,额外的加一些东西（关键字保留字段处理之类的？）

```java
@Getter
public enum QuoteCharacter {
    
    BACK_QUOTE("`", "`"),
    
    SINGLE_QUOTE("'", "'"),
    
    QUOTE("\"", "\""),
    
    BRACKETS("[", "]"),
    
    NONE("", "");
    
    private final String startDelimiter;
    
    private final String endDelimiter;
    
    /**
     * Wrap value with quote character.
     * 
     * @param value value to be wrapped
     * @return wrapped value
     */
    public String wrap(final String value) {
        return startDelimiter + value + endDelimiter;
    }
}
```

相应的变化如下：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0

下面来到: result.append(getConjunctionText(each));

```java
public abstract class AbstractSQLBuilder implements SQLBuilder {
    private String getConjunctionText(final SQLToken sqlToken) {
        return context.getSql().substring(getStartIndex(sqlToken), getStopIndex(sqlToken));
    }
    
    private int getStartIndex(final SQLToken sqlToken) {
        int startIndex = sqlToken instanceof Substitutable ? ((Substitutable) sqlToken).getStopIndex() + 1 : sqlToken.getStartIndex();
        return Math.min(startIndex, context.getSql().length());
    }
    
    private int getStopIndex(final SQLToken sqlToken) {
        int currentSQLTokenIndex = context.getSqlTokens().indexOf(sqlToken);
        return context.getSqlTokens().size() - 1 == currentSQLTokenIndex ? context.getSql().length() : context.getSqlTokens().get(currentSQLTokenIndex + 1).getStartIndex();
    }
}
```

大意就是获取开始和结束截取点，算法目前还没领会......，但和SQLToken的关系很大，看后面看SQLToken的时候能不能得到解答

相应的变化如下：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0 (user_id, address_id, status

### 第二次循环添加： 
我们看看第二次循环添加：GeneratedKeyInsertColumnToken

```java
@Override
    protected String getSQLTokenText(final SQLToken sqlToken) {
        if (sqlToken instanceof RouteUnitAware) {
            return ((RouteUnitAware) sqlToken).toString(routeUnit);
        }
	// 直接走的这
        return sqlToken.toString();
    }
}

public final class  extends SQLToken implements Attachable {
    
    @Override
    public String toString() {
	// 简单粗暴的直接插入一列名
        return String.format(", %s", column);
    }
}
```

从上面的大意看出，就是插入一列名，相应的变化如下：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0 (user_id, address_id, status, order_id

接着: result.append(getConjunctionText(each))，变成：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0 (user_id, address_id, status, order_id) VALUES

### 第三次循环添加:
我们接着看第三次循环添加：ShardingInsertValuesToken

```java
public final class RouteSQLBuilder extends AbstractSQLBuilder {
    
    @Override
    protected String getSQLTokenText(final SQLToken sqlToken) {
        if (sqlToken instanceof RouteUnitAware) {
	    // 走的这
            return ((RouteUnitAware) sqlToken).toString(routeUnit);
        }
        return sqlToken.toString();
    }
}

public final class ShardingInsertValuesToken extends InsertValuesToken implements RouteUnitAware {
    
    public ShardingInsertValuesToken(final int startIndex, final int stopIndex) {
        super(startIndex, stopIndex);
    }
    
    @Override
    public String toString(final RouteUnit routeUnit) {
        StringBuilder result = new StringBuilder();
	// 这里得到了：(?, ?, ?, ?)，
        appendInsertValue(routeUnit, result);
	// 然后又变成了：(?, ?, ?, ?)
        result.delete(result.length() - 2, result.length());
        return result.toString();
    }
    
    private void appendInsertValue(final RouteUnit routeUnit, final StringBuilder stringBuilder) {
        for (InsertValue each : getInsertValues()) {
            if (isAppend(routeUnit, (ShardingInsertValue) each)) {
                stringBuilder.append(each).append(", ");
            }
        }
    }
    
    private boolean isAppend(final RouteUnit routeUnit, final ShardingInsertValue insertValueToken) {
        if (insertValueToken.getDataNodes().isEmpty() || null == routeUnit) {
            return true;
        }
        for (DataNode each : insertValueToken.getDataNodes()) {
            if (routeUnit.findTableMapper(each.getDataSourceName(), each.getTableName()).isPresent()) {
                return true;
            }
        }
        return false;
    }
}
```

对于这个目前不懂，为啥要经过这个处理，应用场景是啥？

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0 (user_id, address_id, status, order_id) VALUES (?, ?, ?, ?)

接着: result.append(getConjunctionText(each))，变成：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0 (user_id, address_id, status, order_id) VALUES (?, ?, ?, ?)

到这里就解析生成完毕了

## 总结
本篇中详细查看了逻辑SQL转真实SQL的处理过程：

- 原始逻辑SQL： INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)
- result从空字符串变成了： INSERT INTO t_order_0 (user_id, address_id, status, order_id) VALUES (?, ?, ?, ?)

转换中使用了相应的Token类进行对应的处理，组装成真实的SQL

但算法之类的，目前还是不甚了解的，后面找找官方资料，有没有这方面的

下面应该要去看看SQLToken的生成了，通过上面的跟踪发现，真实SQL的生成和SQLToken是息息相关的