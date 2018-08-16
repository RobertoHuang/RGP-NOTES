1.定时任务配置类

```java
@Configuration
public class ScheduleConfiguration {
    @Autowired
    private List<Trigger> triggerList;

    @Bean
    public SchedulerFactoryBean createSchedulerFactoryBean() {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        schedulerFactoryBean.setJobFactory(new SpringBeanJobFactory());
        Trigger[] triggers = triggerList.stream().toArray(Trigger[]::new);
        schedulerFactoryBean.setTriggers(triggers);
        return schedulerFactoryBean;
    }
}
```

2.定时任务实体类
```java
@Slf4j
@Component
public class HeartBeatJob {
    public void execute() {
       log.info("正在发送心跳检测");
    }
}
```

3.定义`MethodInvokingJobDetailFactoryBean`
```java
@Configuration
public class JobDetailConfiguration {
    @Autowired
    private HeartBeatJob heartBeatJob;

    @Bean(name = "heartBeatJobDetail")
    public MethodInvokingJobDetailFactoryBean heartBeatJobDetail() {
        MethodInvokingJobDetailFactoryBean methodInvokingJobDetailFactoryBean = new MethodInvokingJobDetailFactoryBean();
        methodInvokingJobDetailFactoryBean.setConcurrent(false);
        methodInvokingJobDetailFactoryBean.setName("HEART-BEAT");
        methodInvokingJobDetailFactoryBean.setGroup("AI-BARRIER-CLIENT");
        methodInvokingJobDetailFactoryBean.setTargetObject(heartBeatJob);
        methodInvokingJobDetailFactoryBean.setTargetMethod("execute");
        return methodInvokingJobDetailFactoryBean;
    }
}

注意:setConcurrent:是否并发执行 

例如每5s执行一次任务，但是当前任务还没有执行完，就已经过了5s了 
如果此处为true则下一个任务会执行 如果此处为false则下一个任务会等待上一个任务执行完后再开始执行
```
4.定义`CronTriggerFactoryBean`触发器

```java
@Configuration
public class TriggerConfiguration{
    @Bean(name = "heartBeatTrigger")
    public CronTriggerFactoryBean barrierHeartBeatTrigger(@Qualifier(value = "heartBeatTrigger") MethodInvokingJobDetailFactoryBean barrierHeartBeatJobDetail) {
        CronTriggerFactoryBean cronTriggerFactoryBean = new CronTriggerFactoryBean();
        cronTriggerFactoryBean.setJobDetail(barrierHeartBeatJobDetail.getObject());
        // 每五分钟执行一次
        cronTriggerFactoryBean.setCronExpression("0 0/1 * * * ?");
        cronTriggerFactoryBean.setName("HEART-BEAT-TRIGGER");
        return cronTriggerFactoryBean;
    }
}
```

