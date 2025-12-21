[toc]

## 一、为什么需要新的日期时间API？

在Java 8之前，Java开发者主要使用 `java.util.Date` 和 `java.util.Calendar` 类来处理日期和时间。然而，这些旧API存在诸多问题，给日常开发带来了不少困扰。

**旧版API的主要缺陷**：

1. **设计混乱**：日期类分散在`java.util`、`java.sql`和`java.text`等多个包中，缺乏统一性
2. **线程不安全**：`Date`和`Calendar`都是可变类，在多线程环境下需要额外的同步处理
3. **API设计反人类**：月份从0开始（0表示一月），年份从1900开始，这容易导致编程错误
4. **时区处理复杂**：需要依赖`TimeZone`和`Calendar`类进行复杂的时区转换

**新版API的核心优势**：

- **不可变性**：所有日期时间对象都是不可变的，线程安全得到保障
- **流畅的API设计**：方法命名直观，支持链式调用
- **清晰的类型分离**：将日期、时间、日期时间等概念分离为不同的类
- **默认ISO标准**：遵循ISO-8601国际标准，减少理解成本

下面的表格对比了新旧API的主要区别：

| 特性 | 旧API（Date/Calendar） | 新API（java.time） |
|------|----------------------|-------------------|
| **线程安全** | 不安全，需要同步 | 不可变，天然线程安全 |
| **API设计** | 混乱，不易使用 | 直观，方法命名清晰 |
| **月份表示** | 0-11（0表示一月） | 1-12（符合习惯） |
| **格式化** | SimpleDateFormat线程不安全 | DateTimeFormatter线程安全 |
| **类型区分** | Date包含日期和时间 | 明确区分LocalDate、LocalTime等 |

## 二、java.time包核心类介绍

Java 8引入的`java.time`包采用**领域驱动设计**理念，将日期时间概念划分为多个清晰的类型。

### 2.1 核心类关系图

```
java.time包
├── LocalDate (仅日期)
├── LocalTime (仅时间)
├── LocalDateTime (日期+时间)
├── Instant (时间戳)
├── Duration (时间间隔)
├── Period (日期间隔)
└── DateTimeFormatter (格式化)
```

### 2.2 不可变性的重要性

新API中所有类都是**不可变的**，这意味着一旦创建，对象的状态就不能被修改。任何修改操作都会返回一个新的对象。

**不可变性的优势**：
- **线程安全**：无需担心多线程环境下的并发修改
- **预测性强**：对象状态在生命周期内保持不变
- **易于缓存**：可以安全地缓存和重用对象

## 三、LocalDate：纯日期处理

`LocalDate`表示一个**不带时间**和时区的日期，如"2025-10-04"。它适用于只需要日期信息的场景，如生日、节假日、会议日期等。

### 3.1 创建LocalDate对象

```java
// 获取当前日期
LocalDate today = LocalDate.now();
System.out.println("当前日期: " + today); // 输出: 2025-10-04

// 创建指定日期
LocalDate birthday = LocalDate.of(1990, 1, 15);
LocalDate independenceDay = LocalDate.of(1949, Month.OCTOBER, 1);

// 通过字符串解析（ISO格式）
LocalDate parsedDate = LocalDate.parse("2025-12-25");
```

### 3.2 常用操作示例

