# Java Quartz 相关使用记录
***
### 获取下次执行时间
#### 使用quartz获取最近执行的时间
```java
/**
 * <dependency>
 *    <groupId>org.quartz-scheduler</groupId>
 *    <artifactId>quartz</artifactId>
 * </dependency>
 *
 */

import org.quartz.TriggerUtils;
import org.quartz.impl.triggers.CronTriggerImpl;
import org.quartz.spi.OperableTrigger;

/**
 * Latest list.
 * <p>
 * String cron = "0 0/2 * * * ?";
 *
 * @param cron  the cron
 * @param count the count
 * @return the list
 * @throws ParseException the parse exception
 */
public static List<LocalDateTime> latest(String cron, int count) throws ParseException {
    CronTriggerImpl cronTriggerImpl = new CronTriggerImpl();
    // 设置cron表达式
    cronTriggerImpl.setCronExpression(cron);
    // 根据count获取最近执行时间
    List<Date> dateList = TriggerUtils.computeFireTimes(cronTriggerImpl, null, count);
    return dateList.stream().map(Java8DateTimeUtils::toJava8).collect(Collectors.toList());
}
```

## 参考链接
- [使用Java计算cron最近执行周期](https://iogogogo.github.io/2021/04/02/java-cron-calc-cycle/)