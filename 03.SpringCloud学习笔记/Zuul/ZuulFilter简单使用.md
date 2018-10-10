## ZuulFilter

实现方式: 继承`ZuulFilter`

`run`: 过滤器的具体逻辑

`filterType`: 过滤器类型 可选的值`PRE_TYPE`、`ROUTE_TYPE`、`POST_TYPE`、`ERROR_TYPE`

`filterOrder`: 同一类型的过滤器可以通过`filterOrder`来决定执行顺序，数值越大优先级越低

`shouldFilter `: 判断该过滤器是否需要执行，通过此函数可以实现过滤器开关

过滤器没有直接的方式来访问对方，它们可以使用`RequestContext`共享状态，这是一个类似`Map`结构内部使用的是`ThreadLocal`实现的，通过`RequestContext.getCurrentContext()`获取该对象，常用方法

```java
// 从RequestContext获取上下文
RequestContext requestContext = RequestContext.getCurrentContext();
// 是否继续执行后续的过滤器
requestContext.setSendZuulResponse(true);
// 设置响应状态码
requestContext.setResponseStatusCode(HttpStatus.OK.value())
// 设置请求响应体
requestContext.setResponseBody("{\"username\":\"RobertoHaung\"}");
// 设置一些共享参数
requestContext.set("NEED_FILTER", true);
```

