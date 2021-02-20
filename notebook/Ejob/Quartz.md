## 入门使用

- 引入 Quartz 依赖
- 入门步骤
  1. 实现 org.quartz.Job，创建自己的 job 实现		===> 如：StatelessJobFactory
  2. 实例化 SchedulerFactory，获取 scheduler 实例，构建 jobDetail、Trigger。然后用 Scheduler 实例将 JobDetail、Trigger 加入调度容器并 start。
     - **scheduler 被停止后，除非重新实例化，否则不能重新启动；只有当scheduler启动后，即使处于暂停状态也不行，trigger才会被触发（job才会被执行）。**



## 关键 API

- **Scheduler** - 与调度程序交互的主要API。=====> 调度容器，管理所有的 Job、JobDetail、Trigger
- **Job** - 由希望由调度程序执行的组件实现的接口。主要是业务内容

```java
/**
	定义一个实现了 Job 的类，这个类表明 job 需要完成那些业务。
	当一个 Job 被 trigger 触发时，execute() 方法会被 scheduler 的一个工作线程调用；
	传递给 execute() 方法的 JobExecutionContext 对象中保存着该 job 运行时的一些信息 ，比如执行 job 的 scheduler 的引用，触发 job 的 trigger 的引用，JobDetail 对象引用，以及一些其它信息。
*/
```

- **JobExecutionContext**

```java
/**
	保存着该 job 运行时的一些信息 ，比如执行 job 的 scheduler 的引用，触发 job 的 trigger 的引用，JobDetail 对象引用，以及一些其它信息。
*/    
```

- **JobDetail** - 用于定义作业的实例。封装了一些 Job 的属性，如 Job 的名称和分组。

```java
/**
	JobDetail 对象是在将 job 加入 scheduler 时，由客户端程序（你的程序）创建的。
	它包含 job 的各种属性设置，以及用于存储 job 实例状态信息的 JobDataMap。
	在创建 JobDetail 时，我们需要将要执行的 job 的类名传给了 JobDetail，这样 scheduler 就知道了要执行何种类型的 job。
	我们还可以给通过 JobDetail 给 job 设置 name 和 group。
	每次当 scheduler 执行 job 时，在调用其 execute(…) 方法之前会创建该类的一个新的实例；执行完毕，对该实例的引用就被丢弃了，实例会被垃圾回收；这种执行策略带来的一个后果是，job 必须有一个无参的构造函数（当使用默认的 JobFactory 时）；另一个后果是，在 job 类中，不应该定义有状态的数据属性，因为在 job 的多次执行中，这些属性的值不会保留。
	
	其他特性：
	Durability：如果一个 job 是非持久的，当没有活跃的 trigger 与之关联的时候，会被自动地从 scheduler 中删除。也就是说，非持久的 job 的生命期是由 trigger 的存在与否决定的；
	RequestsRecovery：如果一个 job 是可恢复的，并且在其执行的时候，scheduler 发生硬关闭（hard shutdown)（比如运行的进程崩溃了，或者关机了），则当 scheduler 重新启动的时候，该 job 会被重新执行。此时，该 job 的 JobExecutionContext.isRecovering() 返回 true
*/
```

- **JobDataMap**：在 job 的执行过程中，可以从 JobDataMap 中取出数据。

```java
/**
	JobDataMap 中可以包含不限量的（序列化的）数据对象，在 job 实例执行的时候，可以使用其中的数据；
	JobDataMap 是 Java Map 接口的一个实现，额外增加了一些便于存取基本类型的数据的方法。
	将 job 加入到 scheduler 之前，在构建 JobDetail 时，可以将数据放入 JobDataMap。
	方式有两种:
		1.直接在构建 JobDetail 时通过 JobBuilder 的 usingJobData 方法将数据放入 JobDataMap 中。
		2.自己构建 JobDataMap，然后塞入 JobDetail 中。或者从 JobDetail 中获取往里面 put 数据。
		
    如果你在 job 类中，为 JobDataMap 中存储的数据的 key 增加 set 方法，那么 Quartz 的默认 JobFactory 实现在 job 被实例化的时候会自动调用这些 set 方法，这样你就不需要在 execute() 方法中显式地从 map 中取数据了。
    
    
    Trigger 中也有一个 JobDataMap，塞值和取值与 JobDetail 中的操作差不多。
    在 Job 执行时，JobExecutionContext 中的 JobDataMap 虽然为我们提供了很多的便利。它是 JobDetail 中的 JobDataMap 和 Trigger 中的 JobDataMap 的并集，但是如果存在相同的数据，则后者会覆盖前者的值。
    如果在 job 运行中，从 JobExecutionContext 分别获取 JobDetail 和 Trigger，然后再分别获取它们中的 JobDataMap，就算有相同的 key，也是没有什么影响的，只有当调用 getMergedJobDataMap() 方法时才会获取它们的 JobDataMap 合并后的数据，或者用 set 方法获取也是获取到覆盖后的值。
*/
```

