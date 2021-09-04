# ShardingSphere SQLToken 生成探索
***
## 简介
在上篇文章中，我们探索了ShardingSphere中逻辑SQL变成真实SQL的解析过程，其中SQLToken在其中有着关键的作用，这篇文章中就探索下SQLToken的生成

## 源码分析
基于上文：

- [ShardingSphere 语句解析生成初探](https://juejin.cn/post/7003073129643769869)

我们继续看看SQLToken的生成

首先定位到SQLToken生成的关键代码部分：

再跟着下去，来到核心的部分：

INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?)

```java
public final class SQLTokenGenerators {
    
    /**
     * Generate SQL tokens.
     *
     * @param sqlStatementContext SQL statement context
     * @param parameters SQL parameters
     * @param schema sShardingSphere schema
     * @return SQL tokens
     */
    @SuppressWarnings("unchecked")
    public List<SQLToken> generateSQLTokens(final SQLStatementContext sqlStatementContext, final List<Object> parameters, final ShardingSphereSchema schema) {
        List<SQLToken> result = new LinkedList<>();
	// 遍历所有的sqlTokenGenerators
        for (SQLTokenGenerator each : sqlTokenGenerators) {
            setUpSQLTokenGenerator(each, parameters, schema, result);
	    // 这个应该类似判断是否符合条件
            if (!each.isGenerateSQLToken(sqlStatementContext)) {
                continue;
            }
	    // 下面就是生成Token
            if (each instanceof OptionalSQLTokenGenerator) {
                SQLToken sqlToken = ((OptionalSQLTokenGenerator) each).generateSQLToken(sqlStatementContext);
                if (!result.contains(sqlToken)) {
                    result.add(sqlToken);
                }
            } else if (each instanceof CollectionSQLTokenGenerator) {
                result.addAll(((CollectionSQLTokenGenerator) each).generateSQLTokens(sqlStatementContext));
            }
        }
        return result;
    }
}
```

所有的sqlTokenGenerators如下图：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a276f756b8644b9a5d0ab004fa580b2~tplv-k3u1fbpfcp-watermark.image)

我们跟着看一下：setUpSQLTokenGenerator(each, parameters, schema, result);

```java
public final class SQLTokenGenerators {
    
    private void setUpSQLTokenGenerator(final SQLTokenGenerator sqlTokenGenerator, final List<Object> parameters, final ShardingSphereSchema schema, final List<SQLToken> previousSQLTokens) {
        if (sqlTokenGenerator instanceof ParametersAware) {
            ((ParametersAware) sqlTokenGenerator).setParameters(parameters);
        }
        if (sqlTokenGenerator instanceof SchemaMetaDataAware) {
            ((SchemaMetaDataAware) sqlTokenGenerator).setSchema(schema);
        }
        if (sqlTokenGenerator instanceof PreviousSQLTokensAware) {
            ((PreviousSQLTokensAware) sqlTokenGenerator).setPreviousSQLTokens(previousSQLTokens);
        }
    }
}
```

看到目前有三种Aware，自己目前不太清楚其大致作用是啥。但没有提前返回，会不会存在同时匹配多个的情况？暂时先不管，继续跟下去

#### RemoveTokenGenerator
通过debug sqlStatementContext 的内容，我们大致可以知道，remove的相关东西都是直接写到 sqlStatementContext 中的，但具体作用还是不太了解，后面去补一补应用场景

一顿操作下来，返回的false

```java
public final class RemoveTokenGenerator implements CollectionSQLTokenGenerator<SQLStatementContext<?>> {
    
    @Override
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
	// 下面两个都是直接去取sqlStatementContext对应的字段有没有
        boolean containsRemoveSegment = false;
        if (sqlStatementContext instanceof RemoveAvailable) {
            containsRemoveSegment = !((RemoveAvailable) sqlStatementContext).getRemoveSegments().isEmpty();
        }
        boolean containsSchemaName = false;
        if (sqlStatementContext instanceof TableAvailable) {
            containsSchemaName = ((TableAvailable) sqlStatementContext).getTablesContext().getSchemaName().isPresent();
        }
        return containsRemoveSegment || containsSchemaName;
    }
}
```

#### TableTokenGenerator
这个默认就返回true了，并且走的是CollectionSQLTokenGenerator分支，那我们就接口看其token的生成：

```java
@Setter
public final class TableTokenGenerator implements CollectionSQLTokenGenerator, ShardingRuleAware {
    
    private ShardingRule shardingRule;
    
    @Override
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return true;
    }
    
    // 生成前还需要判断一下
    @Override
    public Collection<TableToken> generateSQLTokens(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof TableAvailable ? generateSQLTokens((TableAvailable) sqlStatementContext) : Collections.emptyList();
    }
    
    private Collection<TableToken> generateSQLTokens(final TableAvailable sqlStatementContext) {
        Collection<TableToken> result = new LinkedList<>();
	// 获取获取的table相关的东西，这个类型是InsertStatementContext，所以就获取了getOriginalTables
        for (SimpleTableSegment each : sqlStatementContext.getAllTables()) {
	    // 判断是否有规则匹配上
            if (shardingRule.findTableRule(each.getTableName().getIdentifier().getValue()).isPresent()) {
		// 匹配上则添加token
                result.add(new TableToken(each.getTableName().getStartIndex(), each.getTableName().getStopIndex(), each, (SQLStatementContext) sqlStatementContext, shardingRule));
            }
        }
        return result;
    }
}
```

