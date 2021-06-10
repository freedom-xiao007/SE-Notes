# Mysql
***

## 问题集锦
### order by 导致分页出现重复数据问题
由于 login_time 不唯一导致分页出现重复数据

在使用 order by 的时候, 在本身需要排序的 目标字段 之后再加上一个 唯一字段 (比如PK或者UNIQUE字段), 保证顺序的唯一性.

## 参考链接
- [Go语言：解决数据库中null值的问题](https://blog.csdn.net/qq_15437667/article/details/78780945)
- [Go查询MySQLdatetime列，设置时区](https://loesspie.com/2018/11/27/go-mysql-datetime-timezone/)
- [前后端分离时间数据和格式化的问题](https://www.jianshu.com/p/c427ac291c14)
- [深入理解GO时间处理(time.Time)](https://studygolang.com/articles/11975)
- [go 语言系列 操作Mysql](https://www.cnblogs.com/flying1819/articles/8832613.html)
- [order by 导致分页出现重复数据问题](https://blog.csdn.net/hbtj_1216/article/details/80619102)