- **Job 实例**：**正在执行的job**

```java
/**
	我们可以为 job 类创建多个实例（JobDetail），然后这些实例采用不同的作业参数来运行。
	一个Job可以有多个Trigger，但多个Job不能对应同一个Trigger
*/
```



- **Trigger**（即触发器） - 定义执行给定作业的计划的组件。主要封装一些调度要素，比如调度周期、开始时间、结束时间等等。

  - **一个Job可以有多个Trigger，但多个Job不能对应同一个Trigger**

  - 常用的公共属性：

    - **jobKey 属性**：当 trigger 触发时被执行的 job 的身份；
    - **startTime 属性**：设置 trigger 第一次触发的时间；该属性的值是 java.util.Date 类型，表示某个指定的时间点；有些类型的 trigger，会在设置的startTime 时立即触发，有些类型的 trigger，表示其触发是在 startTime 之后开始生效。比如，现在是1月份，你设置了一个 trigger–“在每个月的第5天执行”，然后你将 startTime 属性设置为4月1号，则该 trigger 第一次触发会是在几个月以后了(即4月5号)。
    - **endTime 属性**：表示 trigger 失效的时间点。比如，”每月第5天执行”的 trigger，如果其 endTime是7月1号，则其最后一次执行时间是6月5号。
    - **priority** **属性**：优先级

    ```java
    /**
    	如果你的 trigger 很多(或者 Quartz 线程池的工作线程太少)，Quartz 可能没有足够的资源同时触发所有的 trigger；
    	这种情况下，你可能希望控制哪些 trigger 优先使用 Quartz 的工作线程，要达到该目的，可以在 trigger 上设置 priority 属性。
    	比如，你有N个 trigger 需要同时触发，但只有Z个工作线程，优先级最高的Z个 trigger 会被首先触发。如果没有为 trigger 设置优先级，trigger 使用默认优先级，值为5；priority 属性的值可以是任意整数，正数、负数都可以。
    	注意：只有同时触发的 trigger 之间才会比较优先级。10:59 触发的 trigger 总是在 11:00 触发的 trigger 之前执行。
    	注意：如果 trigger 是可恢复的，在恢复后再调度时，优先级与原 trigger 是一样的。
    */
    ```

    - **misfire Instructions 属性**：错过触发。

    ```java
    /**
    	如果 scheduler 关闭了，或者 Quartz 线程池中没有可用的线程来执行 job，此时持久性的 trigger 就会错过(miss)其触发时间，即错过触发(misfire)。
    	当下次调度器启动或者有可以线程时，会检查处于 misfire 状态的 Trigger。而 misfire 的状态值决定了调度器如何处理这个 Trigger。
    	不同类型的 trigger，有不同的 misfire 机制。它们默认都使用“智能机制(smart policy)”，即根据 trigger 的类型和配置动态调整行为。
    */
    ```

    - **calendar 属性**：日历实例。

```java
/**
	所有类型的 trigger 都有 TriggerKey 这个属性，表示 trigger 的身份；除此之外，trigger 还有很多其它的公共属性。这些属性，在构建 trigger 的时候可以通过 TriggerBuilder 设置。
	private TriggerKey key;
    private String description;
    private Date startTime = new Date();
    private Date endTime;
    private int priority = 5;
    private String calendarName;
    private JobKey jobKey;
    private JobDataMap jobDataMap = new JobDataMap();
    private ScheduleBuilder<?> scheduleBuilder = null;
*/
```



