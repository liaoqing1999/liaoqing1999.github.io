---
layout: post
title: 定时任务quartz 管理和日志
date: 2020-09-29 14:06:44
tags: Quartz
categories: Quartz
---

# 定时任务quartz 管理和日志

## 前言

上一篇介绍了quartz的结构和基础用法，现在则是利用反射来实现定时任务的抽取，实现动态配置定时任务和相关日志管理

## 思路

两个实体任务表和任务日志表。

任务表存放：id、任务名、任务组名、调用类名方法名、调用目标参数、cron表达式、错误策略等

```
CREATE TABLE `td_sm_task` (
  `task_id` varchar(64) NOT NULL COMMENT '任务ID',
  `task_name` varchar(64) NOT NULL DEFAULT '' COMMENT '任务名称',
  `task_group` varchar(64) DEFAULT 'DEFAULT' COMMENT '任务组名',
  `invoke_target` varchar(500) NOT NULL COMMENT '注解value',
  `invoke_param` varchar(500) DEFAULT NULL COMMENT '调用目标参数',
  `cron_expression` varchar(255) DEFAULT '' COMMENT 'cron执行表达式',
  `misfire_policy` varchar(20) DEFAULT '3' COMMENT '计划执行错误策略（1立即执行 2执行一次 3放弃执行）',
  `concurrent` char(1) DEFAULT '1' COMMENT '是否并发执行（0允许 1禁止）',
  `status` char(1) DEFAULT '0' COMMENT '状态（0正常 1暂停）',
  `create_by` varchar(64) DEFAULT '' COMMENT '创建者',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_by` varchar(64) DEFAULT '' COMMENT '更新者',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  `remark` varchar(500) DEFAULT '' COMMENT '备注信息',
  PRIMARY KEY (`task_id`,`task_name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='定时任务调度表';
```

日志表存放：任务名、运行时间、运行结果等

```
CREATE TABLE `td_sm_task_log` (
  `task_log_id` varchar(64) NOT NULL COMMENT '任务日志ID',
  `task_name` varchar(64) NOT NULL COMMENT '任务名称',
  `task_group` varchar(64) DEFAULT NULL COMMENT '任务组名',
  `invoke_target` varchar(500) NOT NULL COMMENT '调用目标字符串',
  `invoke_param` varchar(500) DEFAULT NULL COMMENT '调用目标参数',
  `task_message` varchar(2500) DEFAULT NULL COMMENT '日志信息',
  `status` char(1) DEFAULT '0' COMMENT '执行状态（0正常 1失败）',
  `exception_info` varchar(2000) DEFAULT '' COMMENT '异常信息',
  `start_time` datetime DEFAULT NULL COMMENT '创建时间',
  `stop_time` datetime DEFAULT NULL COMMENT '停止时间',
  PRIMARY KEY (`task_log_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='定时任务调度日志表';
```

实现思路

根据任务表里面的实体建立对应策略的任务实例，实例方法中，根据调用方法和参数反射调用对应方法，每次调用保存日志。

应用启动时，查找所有任务表，并开启所有运行中的任务

## Job类

三个类 一个基础类 负责实现job接口和每次调用时新增日志

两个继承类 主要是区分并发和非并发

```
//基础类
public abstract class AbstractQuartzTask implements Job
{
    private static final Logger log = LoggerFactory.getLogger(AbstractQuartzTask.class);

    /**
     * 线程本地变量
     */
    private static ThreadLocal<Date> threadLocal = new ThreadLocal<>();

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException
    {
    	TaskDTO sysTask = new TaskDTO();
        BeanCopierUtil.copy(context.getMergedJobDataMap().get(ScheduleConstants.TASK_PROPERTIES), sysTask);
        try
        {
            before(context, sysTask);
            if (sysTask != null)
            {
                doExecute(context, sysTask);
            }
            after(context, sysTask, null);
        }
        catch (Exception e)
        {
            log.error("任务执行异常  - ：", e);
            after(context, sysTask, e);
        }
    }

    /**
     * 执行前
     *
     * @param context 工作执行上下文对象
     * @param sysTask 系统计划任务
     */
    protected void before(JobExecutionContext context, TaskDTO sysTask)
    {
        threadLocal.set(new Date());
    }

    /**
     * 执行后
     *
     * @param context 工作执行上下文对象
     * @param sysTask 系统计划任务
     */
    protected void after(JobExecutionContext context, TaskDTO sysTask, Exception e)
    {
        Date startTime = threadLocal.get();
        threadLocal.remove();

        final TaskLogDTO sysTaskLog = new TaskLogDTO();
        sysTaskLog.setTaskName(sysTask.getTaskName());
        sysTaskLog.setTaskGroup(sysTask.getTaskGroup());
        sysTaskLog.setInvokeTarget(sysTask.getInvokeTarget());
        sysTaskLog.setStartTime(startTime);
        sysTaskLog.setStopTime(new Date());
        long runMs = sysTaskLog.getStopTime().getTime() - sysTaskLog.getStartTime().getTime();
       
        String stautsMsg = "";
        String errorMsg = "";
        if (e != null)
        {
            sysTaskLog.setStatus(Constants.FAIL);
            errorMsg = StringUtils.substring(ExceptionUtil.getExceptionMessage(e), 0, 2000);
            stautsMsg = "执行失败！";
        }
        else
        {
            sysTaskLog.setStatus(Constants.SUCCESS);
            stautsMsg = "执行成功！";
        }
        sysTaskLog.setTaskMessage(stautsMsg+" 总共耗时：" + runMs + "毫秒"+errorMsg);
        // 写入数据库当中
        SpringUtils.getBean(TaskLogService.class).addTaskLog(sysTaskLog);
    }

    /**
     * 执行方法，由子类重载
     *
     * @param context 工作执行上下文对象
     * @param sysTask 系统计划任务
     * @throws Exception 执行过程中的异常
     */
    protected abstract void doExecute(JobExecutionContext context, TaskDTO sysTask) throws Exception;
}

//非并发
public class QuartzTaskExecution extends AbstractQuartzTask
{
    @Override
    protected void doExecute(JobExecutionContext context, TaskDTO sysJob) throws Exception
    {
    	TaskInvokeUtil.invokeMethod(sysJob);
    }
}

//并发子类
@DisallowConcurrentExecution
public class QuartzDisallowConcurrentExecution extends AbstractQuartzTask
{
    @Override
    protected void doExecute(JobExecutionContext context, TaskDTO sysJob) throws Exception
    {
        TaskInvokeUtil.invokeMethod(sysJob);
    }
}

```

## 反射

有两种方式，一种是通过bean名+方法名确认，一种是通过注解，我使用的是通过注解

通过注解  在应用启动 的时候扫描所有bean并查找其中添加了注解的方法，存储到静态变量里面。

```
// 在bean创建完成后 查找使用了C2SysTask注解的方法
@Service
public class TaskBeanPostProcessor implements BeanPostProcessor {
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		Class<? extends Object> cla = bean.getClass();
		// 代理类的名字
		String agentName = cla.getName();
		// 原始类名字
		String originalName = cla.getName();
		// 如果被代理（事务管理、切面等） 找到原始类
		if (originalName.contains(Constants.SPRING_AGENT_NAME) || originalName.contains(Constants.SPRING_AGENT_SYMBOL)) {
			originalName = originalName.substring(0, originalName.indexOf("$$"));
			try {
				cla = Class.forName(originalName);
			} catch (ClassNotFoundException e) {
				throw new RuntimeException(e);
			}
		}
		Method[] methods = cla.getMethods();
		// 循环方法 查找添加C2SysTask注解的方法
		for (Method m : methods) {
			if (m.isAnnotationPresent(C2SysTask.class)) {

				C2SysTask annotation = m.getAnnotation(C2SysTask.class);
				TaskAnnotationBean beanC2SysTask = TaskInvokeUtil.getBeanFromC2SysTask(annotation.value());
				// 如果C2SysTask注解value值存在相同的值
				if (beanC2SysTask != null) {
					String msg = "C2SysTask注解检测到了相同的value！->>" + annotation.value() + "\n";
					msg += "\t" + beanC2SysTask.getOriginalName() + "中的方法" + beanC2SysTask.getMethodName();
					msg += "和" + originalName + "中的方法" + m.getName() + "在注解C2SysTask中的value冲突！";
					try {
						throw new TaskAnnotationException(msg);
					} catch (TaskAnnotationException e) {
						e.printStackTrace();
						System.exit(0);
					}
				}
				TaskAnnotationBean taskAnnotationBean = new TaskAnnotationBean();
				taskAnnotationBean.setValue(annotation.value());
				taskAnnotationBean.setBeanName(beanName);
				taskAnnotationBean.setAgentName(agentName);
				taskAnnotationBean.setOriginalName(originalName);
				taskAnnotationBean.setMethodName(m.getName());
				Parameter[] parameters = m.getParameters();
				List<C2SysTaskParameter> parametersList = new ArrayList<C2SysTaskParameter>();
				// 循环拿出参数
				for (Parameter p : parameters) {
					C2SysTaskParameter c2Parameter = new C2SysTaskParameter();
					c2Parameter.setName(p.getName());
					c2Parameter.setType(p.getType());
					c2Parameter.setParameterizedType(p.getParameterizedType());
					parametersList.add(c2Parameter);
				}
				taskAnnotationBean.setParametersList(parametersList);
				Constants.C2_SYS_TASK_ANNOTATION_BEANLIST.add(taskAnnotationBean);
			}
		}
		return bean;
	}

}
```

反射执行工具类

```
public class TaskInvokeUtil {
	/**
	 * 执行方法
	 *
	 * @param sysTask
	 *            系统任务
	 */
	public static void invokeMethod(TaskDTO sysTask) throws Exception {
		String invokeTarget = sysTask.getInvokeTarget();
		String paramStr = sysTask.getInvokeParam();
		TaskAnnotationBean task = getBeanFromC2SysTask(invokeTarget);
		if (task != null) {
			String beanName = task.getBeanName();
			String methodName = task.getMethodName();
			Object bean = null;
			List<Object[]> methodParams = getMethodParams(paramStr, task.getParametersList());
			if (!isValidClassName(beanName)) {
				bean = SpringUtils.getBean(beanName);
			} else {
				bean = Class.forName(beanName).newInstance();
			}
			invokeMethod(bean, methodName, methodParams);
		}

	}