```java
public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate today = LocalDate.now();
        
        // 获取日期组成部分
        int year = today.getYear();
        Month month = today.getMonth();
        int dayOfMonth = today.getDayOfMonth();
        DayOfWeek dayOfWeek = today.getDayOfWeek();
        
        System.out.printf("年: %d, 月: %s, 日: %d, 星期: %s%n", 
                         year, month, dayOfMonth, dayOfWeek);
        
        // 日期加减操作
        LocalDate tomorrow = today.plusDays(1);
        LocalDate nextWeek = today.plusWeeks(1);
        LocalDate nextMonth = today.plusMonths(1);
        LocalDate nextYear = today.plusYears(1);
        
        // 日期调整（TemporalAdjuster的强大功能）
        LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
        LocalDate lastDayOfMonth = today.with(TemporalAdjusters.lastDayOfMonth());
        
        // 日期比较
        boolean isBefore = today.isBefore(nextWeek);
        boolean isAfter = today.isAfter(nextWeek);
        boolean isEqual = today.isEqual(today);
        
        // 计算两个日期之间的周期
        LocalDate startDate = LocalDate.of(2025, 1, 1);
        Period period = Period.between(startDate, today);
        System.out.printf("相差: %d年%d个月%d天%n", 
                         period.getYears(), period.getMonths(), period.getDays());
    }
}
```

### 3.3 实战案例：生日提醒系统

```java
public class BirthdayReminder {
    public static void checkBirthdayReminder(LocalDate birthday) {
        LocalDate today = LocalDate.now();
        LocalDate nextBirthday = birthday.withYear(today.getYear());
        
        // 如果今年生日已过，计算明年生日
        if (nextBirthday.isBefore(today)) {
            nextBirthday = nextBirthday.plusYears(1);
        }
        
        Period untilBirthday = Period.between(today, nextBirthday);
        
        if (untilBirthday.getMonths() == 0 && untilBirthday.getDays() <= 7) {
            System.out.println("生日快到了！还有 " + untilBirthday.getDays() + " 天");
        }
        
        // 检查是否是今天生日
        if (today.getMonth() == birthday.getMonth() && 
            today.getDayOfMonth() == birthday.getDayOfMonth()) {
            System.out.println("生日快乐！");
        }
    }
}
```

## 四、LocalTime：纯时间处理

`LocalTime`表示一个**不带日期**和时区的时间，如"14:30:45.123"。它适用于只需要时间信息的场景，如营业时间、会议时间等。

### 4.1 创建LocalTime对象

```java
// 获取当前时间
LocalTime now = LocalTime.now();
System.out.println("当前时间: " + now); // 输出: 14:30:45.123

// 创建指定时间
LocalTime meetingTime = LocalTime.of(14, 30); // 14:30
LocalTime preciseTime = LocalTime.of(9, 15, 30, 100000000); // 09:15:30.100

// 通过字符串解析
LocalTime parsedTime = LocalTime.parse("20:45:30");
```

### 4.2 常用操作示例

```java
public class LocalTimeExample {
    public static void main(String[] args) {
        LocalTime now = LocalTime.now();
        
        // 获取时间组成部分
        int hour = now.getHour();
        int minute = now.getMinute();
        int second = now.getSecond();
        int nano = now.getNano();
        
        System.out.printf("时: %d, 分: %d, 秒: %d, 纳秒: %d%n", 
                         hour, minute, second, nano);
        
        // 时间加减操作
        LocalTime inOneHour = now.plusHours(1);
        LocalTime in30Minutes = now.plusMinutes(30);
        LocalTime minus15Seconds = now.minusSeconds(15);
        
        // 时间比较
        LocalTime startTime = LocalTime.of(9, 0);
        LocalTime endTime = LocalTime.of(17, 0);
        
        boolean isWorkingHour = now.isAfter(startTime) && now.isBefore(endTime);
        System.out.println("是否在工作时间内: " + isWorkingHour);
        
        // 计算时间间隔（Duration）
        Duration untilEndOfWork = Duration.between(now, endTime);
        System.out.println("距离下班还有: " + untilEndOfWork.toHours() + "小时" + 
                          untilEndOfWork.toMinutesPart() + "分钟");
    }
}
```

### 4.3 实战案例：营业时间检查

