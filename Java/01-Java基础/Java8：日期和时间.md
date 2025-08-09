# 1. LocalDate对象
```java
// 生成LocalDate对象的两种方式
LocalDate now = LocalDate.now();						//生成当天日期
LocalDate of = LocalDate.of(2021, Month.DECEMBER, 11);	//生成指定日期
LocalDate of = LocalDate.of(2021, 11, 11);


System.out.println(now);								//   2021-12-04
System.out.println(of);									//   2021-12-11
System.out.println(localDate.plusYears(1));				//   2022-11-11
System.out.println(localDate.getDayOfMonth());			//   11
System.out.println(localDate.getDayOfYear());			//   315
System.out.println(localDate.getDayOfWeek());			//   THURSDAY
System.out.println(localDate.getYear());				//	 2021
System.out.println(localDate.getMonth());				//   NOVEMBER
System.out.println(localDate.getMonthValue());			//   11
System.out.println(localDate.isAfter(LocalDate.of(2020,11,11)));		//  true
System.out.println(localDate.isLeapYear());				//  false
```

## localDate的方法

| 方法                                       | 描述                                |
| ---------------------------------------- | --------------------------------- |
| now、of                                   | 生成一个Local对象，要么从当前时间生成，要么从指定的年月日生成 |
| plusDays、plusWeeks、plusMonths、plusYears  | 在当前的localDate上加上一定量的天、星期、月、年      |
| minusDays、minusWeeks、minusMonths、minusYears | 在当前的localDate上减去一定量的天、星期、月、年      |
| getDayOfMonth                            | 获取月的日期，在1到31之间                    |
| getDayOfYear                             | 获取年的日期，在1到366之间                   |
| getDayOfWeek                             | 获取星期的日期，返回DayOfWeek的枚举值           |
| getYear                                  | 获取LocalDate的年份                    |
| getMonth、getMonthValue                   | 获取月份Month的枚举值，或者1-12的数字           |
| isAfter、isBefore                         | 与另一个LocalDate对象做比较                |
| isLeapYear                               | 如果是闰年、返回true                      |




除了LocalDate之外，还有MonthDay、YearMonth、Year等类可以描述部分日期，如下：

```java
MonthDay monthDay = MonthDay.of(12, 12);
System.out.println(monthDay);         				//   --12-12
System.out.println(monthDay.getMonth());			//   DECEMBER
```



# 2. LocalTime 对象
```java
LocalTime now = LocalTime.now();
LocalTime of = LocalTime.of(11, 11, 11);

System.out.println(now);							//   16:27:21.964
System.out.println(of);								//   11:11:11
System.out.println(localTime.plusHours(1));			//   12:11:11
System.out.println(localTime.plusMinutes(1));		//   11:12:11
System.out.println(localTime.plusSeconds(1));		//   11:11:12
System.out.println(localTime.getHour());			//   11
System.out.println(localTime.getMinute());			//   11
System.out.println(localTime.isAfter(LocalTime.of(11,11,10)));	//   true
```