	/**
	 * 调用任务方法
	 *
	 * @param bean
	 *            目标对象
	 * @param methodName
	 *            方法名称
	 * @param methodParams
	 *            方法参数
	 */
	private static void invokeMethod(Object bean, String methodName, List<Object[]> methodParams)
			throws NoSuchMethodException, SecurityException, IllegalAccessException, IllegalArgumentException,
			InvocationTargetException {
		if (StringUtils.isNotNull(methodParams) && methodParams.size() > 0) {
			Method method = bean.getClass().getDeclaredMethod(methodName, getMethodParamsType(methodParams));
			method.invoke(bean, getMethodParamsValue(methodParams));
		} else {
			Method method = bean.getClass().getDeclaredMethod(methodName);
			method.invoke(bean);
		}
	}

	/**
	 * 校验是否为为class包名
	 * 
	 * @param str
	 *            名称
	 * @return true是 false否
	 */
	public static boolean isValidClassName(String invokeTarget) {
		return StringUtils.countMatches(invokeTarget, ".") > 1;
	}

	/**
	 * 获取method方法参数相关列表
	 * 
	 * @param invokeTarget
	 *            目标字符串
	 * @param list
	 * @return method方法相关参数列表
	 */
	public static List<Object[]> getMethodParams(String invokeTarget, List<C2SysTaskParameter> list) {
		
		List<Object[]> classs = new LinkedList<>();
		if(StringUtils.isEmpty(invokeTarget)) {
			return classs;
		}
		String[] methodParams = invokeTarget.split(",");
		for (int i = 0; i < list.size(); i++) {
			C2SysTaskParameter parameter = list.get(i);
			Class<?> type = parameter.getType();
			String par = methodParams[i];
			Object cast =null;
			if(type == Integer.class || type == Integer.TYPE) {
				cast = Integer.valueOf(par);
			}else if(type == Double.class || type == Double.TYPE) {
				cast = Double.valueOf(par);
			}else if(type == Float.class || type == Float.TYPE) {
				cast = Float.valueOf(par);
			}else if(type == Boolean.class || type == Boolean.TYPE) {
				cast = Boolean.valueOf(par);
			}else if(type == Byte.class || type == Byte.TYPE) {
				cast = Byte.valueOf(par);
			}else if(type == Short.class || type == Short.TYPE) {
				cast = Short.valueOf(par);
			}else if(type == Long.class || type == Long.TYPE) {
				cast = Long.valueOf(par);
			}else if(type == Character.class || type == Character.TYPE) {
				cast = par.charAt(0);
			}else if(type == String.class) {
				cast = par;
			}else {
				cast = type.cast(par);
			}
			classs.add(new Object[] {cast,type});
		}
		return classs;
	}