```java
public class BusinessHoursChecker {
    private final LocalTime openingTime;
    private final LocalTime closingTime;
    
    public BusinessHoursChecker(LocalTime opening, LocalTime closing) {
        this.openingTime = opening;
        this.closingTime = closing;
    }
    
    public boolean isOpen() {
        LocalTime now = LocalTime.now();
        return !now.isBefore(openingTime) && !now.isAfter(closingTime);
    }
    
    public Duration getTimeUntilOpen() {
        LocalTime now = LocalTime.now();
        if (now.isBefore(openingTime)) {
            return Duration.between(now, openingTime);
        }
        return Duration.ZERO;
    }
    
    // 使用示例
    public static void main(String[] args) {
        BusinessHoursChecker store = new BusinessHoursChecker(
            LocalTime.of(9, 0), LocalTime.of(21, 0));
        
        System.out.println("店铺是否营业: " + store.isOpen());
        if (!store.isOpen()) {
            Duration untilOpen = store.getTimeUntilOpen();
            System.out.println("距离营业还有: " + 
                untilOpen.toHours() + "小时" + untilOpen.toMinutesPart() + "分钟");
        }
    }
}
```

## 五、LocalDateTime：日期时间处理

`LocalDateTime`是`LocalDate`和`LocalTime`的组合，表示一个**不带时区**的日期时间，如"2025-10-04T14:30:45"。它适用于需要同时处理日期和时间的场景。

### 5.1 创建LocalDateTime对象

```java
// 获取当前日期时间
LocalDateTime now = LocalDateTime.now();
System.out.println("当前日期时间: " + now); // 输出: 2025-10-04T14:30:45.123

// 创建指定日期时间
LocalDateTime meetingDateTime = LocalDateTime.of(2025, 10, 15, 14, 30);
LocalDateTime preciseDateTime = LocalDateTime.of(2025, Month.DECEMBER, 25, 20, 45, 30);

// 通过LocalDate和LocalTime组合
LocalDate date = LocalDate.of(2025, 10, 4);
LocalTime time = LocalTime.of(14, 30);
LocalDateTime dateTime = LocalDateTime.of(date, time);

// 通过字符串解析
LocalDateTime parsedDateTime = LocalDateTime.parse("2025-10-04T14:30:45");
```

### 5.2 常用操作示例

```java
public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime now = LocalDateTime.now();
        
        // 获取日期时间组成部分
        System.out.println("年: " + now.getYear());
        System.out.println("月: " + now.getMonthValue());
        System.out.println("日: " + now.getDayOfMonth());
        System.out.println("时: " + now.getHour());
        System.out.println("分: " + now.getMinute());
        System.out.println("秒: " + now.getSecond());
        
        // 日期时间加减操作
        LocalDateTime inOneWeek = now.plusWeeks(1);
        LocalDateTime in3Hours = now.plusHours(3);
        LocalDateTime lastMonth = now.minusMonths(1);
        
        // 转换为LocalDate或LocalTime
        LocalDate datePart = now.toLocalDate();
        LocalTime timePart = now.toLocalTime();
        
        // 日期时间比较
        LocalDateTime appointment = LocalDateTime.of(2025, 10, 10, 10, 0);
        if (now.isBefore(appointment)) {
            Duration duration = Duration.between(now, appointment);
            System.out.println("距离预约还有: " + duration.toDays() + "天");
        }
        
        // 与LocalDate和LocalTime的互操作
        LocalDateTime combined = datePart.atTime(timePart);
        LocalDate extractedDate = combined.toLocalDate();
        LocalTime extractedTime = combined.toLocalTime();
    }
}
```

### 5.3 实战案例：会议调度系统