- **JobKey 和 TriggerKey**：JobKey 是表明 Job 身份的一个对象，里面封装了 Job 的 name 和 group，TriggerKey 同理。
  - JobBuilder 和 TriggerBuilder 中的 withIdentity 方法直接根据 job 的 name 和 group 创建新的 JobKey 和 TriggerKey
- **JobBuilder** - 用于定义/构建 JobDetail 实例，用于定义作业的实例。
- **TriggerBuilder** - 用于定义/构建触发器实例。
- **SchedulerBuilder** 定义不同类型的调度计划
  - CronScheduleBuilder 定义corn表达式类型的schedule
  - SimpleScheduleBuilder 定义简单类型的schedule



## Job状态与并发（ejob 较少使用）

- 只创建一个job类，然后创建多个与该job关联的JobDetail实例，每一个实例都有自己的属性集和JobDataMap，最后，将所有的实例都加到scheduler中	====> 涉及到 job 的并发

```java
/**
	在 job 类上可以加入一些注解，这些注解会影响 job 的状态和并发性。
	
	@DisallowConcurrentExecution：将该注解加到 job 类上，告诉 Quartz 不要并发地执行同一个 job 定义（这里指特定的 job 类）的多个实例。该限制是针对JobDetail 的，而不是 job 类的。但是我们认为（在设计 Quartz 的时候）应该将该注解放在 job 类上，因为 job 类的改变经常会导致其行为发生变化。
	
	@PersistJobDataAfterExecution：将该注解加在 job 类上，告诉 Quartz 在成功执行了 job 类的 execute 方法后（没有发生任何异常），更新 JobDetail 中 JobDataMap 的数据，使得该 job（即 JobDetail）在下一次执行的时候，JobDataMap 中是更新后的数据，而不是更新前的旧数据。和 @DisallowConcurrentExecution 注解一样，尽管注解是加在 job 类上的，但其限制作用是针对 job 实例的，而不是 job 类的。由 job 类来承载注解，是因为 job 类的内容经常会影响其行为状态（比如，job 类的 execute 方法需要显式地“理解”其”状态“）。
	
	如果你使用了 @PersistJobDataAfterExecution 注解，我们强烈建议你同时使用 @DisallowConcurrentExecution 注解，因为当同一个 job（JobDetail） 的两个实例被并发执行时，由于竞争，JobDataMap 中存储的数据很可能是不确定的。
	
	简单来说，针对每一个 Job 类，Quartz 的建议是加上 @DisallowConcurrentExecution 注解，因为当有两个或多个有状态的 JobDetail 实例（针对的是同一个Job 类，即 JobBuilder.newJob(Class <? extends Job> jobClass) 方法中的参数是同一个）放入 Scheduler 容器时，如果这时，你还建立了两个 Trigger 来触发这个 Job：一个每五分钟触发，另一个也是每五分钟触发。假如这两个 Trigger 试图在同一时刻触发 Job，框架是不允许这种事情发生的，因为这会有并发数据问题，JobDataMap 中存储的数据很可能是不确定的。
*/
```



## 日历（calendar）

Quartz 的 Calendar 对象(不是 java.util.Calendar 对象)可以在定义和存储 trigger 的时候与 trigger 进行关联。
**Calendar 用于从 trigger 的调度计划中排除时间段。比如，可以创建一个 trigger，每个工作日的上午 9:30 执行，然后增加一个 Calendar，排除掉所有的商业节日。**
Calendar 排除时间段的单位可以精确到毫秒，同时 Quartz 提供了很多开箱即用的实现来满足我们的不同需求，如下

| 类名                                     | 用法                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| org.quartz.impl.calendar.HolidayCalendar | 指定特定的日期，比如20140613 精度到天                        |
| org.quartz.impl.calendar.AnnualCalendar  | 指定每年的哪一天，精度是天                                   |
| org.quartz.impl.calendar.MonthlyCalendar | 指定每月的几号，可选值为1-31，精度是天                       |
| org.quartz.impl.calendar.WeeklyCalendar  | 指定每星期的星期几，可选值比如为java.util.Calendar.SUNDAY，精度是天 |
| org.quartz.impl.calendar.DailyCalendar   | 指定每天的时间段（rangeStartingTime, rangeEndingTime)，格式是HH:MM[:SS[:mmm]]，也就是最大精度可以到毫秒 |
| org.quartz.impl.calendar.CronCalendar    | 指定Cron表达式，精度取决于Cron表达式，也就是最大精度可以到秒 |