	/**
	 * 获取参数类型
	 * 
	 * @param methodParams
	 *            参数相关列表
	 * @return 参数类型列表
	 */
	public static Class<?>[] getMethodParamsType(List<Object[]> methodParams) {
		Class<?>[] classs = new Class<?>[methodParams.size()];
		int index = 0;
		for (Object[] os : methodParams) {
			classs[index] = (Class<?>) os[1];
			index++;
		}
		return classs;
	}

	/**
	 * 获取参数值
	 * 
	 * @param methodParams
	 *            参数相关列表
	 * @return 参数值列表
	 */
	public static Object[] getMethodParamsValue(List<Object[]> methodParams) {
		Object[] classs = new Object[methodParams.size()];
		int index = 0;
		for (Object[] os : methodParams) {
			classs[index] = (Object) os[0];
			index++;
		}
		return classs;
	}

	/**
	 * 从C2_SYS_TASK_ANNOTATION_BEANLIST查找是否有相同的value
	 * 
	 * @param key
	 * @param value
	 * @return
	 */
	public static TaskAnnotationBean getBeanFromC2SysTask(String value) {
		for (TaskAnnotationBean m : Constants.C2_SYS_TASK_ANNOTATION_BEANLIST) {
			if (value.equals(m.getValue())) {
				return m;
			}
		}
		return null;
	}
}