```java
public class MeetingScheduler {
    private Map<String, LocalDateTime> meetings = new HashMap<>();
    
    public void scheduleMeeting(String name, LocalDateTime dateTime) {
        // 检查时间冲突
        boolean hasConflict = meetings.values().stream()
            .anyMatch(existing -> isOverlapping(existing, dateTime));
        
        if (hasConflict) {
            throw new IllegalArgumentException("时间冲突，无法安排会议");
        }
        
        meetings.put(name, dateTime);
        System.out.println("会议 '" + name + "' 已安排在: " + dateTime);
    }
    
    public void checkReminders() {
        LocalDateTime now = LocalDateTime.now();
        
        meetings.forEach((name, dateTime) -> {
            Duration duration = Duration.between(now, dateTime);
            if (duration.toHours() <= 24 && duration.toHours() >= 0) {
                System.out.println("提醒: 会议 '" + name + "' 将在 " + 
                                 duration.toHours() + " 小时后开始");
            }
        });
    }
    
    private boolean isOverlapping(LocalDateTime existing, LocalDateTime newMeeting) {
        Duration gap = Duration.between(existing, newMeeting);
        return Math.abs(gap.toHours()) < 2; // 假设会议时长2小时
    }
    
    // 使用示例
    public static void main(String[] args) {
        MeetingScheduler scheduler = new MeetingScheduler();
        
        scheduler.scheduleMeeting("项目评审", 
            LocalDateTime.of(2025, 10, 5, 14, 0));
        scheduler.scheduleMeeting("团队周会", 
            LocalDateTime.of(2025, 10, 6, 10, 0));
        
        scheduler.checkReminders();
    }
}
```

## 六、格式化与解析

日期时间的格式化和解析是日常开发中的常见需求，Java 8提供了`DateTimeFormatter`类来解决这个问题。

### 6.1 基本格式化操作

```java
public class DateTimeFormattingExample {
    public static void main(String[] args) {
        LocalDateTime now = LocalDateTime.now();
        
        // 使用预定义的格式化器
        String isoFormat = now.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        System.out.println("ISO格式: " + isoFormat);
        
        // 自定义格式
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH时mm分ss秒");
        String customFormat = now.format(formatter);
        System.out.println("自定义格式: " + customFormat);
        
        // 解析字符串为日期时间
        String dateTimeStr = "2025-10-04 14:30:00";
        DateTimeFormatter parser = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime parsedDateTime = LocalDateTime.parse(dateTimeStr, parser);
        System.out.println("解析结果: " + parsedDateTime);
        
        // 本地化格式化
        DateTimeFormatter germanFormatter = DateTimeFormatter
            .ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(Locale.GERMAN);
        String germanFormat = now.format(germanFormatter);
        System.out.println("德语格式: " + germanFormat);
    }
}
```

### 6.2 实战案例：多格式日期解析器

```java
public class MultiFormatDateParser {
    private static final List<DateTimeFormatter> FORMATTERS = Arrays.asList(
        DateTimeFormatter.ISO_LOCAL_DATE_TIME,
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"),
        DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm"),
        DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm"),
        DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH时mm分ss秒")
    );
    
    public static LocalDateTime parse(String dateTimeString) {
        for (DateTimeFormatter formatter : FORMATTERS) {
            try {
                return LocalDateTime.parse(dateTimeString, formatter);
            } catch (DateTimeParseException e) {
                // 尝试下一种格式
            }
        }
        throw new IllegalArgumentException("无法解析的日期时间格式: " + dateTimeString);
    }
    
    // 使用示例
    public static void main(String[] args) {
        String[] testDates = {
            "2025-10-04T14:30:45",
            "2025-10-04 14:30:45",
            "2025/10/04 14:30",
            "04.10.2025 14:30",
            "2025年10月04日 14时30分45秒"
        };
        
        for (String dateStr : testDates) {
            LocalDateTime dateTime = parse(dateStr);
            System.out.println("解析 '" + dateStr + "' -> " + dateTime);
        }
    }
}
```

## 七、时间间隔计算

在实际应用中，经常需要计算两个日期或时间之间的间隔。Java 8提供了`Period`和`Duration`类来处理这种需求。

### 7.1 Period与Duration的区别

- **Period**：用于计算两个日期之间的间隔（年、月、日）
- **Duration**：用于计算两个时间之间的间隔（小时、分钟、秒、纳秒）

### 7.2 实战案例：项目时间计算器

