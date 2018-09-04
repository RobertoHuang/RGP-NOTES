- `FeignClientsConfiguration`  

  ```java
  @Configuration
  @ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
  protected static class HystrixFeignConfiguration {
  	@Bean
  	@Scope("prototype")
  	@ConditionalOnMissingBean
  	@ConditionalOnProperty(name = "feign.hystrix.enabled", matchIfMissing = false)
  	public Feign.Builder feignHystrixBuilder() {
  		return HystrixFeign.builder();
  	}
  }
  ```

  如果不自定义`Feign.Builder`会优先配置`HystrixFeign.Builder `，若要禁用`Hystrix`有如下两种方式

  - 配置增加`feign.hystrix.enabled=false`

  - `FeignConfiguration`增加如下配置

    ```java
    @Bean
    @Scope("prototype")
    public Feign.Builder feignBuilder() {
        return Feign.builder();
    }
    ```

- 使用 Hystrix 解决内部调用抛出异常问题