从上面的代码中，我们看到了一个TableToken的生成，此时的关键变量如下图：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/632643c4e0364d24b91dcbaa786160b7~tplv-k3u1fbpfcp-watermark.image)

其中遍历只遍历了 OriginalTables，tables和shardingRule这些都是生成好的，符合条件后，直接添加生成token

### IndexTokenGenerator
这个就直接没有匹配上了

```java
public final class IndexTokenGenerator implements CollectionSQLTokenGenerator, ShardingRuleAware, SchemaMetaDataAware {
    
    @Override
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof IndexAvailable && !((IndexAvailable) sqlStatementContext).getIndexes().isEmpty();
    }
}
```

### ConstraintTokenGenerator
这个也没有匹配上

```java
public final class ConstraintTokenGenerator implements CollectionSQLTokenGenerator, ShardingRuleAware {
    
    @Override
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof ConstraintAvailable && !((ConstraintAvailable) sqlStatementContext).getConstraints().isEmpty();
    }
}
```

### GeneratedKeyInsertColumnTokenGenerator -- BaseGeneratedKeyTokenGenerator
先来到抽象类，进行一波判断，成功匹配

```java
public abstract class BaseGeneratedKeyTokenGenerator implements OptionalSQLTokenGenerator<InsertStatementContext> {
    
    @Override
    public final boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
	// 连环三重判断
        return sqlStatementContext instanceof InsertStatementContext && ((InsertStatementContext) sqlStatementContext).getGeneratedKeyContext().isPresent()
                && ((InsertStatementContext) sqlStatementContext).getGeneratedKeyContext().get().isGenerated() && isGenerateSQLToken((InsertStatementContext) sqlStatementContext);
    }
    
    protected abstract boolean isGenerateSQLToken(InsertStatementContext insertStatementContext);
}
```

然后来到子类，其也进行一波判断，并且成功匹配

```java
public final class GeneratedKeyInsertColumnTokenGenerator extends BaseGeneratedKeyTokenGenerator {
    
    @Override
    protected boolean isGenerateSQLToken(final InsertStatementContext insertStatementContext) {
        Optional<InsertColumnsSegment> sqlSegment = insertStatementContext.getSqlStatement().getInsertColumns();
        return sqlSegment.isPresent() && !sqlSegment.get().getColumns().isEmpty()
                && insertStatementContext.getGeneratedKeyContext().isPresent()
                && !insertStatementContext.getGeneratedKeyContext().get().getGeneratedValues().isEmpty();
    }
}
```

其insertStatementContext的内容如下图：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/781aff23fbfd408aa9c6d4f66be45e02~tplv-k3u1fbpfcp-watermark.image)

成功匹配后，走的 OptionalSQLTokenGenerator 分支：

```java
public final class GeneratedKeyInsertColumnTokenGenerator extends BaseGeneratedKeyTokenGenerator {
    @Override
    public GeneratedKeyInsertColumnToken generateSQLToken(final InsertStatementContext insertStatementContext) {
        Optional<GeneratedKeyContext> generatedKey = insertStatementContext.getGeneratedKeyContext();
        Preconditions.checkState(generatedKey.isPresent());
        Optional<InsertColumnsSegment> sqlSegment = insertStatementContext.getSqlStatement().getInsertColumns();
        Preconditions.checkState(sqlSegment.isPresent());
        return new GeneratedKeyInsertColumnToken(sqlSegment.get().getStopIndex(), generatedKey.get().getColumnName());
    }
}
```

上面也是取相关的内容各种判断，其中的Preconditions.checkState挺有意思，看其内容如果为false，会抛出IllegalStateException异常，比if判断简洁啊，感觉又学到了一手

最后生成了Token，里面只用到了开始和结束的index

### GeneratedKeyForUseDefaultInsertColumnsTokenGenerator -- BaseGeneratedKeyTokenGenerator
匹配失败了

```java
public final class GeneratedKeyForUseDefaultInsertColumnsTokenGenerator extends BaseGeneratedKeyTokenGenerator {
    
    @Override
    protected boolean isGenerateSQLToken(final InsertStatementContext insertStatementContext) {
        return insertStatementContext.useDefaultColumns();
    }
}
```

### ShardingInsertValuesTokenGenerator
这个匹配通过，并且走入分支： OptionalSQLTokenGenerator