```java
public class ProjectTimeCalculator {
    public static void calculateProjectTimeline(LocalDate startDate, 
                                               LocalDate endDate,
                                               LocalTime startTime,
                                               LocalTime endTime) {
        // 计算总天数
        Period totalPeriod = Period.between(startDate, endDate);
        long totalDays = ChronoUnit.DAYS.between(startDate, endDate);
        
        System.out.printf("项目总时长: %d年%d个月%d天（共%d天）%n",
                         totalPeriod.getYears(), totalPeriod.getMonths(), 
                         totalPeriod.getDays(), totalDays);
        
        // 计算每日工作时间
        Duration dailyWorkHours = Duration.between(startTime, endTime);
        System.out.printf("每日工作时间: %d小时%d分钟%n",
                         dailyWorkHours.toHours(), dailyWorkHours.toMinutesPart());
        
        // 计算总工作时间
        long totalWorkMinutes = totalDays * dailyWorkHours.toMinutes();
        System.out.printf("总工作时间: %d天 × %d分钟 = %d分钟%n",
                         totalDays, dailyWorkHours.toMinutes(), totalWorkMinutes);
    }
    
    // 使用示例
    public static void main(String[] args) {
        LocalDate startDate = LocalDate.of(2025, 1, 1);
        LocalDate endDate = LocalDate.of(2025, 12, 31);
        LocalTime startTime = LocalTime.of(9, 0);
        LocalTime endTime = LocalTime.of(18, 0);
        
        calculateProjectTimeline(startDate, endDate, startTime, endTime);
    }
}
```

## 八、最佳实践与性能优化

### 8.1 使用建议

1. **选择合适的类型**：
    - 只需要日期 → `LocalDate`
    - 只需要时间 → `LocalTime`
    - 需要日期时间但无需时区 → `LocalDateTime`
    - 需要精确时间点 → `Instant`

2. **避免重复创建**：对于频繁使用的日期时间对象，考虑缓存或重用

3. **注意性能**：在性能敏感的场景，避免在循环中频繁创建格式化器

### 8.2 常见陷阱与解决方案

```java
public class CommonPitfalls {
    // 错误：在循环中重复创建格式化器
    public void badPractice() {
        for (int i = 0; i < 1000; i++) {
            DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd"); // 性能差
            String formatted = LocalDate.now().format(formatter);
        }
    }
    
    // 正确：重用格式化器
    public void goodPractice() {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
        for (int i = 0; i < 1000; i++) {
            String formatted = LocalDate.now().format(formatter); // 性能好
        }
    }
    
    // 错误：忽略闰年等特殊情况
    public void dateCalculationPitfall() {
        LocalDate date = LocalDate.of(2024, 2, 29); // 闰年
        LocalDate nextYear = date.plusYears(1); // 2025-02-28（自动处理）
        System.out.println("明年同日: " + nextYear);
    }
}
```

## 总结

Java 8的新日期时间API通过`LocalDate`、`LocalTime`和`LocalDateTime`等类，为日期时间处理提供了**强大而直观**的解决方案。这些类的不可变性确保了线程安全，流畅的API设计提高了代码可读性，清晰的类型分离让开发者能够更精确地表达业务逻辑。

**核心要点回顾**：
1. **选择正确的类型**：根据需求在`LocalDate`、`LocalTime`和`LocalDateTime`之间选择
2. **利用不可变性**：所有修改操作返回新对象，天然线程安全
3. **掌握常用操作**：熟练使用加减、比较、格式化等方法
4. **理解间隔计算**：正确使用`Period`和`Duration`进行时间间隔计算

在实际项目中，建议全面采用新的日期时间API，逐步替换旧的`Date`和`Calendar`类，这将显著提高代码的**可维护性和可靠性**。

> **下一步学习**：在掌握了本地日期时间处理之后，下一讲我们将深入探讨时区处理（`ZonedDateTime`）、时间戳（`Instant`）以及更高级的日期时间操作技巧。