**注意，所有的Calendar既可以是排除，也可以是包含**

- HolidayCalendar 与 AnnualCalendar 的区别
  - HolidayCalendar 考虑的是年，需要指定确定的年月日
  - AnnualCalendar 是指每一年的哪一天。

所以通常情况下，比如我想排除每年的 5 月 1 日，用 AnnualCalendar 就比较方便，若用 HolidayCalendar，就需要注册多个 HolidayCalendar 到 Scheduler 中。

### AnnualCalendar（Ejob 只用到这个类）

创建与注册

```
	// 构建 SchedulerFactory 实例
	SchedulerFactory schedFact = new StdSchedulerFactory();
	// 获取 Scheduler 实例
	Scheduler scheduler = schedFact.getScheduler();
	// 创建 AnnualCalendar 对象
	AnnualCalendar annualday = new AnnualCalendar();
	Calendar laborDay = GregorianCalendar.getInstance();
	// 设置 5 月 1 日，这里需要注意在 Calendar 月份的基础上 +1 才是真实的月份
	laborDay.set(Calendar.MONTH, Calendar.MAY + 1);
	laborDay.set(Calendar.DATE, 1);
	// 排除每年 5 月 1 日，true 为排除，false 为包含
	annualday .setDayExcluded(laborDay, true);
	// 向 Scheduler 注册日历
	scheduler.addCalendar("annualday", annualday, true, true);
```

**需要注意一下几点**

- AnnualCalendar 就算设置了年，也不会生效，它会在每一年都生效，具体逻辑可以查看源码
- AnnualCalendar.setDayExcluded(java.util.Calendar day, boolean exclude) 方法中的 exclude 是用来决定是包含还是排除
- setDayExcluded 有一个重载的方法，可以传入一个 ArrayList<java.util.Calendar>，一次性加入多个java.util.Calendar，但是该方法默认 exclude 为 true，就是默认是排除
- Scheduler.addCalendar(String calName, Calendar calendar, boolean replace, boolean updateTriggers) 方法中的参数
  - replace 表示是否覆盖之前设置的相同名称的 Calendar ，若为 false，当 add 相同名称的 Calendar  时，会抛 SchedulerException
  - updateTriggers 表示若已存在的 triggers 已经引用 了相同名称的 Calendar，是否更新 triggers 绑定的Calendar

绑定Trigger

```java
SimpleTrigger triggerB = TriggerBuilder.newTrigger()
				                               .withIdentity("helloTriggerB", "helloB")
				                               .startNow()
				                               .withSchedule(SimpleScheduleBuilder
				                               .simpleSchedule()
				                               .withIntervalInSeconds(5).repeatForever())
				                               // 指定 trigger 使用 Calendar
				                               .modifiedByCalendar("annualday")
				                               .build();
```

### HolidayCalendar

该日历与 AnnualCalendar 差不多，区别就是设置的 year 是有效的，如果你希望在未来 5 年内都排除 5 月 1 日这一天，你需要添加 5 个日期,分别是 2019-5-1,2020-5-1,2021-5-1,2022-5-1,2023-5-1

```java
		// 构建 SchedulerFactory 实例
		SchedulerFactory schedFact = new StdSchedulerFactory();
		// 获取 Scheduler 实例
		Scheduler scheduler = schedFact.getScheduler();
		// 创建 HolidayCalendar 对象
		HolidayCalendar holidays = new HolidayCalendar();
		Calendar laborDay1 = new GregorianCalendar(2019, 5, 1);
		Calendar laborDay2 = new GregorianCalendar(2020, 5, 1);
		Calendar laborDay3 = new GregorianCalendar(2021, 5, 1);
		Calendar laborDay4 = new GregorianCalendar(2022, 5, 1);
		Calendar laborDay5 = new GregorianCalendar(2023, 5, 1);
		
		holidays.addExcludedDate(laborDay1.getTime());
		holidays.addExcludedDate(laborDay2.getTime());
		holidays.addExcludedDate(laborDay3.getTime());
		holidays.addExcludedDate(laborDay4.getTime());
		holidays.addExcludedDate(laborDay5.getTime());
		// 向 Scheduler 注册日历
		scheduler.addCalendar("holidays", holidays, true, true);
```

