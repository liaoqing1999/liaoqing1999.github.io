---

layout: post
title: 定时任务quartz
date: 2020-09-29 09:29:51
tags: Quartz
categories: Quartz
---

# 定时任务Quartz框架

## 前言

最近工作上接触到了定时任务，开始了解Quartz框架

## 参考链接

W3Cschool文档：https://www.w3cschool.cn/quartz_doc/

Quartz官方地址：http://www.quartz-scheduler.org/documentation/

## Quartz主要部分

Quartz是一个开源的任务调度框架，有着简单、轻量的优势，但在集群环境中Quartz的表现不怎么友好，如果使用集群环境可以考虑[xxl-job](https://www.xuxueli.com/xxl-job/)

- Scheduler - 与调度程序交互的主要API，主要用于开始、暂停、删除任务等操作。

- Job - 你想要调度器执行的任务组件需要实现的接口，任务的实现类，实现 Job接口，execute方法为任务的主体实现部分

- JobDetail - 用于定义作业的实例。job的实例化，一个job类 可以用于构造多个JobDetail 

- Trigger（即触发器） - 定义执行给定作业的计划的组件。实现定时功能（多少天、分等执行一次）

- JobBuilder - 用于定义/构建 JobDetail 实例，用于定义作业的实例。简单来说JobBuilder 将一个job类实例化成JobDetail 实例，JobDetail jobDetail = JobBuilder.newJob(jobClass).withIdentity("myTrigger", "group1").build();

- TriggerBuilder - 用于定义/构建触发器实例。Trigger的构建方法

- Scheduler 的生命期，从 SchedulerFactory 创建它时开始，到 Scheduler 调用shutdown() 方法时结束；Scheduler 被创建后，可以增加、删除和列举 Job 和 Trigger，以及执行其它与调度相关的操作（如暂停 Trigger）。但是，Scheduler 只有在调用 start() 方法后，才会真正地触发 trigger（即执行 job）


## 简单例子

maven引入jar包

```
<dependency>
	<groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.1</version>
    </exclusions>
</dependency>
```

测试类   HelloJob类实现job接口即可

```
 public class QuartzTest {

      public static void main(String[] args) {

          try {
              Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
              scheduler.start();
			  JobDetail job = newJob(HelloJob.class)
			  .withIdentity("job1","group1")
			  .build();
			  Trigger trigger = newTrigger()
			  .withIdentity("trigger1", "group1")
			  .startNow().withSchedule(simpleSchedule()
              .withIntervalInSeconds(40)
              .repeatForever()).build();
              scheduler.scheduleJob(job, trigger);
              scheduler.shutdown();
          } catch (SchedulerException se) {
              se.printStackTrace();
          }
      }
  }
```

## Job与JobDetail介绍

Job就是任务的实体，其中的execute方法为任务的执行方法，每一个job都需要实现Job接口

JobDetail则用于包装Job实例所包含的属性，如任务名、组名等

```
 //最常见的构建方法
 JobDetail job = JobBuilder.newJob(HelloJob.class)
      .withIdentity("myJob", "group1") // name "myJob", group "group1"
      .build();

```

在job类中我们可以看到execute方法只有一个参数JobExecutionContext，那如果我们需要向job传递参数应该怎么实现呢？

```
public void execute(JobExecutionContext context) throws JobExecutionException{
	
}
```

答案是通过JobDataMap，是JobDetail的一部分

```
//这样定义我们就通过JobDataMap 存储了两个变量jobSays和myFloatValue
JobDetail job = newJob(DumbJob.class)
      .withIdentity("myJob", "group1") // name "myJob", group "group1"
      .usingJobData("jobSays", "Hello World!")
      .usingJobData("myFloatValue", 3.141f)
      .build();
      
//在execute方法中我们就可以通过key取出来了      
  	  JobDataMap dataMap = context.getJobDetail().getJobDataMap();
      String jobSays = dataMap.getString("jobSays");
      float myFloatValue = dataMap.getFloat("myFloatValue");
```

Job的注解

```
/*
@DisallowConcurrentExecution：将该注解加到job类上，告诉Quartz不要并发地执行同一个job定义（这里指特定的job类）的多个实例。请注意这里的用词。拿前一小节的例子来说，如果“SalesReportJob”类上有该注解，则同一时刻仅允许执行一个“SalesReportForJoe”实例，但可以并发地执行“SalesReportForMike”类的一个实例。所以该限制是针对JobDetail的，而不是job类的。但是我们认为（在设计Quartz的时候）应该将该注解放在job类上，因为job类的改变经常会导致其行为发生变化。
*/
@DisallowConcurrentExecution
/*
@PersistJobDataAfterExecution：将该注解加在job类上，告诉Quartz在成功执行了job类的execute方法后（没有发生任何异常），更新JobDetail中JobDataMap的数据，使得该job（即JobDetail）在下一次执行的时候，JobDataMap中是更新后的数据，而不是更新前的旧数据。和 @DisallowConcurrentExecution注解一样，尽管注解是加在job类上的，但其限制作用是针对job实例的，而不是job类的。由job类来承载注解，是因为job类的内容经常会影响其行为状态（比如，job类的execute方法需要显式地“理解”其”状态“）。
如果你使用了@PersistJobDataAfterExecution注解，我们强烈建议你同时使用@DisallowConcurrentExecution注解，因为当同一个job（JobDetail）的两个实例被并发执行时，由于竞争，JobDataMap中存储的数据很可能是不确定的。
*/
@PersistJobDataAfterExecution
```

## Triggers

Triggers用于规定任务何时执行、按何种规律执行（如每月一号执行一次备份数据任务）

Triggers有两种常用子类：SimpleTrigger、CronTrigger

Triggers的公共属性：

- jobKey属性：当trigger触发时被执行的job的身份；
- startTime属性：设置trigger第一次触发的时间；该属性的值是java.util.Date类型，表示某个指定的时间点；有些类型的trigger，会在设置的startTime时立即触发，有些类型的trigger，表示其触发是在startTime之后开始生效。比如，现在是1月份，你设置了一个trigger–“在每个月的第5天执行”，然后你将startTime属性设置为4月1号，则该trigger第一次触发会是在几个月以后了(即4月5号)。
- endTime属性：表示trigger失效的时间点。比如，”每月第5天执行”的trigger，如果其endTime是7月1号，则其最后一次执行时间是6月5号。

优先级：

如果你的trigger很多(或者Quartz线程池的工作线程太少)，Quartz可能没有足够的资源同时触发所有的trigger；这种情况下，你可能希望控制哪些trigger优先使用Quartz的工作线程，要达到该目的，可以在trigger上设置priority属性。

priority属性的值可以是任意整数，正数、负数都可以。

注意：只有同时触发的trigger之间才会比较优先级。10:59触发的trigger总是在11:00触发的trigger之前执行。

注意：如果trigger是可恢复的，在恢复后再调度时，优先级与原trigger是一样的。

```
//优先级、开始时间、结束时间
TriggerBuilder.newTrigger().withPriority(1).startAt(startDate).endAt(endDate).withIdentity("job1","group1");

```

错过触发(misfire Instructions)

如果scheduler关闭了，或者Quartz线程池中没有可用的线程来执行job，此时持久性的trigger就会错过(miss)其触发时间，即错过触发(misfire)。

不同类型的trigger，有不同的misfire机制。它们默认都使用“智能机制(smart policy)”，即根据trigger的类型和配置动态调整行为。

当scheduler启动的时候，查询所有错过触发(misfire)的持久性trigger。然后根据它们各自的misfire机制更新trigger的信息。

### SimpleTrigger

SimpleTrigger可以满足的调度需求是：在具体的时间点执行一次，或者在具体的时间点执行，并且以指定的间隔重复执行若干次。

SimpleTrigger的属性包括：开始时间、结束时间、重复次数以及重复的间隔。

```
 //指定时间触发，每隔10秒执行一次，重复10次：
 trigger = newTrigger()
        .withIdentity("trigger3", "group1")
        .startAt(myTimeToStartFiring)
        .withSchedule(simpleSchedule()
            .withIntervalInSeconds(10)
            .withRepeatCount(10)) 
        .forJob(myJob)                   
        .build();
```

SimpleTrigger Misfire策略

- withMisfireHandlingInstructionFireNow ：立即执行一次
- withMisfireHandlingInstructionIgnoreMisfires：重做错过的所有频率周期
- withMisfireHandlingInstructionNextWithExistingCount：不立即执行，即放弃错过的
- withMisfireHandlingInstructionNowWithExistingCount：立即执行一次
- withMisfireHandlingInstructionNextWithRemainingCount：不立即执行，即放弃错过的
- withMisfireHandlingInstructionNowWithRemainingCount：立即执行一次
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT：此指令导致trigger忘记原始设置的starttime和repeat-count，触发器的repeat-count将被设置为剩余的次数

### CronTrigger

基于日历，可以指定号时间表，如每周五，但同时也可以设置startTime和endTime

```
 //建立一个触发器，每隔一分钟，每天上午8点至下午5点之间：
 trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(cronSchedule("0 0/2 8-17 * * ?"))
    .forJob("myJob", "group1")
    .build();
```

CronTrigger Misfire策略

- withMisfireHandlingInstructionDoNothing：不触发立即执行，即放弃错过的
- withMisfireHandlingInstructionIgnoreMisfires：重做错过的所有频率周期
- withMisfireHandlingInstructionFireAndProceed：立即执行一次