```

## 启动任务

工具任务实体 创建对应任务实体，并设置相关策略

```
public class ScheduleUtils
{
    /**
     * 得到quartz任务类
     *
     * @param sysTask 执行计划
     * @return 具体执行任务类
     */
    private static Class<? extends Job> getQuartzTaskClass(TaskDTO sysTask)
    {
        boolean isConcurrent = "0".equals(sysTask.getConcurrent());
        return isConcurrent ? QuartzTaskExecution.class : QuartzDisallowConcurrentExecution.class;
    }

    /**
     * 构建任务触发对象
     */
    public static TriggerKey getTriggerKey(String jobId, String jobGroup)
    {
        return TriggerKey.triggerKey(ScheduleConstants.TASK_CLASS_NAME + jobId, jobGroup);
    }

    /**
     * 构建任务键对象
     */
    public static JobKey getTaskKey(String jobId, String jobGroup)
    {
        return JobKey.jobKey(ScheduleConstants.TASK_CLASS_NAME + jobId, jobGroup);
    }

    /**
     * 创建定时任务
     */
    public static void createScheduleTask(Scheduler scheduler, TaskDTO job) throws SchedulerException, TaskException
    {
        Class<? extends Job> jobClass = getQuartzTaskClass(job);
        // 构建job信息
        String jobId = job.getTaskId();
        String jobGroup = job.getTaskGroup();
        JobDetail jobDetail = JobBuilder.newJob(jobClass).withIdentity(getTaskKey(jobId, jobGroup)).build();

        // 表达式调度构建器
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule(job.getCronExpression());
        cronScheduleBuilder = handleCronScheduleMisfirePolicy(job, cronScheduleBuilder);

        // 按新的cronExpression表达式构建一个新的trigger
        CronTrigger trigger = TriggerBuilder.newTrigger().withIdentity(getTriggerKey(jobId, jobGroup))
                .withSchedule(cronScheduleBuilder).build();

        // 放入参数，运行时的方法可以获取
        jobDetail.getJobDataMap().put(ScheduleConstants.TASK_PROPERTIES, job);

        // 判断是否存在
        if (scheduler.checkExists(getTaskKey(jobId, jobGroup)))
        {
            // 防止创建时存在数据问题 先移除，然后在执行创建操作
            scheduler.deleteJob(getTaskKey(jobId, jobGroup));
        }

        scheduler.scheduleJob(jobDetail, trigger);

        // 暂停任务
        if (job.getStatus().equals(ScheduleConstants.Status.PAUSE.getValue()))
        {
            scheduler.pauseJob(ScheduleUtils.getTaskKey(jobId, jobGroup));
        }
    }