**需要注意的是**

- HolidayCalendar 只能做排除操作
- HolidayCalendar 里面维护了一个 TreeSet 集合来存放通过 addExcludedDate 加入的排除日期，所以若是需要排除多个日期，多次调用 addExcludedDate 方法即可。

### MonthlyCalendar

月日历，你可以定义一个月当中的若干天不触发。比如我想排除每个月的 2,4,5 号不触发

```
		MonthlyCalendar monthlyCalendar = new MonthlyCalendar();
		monthlyCalendar.setDayExcluded(2, true);
		monthlyCalendar.setDayExcluded(4, true);
		monthlyCalendar.setDayExcluded(6, true);
		// 向 Scheduler 注册日历
		scheduler.addCalendar("monthlyCalendar", monthlyCalendar, true, true);
```

### WeeklyCalendar

星期日历，可以定义每周内周几不触发。例如设置每周四不触发

```
		WeeklyCalendar weeklyCalendar = new WeeklyCalendar();
		weeklyCalendar.setDayExcluded(Calendar.THURSDAY, true);
		// 向 Scheduler 注册日历
		scheduler.addCalendar("weeklyCalendar", weeklyCalendar, true, true);
```

**需要注意的是**

- 当我们定义了 WeeklyCalendar ，周六和周日就默认被排除了，如果想要不排除，需要自己把这两天通过setDayExcluded 设置为false,即不排除。可以查看如下源码

  ```
   public WeeklyCalendar() {
      this(null, null);
  }
  public WeeklyCalendar(Calendar baseCalendar, TimeZone timeZone) {
      super(baseCalendar, timeZone);
  	//默认排除周六和周日
      excludeDays[java.util.Calendar.SUNDAY] = true;
      excludeDays[java.util.Calendar.SATURDAY] = true;
      excludeAll = areAllDaysExcluded();
  }
  ```

- WeeklyCalendar .setDayExcluded(int wday, boolean exclude) 中的参数务必用 java.util.Calendar 中的定义的常量，因为国际上一周的开始第一天是周日，可以查看 java.util.Calendar 中定义的周常量，是从 SUNDAY（1）开始的。

### DailyCalendar

时间范围日历，定义一个时间范围，可以让触发器在这个时间范围内触发，或者在这个时间范围内不触发，每一个 DailyCalendar 的实例只能设置一次时间范围，并且这个时间范围不能超过一天的边界。
例如我要排除每天当中晚上 20 点到 21 点

```java
		// 构建 SchedulerFactory 实例
		SchedulerFactory schedFact = new StdSchedulerFactory();
		// 获取 Scheduler 实例
		Scheduler scheduler = schedFact.getScheduler();
		
		Calendar start = GregorianCalendar.getInstance();
		start.setTime(DateBuilder.dateOf(20, 0, 0));
		
		Calendar end = GregorianCalendar.getInstance();
		end.setTime(DateBuilder.dateOf(21, 0, 0));
		
		DailyCalendar dailyCalendar = new DailyCalendar(start, end);
		dailyCalendar.setInvertTimeRange(false);
		// 向 Scheduler 注册日历
		scheduler.addCalendar("dailyCalendar", dailyCalendar, true, true);
```

**需要注意的是**

