# `Spring Retry`

`Spring Retry`主要实现了重试和垄断，为了实现该机制`Spring`提供了几个抽象

- `RetryTemplate` 重试模板
- `RetryPolicy` 重试策略(重试方式)
- `BackOffPolicy` 两次重试之间的回避策略

- `RetryCallback` 重试回调，即具体的重试动作
- `RecoveryCallback` 当所有重试都失败后，回调该接口
- `RetryListener` 监听重试行为，主要用于监控统计等操作
- `RetryContext` 重试语境下的上下文，可用于在多次`Retry`或者`Retry`和`Recover`之间传递参数或状态

看到这里应该还是一脸懵逼这啥玩意啊，接下来通过一个`DEMO`就能懂得这些抽象的作用了

```java
public class SpringRetryTest {
    public static void main(String[] args) throws Exception {
        testSpringRetry();
    }

    public static void testSpringRetry() throws Exception {
        // 构建重试模板
        RetryTemplate retryTemplate = new RetryTemplate();

        // 设置重试策略(此处为按次数重试)
        SimpleRetryPolicy policy = new SimpleRetryPolicy(3, Collections.singletonMap(Exception.class, true));
        retryTemplate.setRetryPolicy(policy);

        // 设置重试回退操作策略(n毫秒退避后再进行重试)
        FixedBackOffPolicy fixedBackOffPolicy = new FixedBackOffPolicy();
        fixedBackOffPolicy.setBackOffPeriod(3000);
        retryTemplate.setBackOffPolicy(fixedBackOffPolicy);

        // 通过RetryCallback 重试回调实例包装正常逻辑逻辑，第一次执行和重试执行执行的都是这段逻辑
        //RetryContext 重试操作上下文约定，统一spring-try包装
        final RetryCallback<Object, Exception> retryCallback = context -> {
            System.out.println("重试业务逻辑，当前重试次数为:" + context.getRetryCount() + "\r\n");
            context.setAttribute(RetryContext.NAME,"method.key");
            throw new RuntimeException("抛出异常进行下一次重试");
        };

        // 通过RecoveryCallback 重试流程正常结束或者达到重试上限后的退出恢复操作实例
        final RecoveryCallback<Object> recoveryCallback = context -> {
            System.out.println("重试都失败用来兜底的，返回值将做为execute方法的返回值");
            return "兜底返回值";
        };

        // 由retryTemplate 执行execute方法开始逻辑执行
        RetryListener retryListener = new RetryListener() {
            @Override
            public <T, E extends Throwable> boolean open(RetryContext retryContext, RetryCallback<T, E> retryCallback) {
                System.out.println("重试逻辑开始执行啦");
                return true;
            }

            @Override
            public <T, E extends Throwable> void close(RetryContext retryContext, RetryCallback<T, E> retryCallback, Throwable throwable) {
                System.out.println("重试逻辑被关闭啦");
            }

            @Override
            public <T, E extends Throwable> void onError(RetryContext retryContext, RetryCallback<T, E> retryCallback, Throwable throwable) {
                System.out.println("重试逻辑出现异常啦");
            }
        };

        DefaultStatisticsRepository defaultStatisticsRepository = new DefaultStatisticsRepository();
        StatisticsListener statisticsListener = new StatisticsListener(defaultStatisticsRepository);
        retryTemplate.setListeners(new RetryListener[]{retryListener, statisticsListener});
        Object execute = retryTemplate.execute(retryCallback, recoveryCallback);
        System.out.println(execute);

        RetryStatistics statistics = defaultStatisticsRepository.findOne("method.key");
        System.out.println(statistics);
    }
}
```

程序运行结果

```java
重试逻辑开始执行啦
16:55:57.943 [main] DEBUG org.springframework.retry.support.RetryTemplate - Retry: count=0
重试业务逻辑，当前重试次数为:0

重试逻辑出现异常啦
16:56:00.979 [main] DEBUG org.springframework.retry.support.RetryTemplate - Checking for rethrow: count=1
16:56:00.979 [main] DEBUG org.springframework.retry.support.RetryTemplate - Retry: count=1
重试业务逻辑，当前重试次数为:1

重试逻辑出现异常啦
16:56:03.980 [main] DEBUG org.springframework.retry.support.RetryTemplate - Checking for rethrow: count=2
16:56:03.980 [main] DEBUG org.springframework.retry.support.RetryTemplate - Retry: count=2
重试业务逻辑，当前重试次数为:2

重试逻辑出现异常啦
16:56:03.980 [main] DEBUG org.springframework.retry.support.RetryTemplate - Checking for rethrow: count=3
16:56:03.980 [main] DEBUG org.springframework.retry.support.RetryTemplate - Retry failed last attempt: count=3
重试都失败用来兜底的，返回值将做为execute方法的返回值
重试逻辑被关闭啦
兜底返回值
DefaultRetryStatistics [name=method.key, startedCount=3, completeCount=0, recoveryCount=1, errorCount=3, abortCount=0]
```

该程序运行流程比较简单，输出的内容都很清晰。在程序中添加了`StatisticsListener`它用来统计一些信息

## 重试策略

- `NeverRetryPolicy` 不重试
- `AlwaysRetryPolicy` 无限重试
- `SimpleRetryPolicy` 重试N次 默认为3次
- `TimeoutRetryPolicy` 在N毫秒内不断重试
- `CircuitBreakerRetryPolicy` 垄断功能的重试
- `ExceptionClassifierRetryPolicy` 根据不同异常执行不同重试策略
- `CompositeRetryPolicy` 将不同的策略组合起来，有悲观组合和乐观组合

## 回避策略

- `NoBackOffPolicy` 不回避
- `FixedBackOffPolicy` N毫秒退避后重试
- `UniformRandomBackOffPolicy` 随机选择一个`[n, m]`退避后重试
- `ExponentialBackOffPolicy` 指数退避策略，休眠时间指数递增
- `ExponentialRandomBackOffPolicy` 随机指数退避，指数中乘积会混入随机值

## 注解方式实现

- 启动类添加`@EnableRetry`注解 开启`Spring`重试

- 使用`@Retryable` `@Backoff` `@Recover`进行重试配置

  ```java
  @Retryable(value = Exception.class, maxAttempts = 5, backoff = @Backoff(delay = 5000L, multiplier = 2))
  public void execute() {
      
  }
  
  @Recover
  public void revover(Exception e) {
      
  }
  ```

  - `Retryable` 

    - `value` 指定发生的异常进行重试 可以配置多个
    - `include` 指定发生的异常进行重试 可以配置多个
    - `exclude` 指定发生的异常进行不重试 可以配置过个
    - `backoff` 重试补偿机制 默认没有
    - `maxAttempts` 最大重试次数

  - `Backoff`
    - 不设置参数时默认使用`FixedBackOffPolicy`默认重试等待`1000ms`
    - 设置`delay`、`maxDealy`时使用`FixedBackOffPolicy`(指定等待时间范围)
    - 设置`multiplier`时使用`ExponentialBackOffPolicy`(指数级重试间隔)，`multiplier`即指定延迟倍数，比如`delay=5000l，multiplier=2`则第一次重试为5秒，第二次为10秒，第三次为20秒

  - `Recover`

    - 当重试到达指定次数时被注解的方法将被回调  需要注意:发生的异常和入参类型一致时才会回调