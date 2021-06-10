# Crontab中文表达式解析
***
## 简介
最近工作中在使用调度框架，经常和定时表达式打交道，并且有查看表达式中文解释的需求，于是在网上搜集资料和自己进行一定的修改，写了一个Crontab表达式解析的工具类

## 详解
这个没啥好解释，看资料，自己试着动手写写，看看运行时间示例是否和自己想象的一样，就差不多了

不多说，直接上代码：

```java
package com.ninetech.cloud.bw.rpa.util;

import org.springframework.scheduling.support.CronSequenceGenerator;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

/**
 * crontab 工具类
 */
public class CrontabUtils {

    /**
     * 需要展示的表达式时间条数
     */
    private final static int size = 10;
    private static final String common = "*";
    private static final String questionMark = "?";
    private static final String hyphen = "-";
    private static final String slash = "/";
    private static final String comma = ",";
    private static final String second = "秒";
    private static final String minute = "分";
    private static final String hour = "时";
    private static final String day = "日";
    private static final String week = "周";
    private static final String month = "月";

    public static String parse(String crontab) {
        try {
            StringBuilder result = new StringBuilder("表达式：").append(crontab).append("\n");
            result.append(translateToChinese(crontab)).append("\n");
            result.append("\n").append("表达式执行时间列表示例：").append("\n");
            for (String exampleTime : examples(crontab)) {
                result.append(exampleTime).append("\n");
            }
            return result.toString();
        } catch (Exception e) {
            return "无Crontab表达式";
        }
    }

    private static List<String> examples(String crontab) {
        final CronSequenceGenerator generator = new CronSequenceGenerator(crontab);
        Date date = new Date();
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        List<String> res = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            date = generator.next(date);
            res.add(sdf.format(date));
        }
        return res;
    }

    public static String translateToChinese(String cronExp) {
        String[] tmpCorns = cronExp.split(" ");
        StringBuffer sBuffer = new StringBuffer();
        if (tmpCorns.length != 6) {
            throw new RuntimeException("请补全表达式,必须标准的cron表达式才能解析");
        }
        // 解析月
        descMonth(tmpCorns[4], sBuffer);
        // 解析周
        descWeek(tmpCorns[5], sBuffer);
        // 解析日
        descDay(tmpCorns[3], sBuffer);
        // 解析时
        descHour(tmpCorns[2], sBuffer);
        // 解析分
        descMinute(tmpCorns[1], sBuffer);
        // 解析秒
        descSecond(tmpCorns[0], sBuffer);
        return sBuffer.toString();
    }

    private static void descSecond(String s, StringBuffer sBuffer) {
        desc(s, sBuffer, second);
    }

    private static void descMinute(String s, StringBuffer sBuffer) {
        desc(s, sBuffer, minute);
    }

    private static void descHour(String s, StringBuffer sBuffer) {
        desc(s, sBuffer, hour);
    }

    private static void descDay(String s, StringBuffer sBuffer) {
        desc(s, sBuffer, day);
    }

    private static void descWeek(String s, StringBuffer sBuffer) {
        desc(s, sBuffer, week);
    }

    private static void descMonth(String s, StringBuffer sBuffer) {
        desc(s, sBuffer, month);
    }

    private static void desc(String s, StringBuffer sBuffer, String flag) {
        if (s.equals("1/1")) {
            s="*";
        }
        if (s.equals("0/0")) {
            s="0";
        }
        if (common.equals(s)) {
            if (flag.equals(week)) {
                sBuffer.append("每").append(turnWeek(s));
            } else {
                sBuffer.append("每").append(flag);
            }
            return;
        }
        if (questionMark.equals(s)) {
            return ;
        }
        if (s.contains(comma)) {
            String[] arr = s.split(comma);
            for (String value : arr) {
                if (value.length() != 0) {
                    if (flag.equals(week)) {
                        sBuffer.append(turnWeek(value)).append("和");
                    } else {
                        sBuffer.append("第").append(value).append(flag).append("和");
                    }
                }
            }
            sBuffer.deleteCharAt(sBuffer.length()-1);
            sBuffer.append("的");
            return;
        }

        if (s.contains(hyphen)) {
            String[] arr = s.split(hyphen);
            if (arr.length!=2) {
                throw new RuntimeException("表达式错误"+s);
            }
            if (flag.equals(week)) {
                sBuffer.append(turnWeek(arr[0])).append(turnWeek(arr[1]));
            } else {
                sBuffer.append("从第").append(arr[0]).append(flag).append("到第").append(arr[1]).append(flag);
            }
            return;
        }

        if (s.contains(slash)) {
            String[] arr = s.split(slash);
            if (arr.length!=2) {
                throw new RuntimeException("表达式错误"+s);
            }
            if (arr[0].equals(arr[1])||arr[0].equals("0")) {
                sBuffer.append("每").append(arr[1]).append(flag);
            }else {
                sBuffer.append("每").append(arr[1]).append(flag).append("的第").append(arr[0]).append(flag);
            }
            return;
        }

        if (flag.equals(week)) {
            sBuffer.append("每").append(turnWeek(s));
        } else {
            sBuffer.append("第").append(s).append(flag);
        }
    }

    private static String turnWeek(String week){
        switch (week) {
        case "0":
            return"周天";
        case "1":
            return"周一";
        case "2":
            return"周二";
        case "3":
            return"周三";
        case "4":
            return"周四";
        case "5":
            return"周五";
        case "6":
            return"周六";
        case "7":
            return"周日";
        default:
            return null;
        }
    }

    public static void main(String[] args) {
        for (String s : parse("0/10 3/5 1-3 * 1 0").split("\n")) {
            System.out.println(s);
        }
    }
}
```

执行的结果如下：

```
表达式：0/10 3/5 1-3 * 1 0
第1月每周天每日从第1时到第3时每5分的第3分每10秒

表达式执行时间列表示例：
2022-01-02 01:03:00
2022-01-02 01:03:10
2022-01-02 01:03:20
2022-01-02 01:03:30
2022-01-02 01:03:40
2022-01-02 01:03:50
2022-01-02 01:08:00
2022-01-02 01:08:10
2022-01-02 01:08:20
2022-01-02 01:08:30
```

## 参考链接
- [Crontab 在线工具](https://tool.lu/crontab/)
- [New in Spring 5.3: Improved Cron Expressions](https://spring.io/blog/2020/11/10/new-in-spring-5-3-improved-cron-expressions)
- [将cron表达式解析成中文,方便客户理解](https://blog.csdn.net/PiaoMiaoXiaodao/article/details/87982486)
- [spring实战代码之解析CRON表达式](https://blog.csdn.net/jimo_lonely/article/details/105418387)
- [将定时任务cron 解析成中文](https://www.cnblogs.com/panie2015/p/5553404.html)