```java
public final class ShardingInsertValuesTokenGenerator implements OptionalSQLTokenGenerator<InsertStatementContext>, RouteContextAware {
    
    private RouteContext routeContext;
    
    @Override
    public boolean isGenerateSQLToken(final SQLStatementContext sqlStatementContext) {
        return sqlStatementContext instanceof InsertStatementContext && !(((InsertStatementContext) sqlStatementContext).getSqlStatement()).getValues().isEmpty();
    }
    
    @Override
    public InsertValuesToken generateSQLToken(final InsertStatementContext insertStatementContext) {
        Collection<InsertValuesSegment> insertValuesSegments = (insertStatementContext.getSqlStatement()).getValues();
	// 直接取相关的最大最小index
        InsertValuesToken result = new ShardingInsertValuesToken(getStartIndex(insertValuesSegments), getStopIndex(insertValuesSegments));
        Iterator<Collection<DataNode>> originalDataNodesIterator = null == routeContext || routeContext.getOriginalDataNodes().isEmpty()
                ? null : routeContext.getOriginalDataNodes().iterator();
	// 感觉这个类似添加values之类的，如果是多值插入，那应该会有多个
        for (InsertValueContext each : insertStatementContext.getInsertValueContexts()) {
            List<ExpressionSegment> expressionSegments = each.getValueExpressions();
            Collection<DataNode> dataNodes = null == originalDataNodesIterator ? Collections.emptyList() : originalDataNodesIterator.next();
            result.getInsertValues().add(new ShardingInsertValue(expressionSegments, dataNodes));
        }
        return result;
    }
    
    // 取最小值
    private int getStartIndex(final Collection<InsertValuesSegment> segments) {
        int result = segments.iterator().next().getStartIndex();
        for (InsertValuesSegment each : segments) {
            result = Math.min(result, each.getStartIndex());
        }
        return result;
    }
    
    // 取最大值
    private int getStopIndex(final Collection<InsertValuesSegment> segments) {
        int result = segments.iterator().next().getStopIndex();
        for (InsertValuesSegment each : segments) {
            result = Math.max(result, each.getStopIndex());
        }
        return result;
    }
}
```

上面的代码大概就是生成SQL中value相关的token，关键的变量值如下：



![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0afb8f33cad440a4be1b32aed6f1a1fe~tplv-k3u1fbpfcp-watermark.image)

### GeneratedKeyInsertValuesTokenGenerator
好像与ShardingInsertValuesTokenGenerator有关联，目前看不太懂

```java
public final class GeneratedKeyInsertValuesTokenGenerator extends BaseGeneratedKeyTokenGenerator implements PreviousSQLTokensAware {
    
    private List<SQLToken> previousSQLTokens;
    
    @Override
    protected boolean isGenerateSQLToken(final InsertStatementContext insertStatementContext) {
        return !insertStatementContext.getSqlStatement().getValues().isEmpty() && insertStatementContext.getGeneratedKeyContext().isPresent()
                && !insertStatementContext.getGeneratedKeyContext().get().getGeneratedValues().isEmpty();
    }
    
    @Override
    public SQLToken generateSQLToken(final InsertStatementContext insertStatementContext) {
        Optional<InsertValuesToken> result = findPreviousSQLToken();
        Preconditions.checkState(result.isPresent());
        Optional<GeneratedKeyContext> generatedKey = insertStatementContext.getGeneratedKeyContext();
        Preconditions.checkState(generatedKey.isPresent());
        Iterator<Comparable<?>> generatedValues = generatedKey.get().getGeneratedValues().iterator();
        int count = 0;
        List<List<Object>> parameters = insertStatementContext.getGroupedParameters();
        for (InsertValueContext each : insertStatementContext.getInsertValueContexts()) {
            InsertValue insertValueToken = result.get().getInsertValues().get(count);
            DerivedSimpleExpressionSegment expressionSegment = isToAddDerivedLiteralExpression(parameters, count)
                    ? new DerivedLiteralExpressionSegment(generatedValues.next()) : new DerivedParameterMarkerExpressionSegment(each.getParameterCount());
            insertValueToken.getValues().add(expressionSegment);
            count++;
        }
        return result.get();
    }
    
    private Optional<InsertValuesToken> findPreviousSQLToken() {
        for (SQLToken each : previousSQLTokens) {
            if (each instanceof InsertValuesToken) {
                return Optional.of((InsertValuesToken) each);
            }
        }
        return Optional.empty();
    }
    
    private boolean isToAddDerivedLiteralExpression(final List<List<Object>> parameters, final int insertValueCount) {
        return parameters.get(insertValueCount).isEmpty();
    }
}
```

上面的Token生成后，进入没有添加到Token中，最终生成的Token比上个多一个东西，如下图：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37d1a7da3db54c08839c13583ea139fd~tplv-k3u1fbpfcp-watermark.image)

### Token上传结束
上面就是最后一个了，最终的结果如下图：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/911a67845e984d99ae22327a9f253431~tplv-k3u1fbpfcp-watermark.image)

## 总结
目前探索了下SQLToken，会经过SQLTokenGenerate的处理，和上篇文章中一样，其中的Index是关键，但在这部分分析中，index其实是已经带下来了，如下图：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1de86c1974f3427abbe5c4c8eb8d0255~tplv-k3u1fbpfcp-watermark.image)

Token的生成目前感觉就是靠判断类型，然后对应的取到对应的数据

后面跟着了一下，index这些重要的原始信息，是从LogicSQL中带下来的，后面我们会跟踪这部分代码，然后看看能不能将这三部分总结串起来