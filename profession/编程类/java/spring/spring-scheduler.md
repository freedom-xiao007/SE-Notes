# Spring Scheduler 使用记录
***

### 获取执行时间
```java
import org.springframework.scheduling.support.CronSequenceGenerator;
Date date = new Date();
CronSequenceGenerator cronSequenceGenerator = new CronSequenceGenerator("0 */5 * * * ?");

Date time1 = cronSequenceGenerator.next(date);//下次执行时间
Date time2 = cronSequenceGenerator.next(time1);
Date time3 = cronSequenceGenerator.next(time2);
long l = time1.getTime() -(time3.getTime() -time2.getTime());
Date date1 = new Date(l);//上一次任务执行时间
```

## 参考链接
- [springboot根据cron获取任务执行上次和下次执行时间](https://blog.csdn.net/s573626822/article/details/107540293)