- 可以通过 DailyCalendar 中的参数 invertTimeRange 来设置在定义的时间范围内不触发，还是在定义的时间范围外不触发
  - 根据源码中的描述 invertTimeRange = false (默认值）表示在定义的时间范围内触发 invertTimeRange = true 表示在定义的时间范围外不触发
- DailyCalendar 的构造参数支持直接输出字符串，比如：new DailyCalendar(“20:00:00”, “21:00:00”)
- DailyCalendar 可以支持精确到毫秒级别的控制

### CronCalendar

Cron 表达式日历，可以写一个表达式来排除一个时间范围，比如可以设置为排除所有的时间,但是工作时间除外，也就是 在早 8 点-晚 5 点触发，其他时间暂停。

```Java
		// 构建 SchedulerFactory 实例
		SchedulerFactory schedFact = new StdSchedulerFactory();
		// 获取 Scheduler 实例
		Scheduler scheduler = schedFact.getScheduler();
		try {
			CronCalendar cronCalendar= new CronCalendar("* * 0-7,18-23 ? * *");
			// 向 Scheduler 注册日历
			scheduler.addCalendar("cronCalendar", cronCalendar, true, true);
		} catch (ParseException e) {
			e.printStackTrace();
		}
```

### 组合日历

我们在构建 Trigger 时只能通过 modifiedByCalendar(“日历的 name”)方法关联一个已经注册到调度引擎的日历对象，这种情况已经能够满足我们大部分的需求。
但是如果有跟复杂的需求，比如我们需要在每周的周一至周五，排除晚上8点到24点，情况下按照 Trigger 配置的周期运行。可以用三种办法实现以上需求

- CronTrigger，这个稍后在详细介绍，通过构建 cron 表达式来实现。
- 组合日历，每种日历都有一个构造方法，可以传递一个日历对象进来，通过这样的方式可以构建一个复杂的日历对象。
- CronTrigger + 组合日历的方式。

组合日历使用的例子：

```java
	// 构建 SchedulerFactory 实例
		SchedulerFactory schedFact = new StdSchedulerFactory();
		// 获取 Scheduler 实例
		Scheduler scheduler = schedFact.getScheduler();
		
		DailyCalendar dailyCalendar = new DailyCalendar("20:00:00", "23:59:59");
		dailyCalendar.setInvertTimeRange(false);
		
		// 默认是排除周六和周日的
		WeeklyCalendar weeklyCalendar = new WeeklyCalendar(dailyCalendar);
		
		// 向 Scheduler 注册日历
		scheduler.addCalendar("weeklyCalendar", weeklyCalendar, true, true);
```

写一个时间间隔的日历 dailyCalendar,将其作为参数传递给 weeklyCalendar 就可以了，这样引擎在计算日历日期的时候会先判断 dailyCalendar 的时间范围，然后再判断 weeklyCalendar 是时间范围，当条件都满足的是否，触发器才会被触发。



## SimpleTrigger

SimpleTrigger 主要可以满足一些简单的调度需求，比如在某个时间点执行一次，或者在某个时间点开始执行，然后以某一个时间间隔来重复执行。SimpleTrigger 的属性包括：开始时间、结束时间、重复次数以及重复的间隔。其中关于结束时间需要有特殊注意的地方

### SimpleTrigger相关属性设置API

| 方法                                                | 设置类                | 描述                               | 备注                                 |
| --------------------------------------------------- | --------------------- | ---------------------------------- | ------------------------------------ |
| startAt(Date triggerStartTime)                      | TriggerBuilder        | 设置任务开始时间                   | 用DateBuilder可以轻松获取参数Date    |
| startNow()                                          | TriggerBuilder        | 设置任务开始为当前时间             |                                      |
| endAt(Date triggerEndTime)                          | TriggerBuilder        | 设置任务介绍时间                   | 用DateBuilder可以轻松获取参数Date    |
| withIntervalInMilliseconds(long intervalInMillis)   | SimpleScheduleBuilder | 设置任务执行间隔，单位：毫秒       |                                      |
| withIntervalInSeconds(int intervalInSeconds)        | SimpleScheduleBuilder | 设置任务执行间隔，单位：秒         |                                      |
| withIntervalInMinutes(int intervalInMinutes)        | SimpleScheduleBuilder | 设置任务执行间隔，单位：分         |                                      |
| withIntervalInHours(int intervalInHours)            | SimpleScheduleBuilder | 设置任务执行间隔，单位：小时       |                                      |
| withRepeatCount(int triggerRepeatCount)             | SimpleScheduleBuilder | 设置任务重复次数                   |                                      |
| repeatForever()                                     | SimpleScheduleBuilder | 无限期重复                         |                                      |
| repeatSecondlyForever(int seconds)                  | SimpleScheduleBuilder | 设置以多少秒的间隔永远重复         | 重载的方法没有参数，默认1秒          |
| repeatMinutelyForever(int minutes)                  | SimpleScheduleBuilder | 设置以多少分的间隔永远重复         | 重载的方法没有参数，默认1分          |
| repeatHourlyForever(int hours)                      | SimpleScheduleBuilder | 设置以多少小时的间隔永远重复       | 重载的方法没有参数，默认1小时        |
| repeatSecondlyForTotalCount(int count, int seconds) | SimpleScheduleBuilder | 以给定的秒数数间隔重复给定的次数   | 重载的方法没有seconds参数，默认1秒   |
| repeatMinutelyForTotalCount(int count, int minutes) | SimpleScheduleBuilder | 以给定的分钟数数间隔重复给定的次数 | 重载的方法没有seconds参数，默认1分钟 |
| repeatHourlyForTotalCount(int count, int hours)     | SimpleScheduleBuilder | 以给定的小时数间隔重复给定的次数   | 重载的方法没有seconds参数，默认1小时 |

上面表格中的方法基本包含了设置 SimpleTrigger 所有方法，可以看出主要是通过 TriggerBuilder 和 SimpleScheduleBuilder 来设置的。其中关于结束时间我们需要注意 **endTime 属性的值会覆盖设置重复次数的属性值**。很好理解，假如你设置开始时间为早上 8 点，结束时间为早上 9 点，每间隔 10 分钟执行一次，你是不需要去设置重复次数的，设置了也会被 Quertz 自动计算的重复次数覆盖。
另一方面，TriggerBuilder(以及 Quartz 的其它 builder)会为那些没有被显式设置的属性选择合理的默认值。比如：如果你没有调用 withIdentity(…)方法，TriggerBuilder 会为 trigger 生成一个随机的名称；如果没有调用 startAt(…)方法，则默认使用当前时间，即 trigger 立即生效。

- 例子代码
  指定开始时间和结束时间，每隔10秒执行一次

```java
		Date startTime =DateBuilder.dateOf(12, 12, 12, 20, 4, 2019);
		Date endTime =DateBuilder.dateOf(12, 14, 12, 20, 4, 2019);
		Trigger trigger = TriggerBuilder
				.newTrigger().withIdentity("simperTrigger1", "test1")
				.startAt(startTime)
				.withSchedule(SimpleScheduleBuilder
						.simpleSchedule()
						.withIntervalInSeconds(10))
				.endAt(endTime)
				.build();
```

### SimpleTrigger的Misfire策略

在 Trigger 中关于 SimpleTrigger 的 Misfire 策略常量如下，并且可以通过 SimpleSchedulerBuilder 设置。

#### MISFIRE_INSTRUCTION_FIRE_NOW

设置方法：
withMisfireHandlingInstructionFireNow
含义：
——以当前时间为触发频率立即触发执行
——执行至 EndTIme 的剩余周期次数
——以调度或恢复调度的时刻为基准的周期频率，EndTIme 根据剩余次数和当前时间计算得到
——调整后的 EndTIme 会略大于根据 StartTime 计算的到的 EndTIme 值

#### MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY

设置方法：
withMisfireHandlingInstructionIgnoreMisfires
含义：
——以错过的第一个频率时间立刻开始执行
——重做错过的所有频率周期
——当下一次触发频率发生时间大于当前时间以后，按照 Interval 的依次执行剩下的频率
——共执行 RepeatCount+1次

#### MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_CO

设置方法：withMisfireHandlingInstructionNowWithExistingCount
含义：
——以当前时间为触发频率立即触发执行
——执行至 EndTIme 的剩余周期次数
——以调度或恢复调度的时刻为基准的周期频率，EndTIme 根据剩余次数和当前时间计算得到
——调整后的 EndTIme 会略大于根据 StartTime 计算的到的 EndTIme 值

#### MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT

设置方法：withMisfireHandlingInstructionNowWithRemainingCount
含义：
——以当前时间为触发频率立即触发执行
——执行至EndTIme的剩余周期次数
——以调度或恢复调度的时刻为基准的周期频率，EndTIme根据剩余次数和当前时间计算得到
——调整后的EndTIme会略大于根据StartTime计算的到的EndTIme值

#### MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT

设置方法：withMisfireHandlingInstructionNextWithRemainingCount
含义：
——不触发立即执行
——等待下次触发频率周期时刻，执行至EndTIme的剩余周期次数
——以StartTime为基准计算周期频率，并得到EndTIme
——即使中间出现pause，resume以后保持EndTIme时间不变

#### MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT

设置方法：withMisfireHandlingInstructionNextWithExistingCount
含义：
——此指令导致trigger忘记原始设置的StartTime和repeat-count
——触发器的repeat-count将被设置为剩余的次数
——这样会导致后面无法获得原始设定的StartTime和repeat-count值

#### 默认策略

查看源码，SimpleScheduleBuilder 中 misfireInstruction 的默认值是 MISFIRE_INSTRUCTION_SMART_POLICY，这是所有 Trigger 默认的 MisFire 策略，这个策略会根据 Trigger 的状态和类型来自动调节 MisFire 策略。
查看源码可以看到，若设置为默认策略，则按照以下规则来选择 MisFire 策略

- 如果重复计数为 0，则指令将解释为 MISFIRE_INSTRUCTION_FIRE_NOW。
- 如果重复计数为 REPEAT_INDEFINITELY，则指令将解释为MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT。 警告：如果触发器具有非空的结束时间，则使用 MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT 可能会导致触发器在 misfire 时间范围内到达结束时，不会再次触发。
- 如果重复计数大于 0，则指令将解释为MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT。

## CronTrigger

CronTrigger 能精确的制定时间和时间间隔，可以通过设置 Cron Expressions 来配置 CronTrigger，Cron Expressions 是由七个子表达式组成的字符串，用于描述日程表的各个细节。具体怎么设置可与参照[在线Cron表达式生成器](http://cron.qqe2.com/),你甚至可以把它内嵌到你的项目中，用于生成 cron 表达式。

### 构建 CronTrigger

CronTrigger 实例使用 TriggerBuilder（用于触发器的主要属性）和 CronScheduleBuilder（对于 CronTrigger 特定的属性）构建。
例如，构建一个触发器，每天早上 9 点至下午 6 点运行，每 30 分钟执行一次。
先通过在线CronB表达式生成器生成表达式：0 0/30 9-18 * * ?

```Java
		//构建 CronTrigger
		TriggerBuilder.newTrigger()
		.withSchedule(CronScheduleBuilder.cronSchedule("0 0/30 9-18 * * ? "))
		.forJob("jobA", "groupA")
		.build();
```

### CronTrigger Misfire

CronTrigger 的 Misfire 有三种如下，可以通过 CronSchedulerBuilder 设置

#### MISFIRE_INSTRUCTION_DO_NOTHING

设置方法：withMisfireHandlingInstructionDoNothing
含义：
——不触发立即执行
——等待下次 Cron 触发频率到达时刻开始按照 Cron 频率依次执行

#### （Ejob 使用）MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY

设置方法：withMisfireHandlingInstructionIgnoreMisfires
含义：从上一次执行的时间开始调度
——以错过的第一个频率时间立刻开始执行
——重做错过的所有频率周期后
——当下一次触发频率发生时间大于当前时间后，再按照正常的 Cron 频率依次执行

#### MISFIRE_INSTRUCTION_FIRE_ONCE_NOW

设置方法：withMisfireHandlingInstructionFireAndProceed
含义：
——以当前时间为触发频率立刻触发一次执行
——然后按照 Cron 频率依次执行

#### 默认策略

在SimpleTrigger 中已经提到所有 trigger 的默认 Misfire 策略都是 MISFIRE_INSTRUCTION_SMART_POLICY，SimpleTrigger 会根据 tirgger 的状态来调整具体的 Misfire 策略，而 CronTrigger 的默认 Misfire 策略会被 CronTrigger 解释为 MISFIRE_INSTRUCTION_FIRE_NOW，具体可以参照 CronTrigger 实现类的源码