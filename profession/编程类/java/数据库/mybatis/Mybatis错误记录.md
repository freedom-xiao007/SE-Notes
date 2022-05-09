# Mybatis 错误记录
***
## 实体与数据库中数据对不上
### 相关错误数据
- nested exception is org.apache.ibatis.executor.result.ResultMapException: Error attempting to get column 'description' from result set.  Cause: java.lang.NumberFormatException: For input string: "description"
- org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.exceptions.PersistenceException: Cause: java.lang.IndexOutOfBoundsException: Index 2 out of bounds for length 2

### 错误说明
- java的类反射拿到的数据不完整

### 解决办法
不返回实体，返回map

## Connection Java-MySql : Public Key Retrieval is not allowed
add *allowPublicKeyRetrieval=true* in connect config, example:

jdbc:mysql://localhost:3306/db?allowPublicKeyRetrieval=true&useSSL=false

### MyBatis报错 Cannot determine value type from string 'xxxxxx'
```java

There was an unexpected error (type=Internal Server Error, status=500).
Error attempting to get column 'act_name' from result set. Cause: java.sql.SQLDataException: Cannot determine value type from string '201577D0510' ; Cannot determine value type from string '201577D0510'; nested exception is java.sql.SQLDataException: Cannot determine value type from string '201577D0510'
org.springframework.dao.DataIntegrityViolationException: Error attempting to get column 'u_account' from result set.  Cause: java.sql.SQLDataException: Cannot determine value type from string '201577D0510'
; Cannot determine value type from string '201577D0510'; nested exception is java.sql.SQLDataException: Cannot determine value type from string '201577D0510'
	at org.springframework.jdbc.support.SQLExceptionSubclassTranslator.doTranslate(SQLExceptionSubclassTranslator.java:84)
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:72)
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:81)
	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:73)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:446)

```


1. 数据库字段和实体类的属性不匹配
2.重写了实体类的有参构造后，没有写无参构造,补上无参构造即可

```java
//无参构造
    public Act(){}

	//有参构造
    public Act(Integer actId, Integer likeNum) {
        this.id = actId;
        this.likeNum = likeNum;
    }
```


## 参考链接
- [MyBatis报错 Cannot determine value type from string 'xxxxxx'](https://blog.csdn.net/p__jx/article/details/104185357)