    /**
     * 设置定时任务策略
     */
    public static CronScheduleBuilder handleCronScheduleMisfirePolicy(TaskDTO job, CronScheduleBuilder cb)
            throws TaskException
    {
        switch (job.getMisfirePolicy())
        {
            case ScheduleConstants.MISFIRE_DEFAULT:
                return cb;
            case ScheduleConstants.MISFIRE_IGNORE_MISFIRES:
                return cb.withMisfireHandlingInstructionIgnoreMisfires();
            case ScheduleConstants.MISFIRE_FIRE_AND_PROCEED:
                return cb.withMisfireHandlingInstructionFireAndProceed();
            case ScheduleConstants.MISFIRE_DO_NOTHING:
                return cb.withMisfireHandlingInstructionDoNothing();
            default:
                throw new TaskException("The task misfire policy '" + job.getMisfirePolicy()
                        + "' cannot be used in cron schedule tasks", Code.CONFIG_ERROR);
        }
    }
}

```

## service启动任务

应用启动时 初始化所有待运行任务

```
@Service
public class TaskServiceImpl implements TaskService {
	@Autowired
	private Scheduler scheduler;

	@Autowired
	private TaskDao taskDao;

	/**
	 * 项目启动时，初始化定时器 主要是防止手动修改数据库导致未同步到定时任务处理（注：不能手动修改数据库ID和任务组名，否则会导致脏数据）
	 */
	@PostConstruct
	public void init() throws SchedulerException, TaskException {
		scheduler.clear();
		//scheduler = StdSchedulerFactory.getDefaultScheduler();
		List<TaskEO> taskList = taskDao.selectTaskAll();
		for (TaskEO task : taskList) {
			TaskDTO taskDTO = new TaskDTO();
			if (null != task) {
				BeanCopierUtil.copy(task, taskDTO);
			}
			if(ScheduleConstants.Status.NORMAL.getValue().equals(taskDTO.getStatus())) {
				ScheduleUtils.createScheduleTask(scheduler, taskDTO);
			}
		}
	}
	
	
	/**
	 * 立即运行任务
	 * 
	 * @param task
	 *            调度信息
	 */
	@Override
	@Transactional
	public void run(TaskDTO task) throws SchedulerException {
		String taskId = task.getTaskId();
		String taskGroup = task.getTaskGroup();
		TaskDTO properties = selectTaskById(task.getTaskId());
		// 参数
		JobDataMap dataMap = new JobDataMap();
		dataMap.put(ScheduleConstants.TASK_PROPERTIES, properties);
		scheduler.triggerJob(ScheduleUtils.getTaskKey(taskId, taskGroup), dataMap);
	}
}
```

