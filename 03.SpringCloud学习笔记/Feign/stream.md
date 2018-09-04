# Contract

`Contract`的中文意义是合约，`parseAndValidatateMetadata`的作用顾名思义就是将我们传入的接口进行解析验证，看注解的使用是否符合规范，然后返回给我们接口上各种相应的元数据

## BaseContract

其主要逻辑是验证我们使用的时候是否符合了规范，抽取元数据的逻辑留给了子类

```java
public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
      // 检查传入的接口不能带有泛型
      checkState(targetType.getTypeParameters().length == 0, "Parameterized types unsupported: %s", targetType.getSimpleName());
      // 检查传入的接口最多只能有一个父接口
      checkState(targetType.getInterfaces().length <= 1, "Only single inheritance supported: %s", targetType.getSimpleName());
      // 找到传入的接口的父接口，父接口必须没有接口了,既整个继承体系只能有一层
      if (targetType.getInterfaces().length == 1) {
        checkState(targetType.getInterfaces()[0].getInterfaces().length == 0, "Only single-level inheritance supported: %s", targetType.getSimpleName());
      }
      // 新建一个结果集容器
      Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
      // 取得接口里面的所有public方法，包括从父接口继承而来的
      for (Method method : targetType.getMethods()) {
        // 排除掉从Object继承的方法，static方法，接口中的default方法
        if (method.getDeclaringClass() == Object.class || (method.getModifiers() & Modifier.STATIC) != 0 || Util.isDefault(method)) {
          continue;
        }
        // 
        MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
        checkState(!result.containsKey(metadata.configKey()), "Overrides unsupported: %s", metadata.configKey());
        result.put(metadata.configKey(), metadata);
      }
      return new ArrayList<MethodMetadata>(result.values());
    }
}
```

```java
MVC返回Response需要添加MessageConverter

protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
      // 方法元数据封装类
      MethodMetadata data = new MethodMetadata();
      // 解析出方法的返回类型
      data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
      // 用Feign中的静态方法configKey解析出接口&方法作为一种String key，为什么要放在Feign类中
      data.configKey(Feign.configKey(targetType, method));
      // 如果传入的接口有一个父接口，处理父接口上的注解，放入MethodMetadata
      if(targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
      }
      // 处理传入的这个接口上自己的注解
      processAnnotationOnClass(data, targetType);
      // 注意这个方法虚方法，留给了子类去实现。
      
      // 处理这个方法上的注解，同样是虚方法
      for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
      }
      // 检查data里面template是否已放入了这个方法关于HHTP method的信息，比如时GET方法，POST方法之类的。这个操作之前应该是由processAnnotationOnMethod去实现的。
      checkState(data.template().method() != null, "Method %s not annotated with HTTP method type (ex. GET, POST)", method.getName());
      // 得到方法的参数类型
      Class<?>[] parameterTypes = method.getParameterTypes();
      // 得到参数的泛型信息
      Type[] genericParameterTypes = method.getGenericParameterTypes();
      // 得到方法的参数上的在注解
      Annotation[][] parameterAnnotations = method.getParameterAnnotations();
      // 遍历参数的注解
      int count = parameterAnnotations.length;
      for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        //是否是一个“http注解”？
        if (parameterAnnotations[i] != null) {
          isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        // 如果参数是URL类型的，data中放入urlIndex?
        if (parameterTypes[i] == URI.class) {
          data.urlIndex(i);
        } else if (!isHttpAnnotation) {
          // 如果“http注解”为false，检查data中formParams“表单参数”如果不为空，则报错。
          //body参数不能和form参数一起？？？？？
          checkState(data.formParams().isEmpty(), "Body parameters cannot be used with form parameters.");
          // 如果data中bodyIndex不为空，？？？？？？？
          checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
          // 设置bodyIndex
          data.bodyIndex(i);
          // 设置bodyType？？？？
          data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        }
      }
      // 检查headerMap所在的参数必须是一个Map类型，key为String
      if (data.headerMapIndex() != null) {
        checkMapString("HeaderMap", parameterTypes[data.headerMapIndex()], genericParameterTypes[data.headerMapIndex()]);
      }
      // 同理
      if (data.queryMapIndex() != null) {
        checkMapString("QueryMap", parameterTypes[data.queryMapIndex()], genericParameterTypes[data.queryMapIndex()]);
      }

      return data;
    }
}
```

```java
    private static void checkMapString(String name, Class<?> type, Type genericType) {
      checkState(Map.class.isAssignableFrom(type),
              "%s parameter must be a Map: %s", name, type);
      Type[] parameterTypes = ((ParameterizedType) genericType).getActualTypeArguments();
      Class<?> keyClass = (Class<?>) parameterTypes[0];
      checkState(String.class.equals(keyClass),
              "%s key must be a String: %s", name, keyClass.getSimpleName());
    }
```



```java
protected void processAnnotationOnClass(MethodMetadata data, Class<?> targetType) {
  // 检查接口上是否有Headers注解
  if (targetType.isAnnotationPresent(Headers.class)) {
    // 将Headers里的信息转换成Map,“headers”
    String[] headersOnType = targetType.getAnnotation(Headers.class).value();
    checkState(headersOnType.length > 0, "Headers annotation was empty on type %s.", targetType.getName());
    Map<String, Collection<String>> headers = toMap(headersOnType);
    // 将template里面已有的headers也加入进来，合并后放入template里面。
    headers.putAll(data.template().headers());
    data.template().headers(null);
    data.template().headers(headers);
  }
}
```



```java
@Override
protected void processAnnotationOnMethod(MethodMetadata data, Annotation methodAnnotation, Method method) {
  // 提取注解的类型
  Class<? extends Annotation> annotationType = methodAnnotation.annotationType();
  // 如果是RequestLine注解
  if (annotationType == RequestLine.class) {
    // 强转为RequestLine类型后取得，value属性的值
    String requestLine = RequestLine.class.cast(methodAnnotation).value();
    //检查requestLine的值
    checkState(emptyToNull(requestLine) != null, "RequestLine annotation was empty on method %s.", method.getName());
    //如果没找到值里面有空格，比如“GET /mock/resource”
    if (requestLine.indexOf(' ') == -1) {
      // 也没有找到"/"，说明没有正确写值，报错
      checkState(requestLine.indexOf('/') == -1, "RequestLine annotation didn't start with an HTTP verb on method %s.", method.getName());
      // requestline放入template中，返回。
      data.template().method(requestLine);
      return;
    }
    // 如果包含了空格，将空格前的http方法截取出来放入template中
    data.template().method(requestLine.substring(0, requestLine.indexOf(' ')));
    // 根据是否包含了多个空格，去判断是否需要加上http的版本号，
    // 比如“POST /servers HTTP/1.1”
    if (requestLine.indexOf(' ') == requestLine.lastIndexOf(' ')) {
      // no HTTP version is ok
      data.template().append(requestLine.substring(requestLine.indexOf(' ') + 1));
    } else {
      // skip HTTP version
      data.template().append(requestLine.substring(requestLine.indexOf(' ') + 1, requestLine.lastIndexOf(' ')));
    }

  // 解析出decodeSlash,是否解码“/”？？？？？        
  data.template().decodeSlash(RequestLine.class.cast(methodAnnotation).decodeSlash());
} else if (annotationType == Body.class) {//处理另一个可能出现在方法上的注解“Body”
    String body = Body.class.cast(methodAnnotation).value();
    checkState(emptyToNull(body) != null, "Body annotation was empty on method %s.",
               method.getName());
    //body注解的作用是让我们在PUT或者post时自定义请求体，里面的参数用花括号包裹
    if (body.indexOf('{') == -1) {
      //如果没有花括号，说明没有参数替换的必要，直接塞到Template的body里面去
      data.template().body(body);
    } else {
      //有，说明是个模板，先塞到模板属性里面去
      data.template().bodyTemplate(body);
    }
  } else if (annotationType == Headers.class) {
  //方法上也有可能有Headers注解，继续一波类似的骚操作。
  //记得之前在processAnnotationOnClass中的逻辑么
    String[] headersOnMethod = Headers.class.cast(methodAnnotation).value();
    checkState(headersOnMethod.length > 0, "Headers annotation was empty on method %s.", method.getName());
    data.template().headers(toMap(headersOnMethod));
  }
}
```



```java
@Override
protected boolean processAnnotationsOnParameter(MethodMetadata data, Annotation[] annotations, int paramIndex) {
  // isHttpAnnotation
  boolean isHttpAnnotation = false;
  for (Annotation annotation : annotations) {
    Class<? extends Annotation> annotationType = annotation.annotationType();
    if (annotationType == Param.class) {
      // 同样是强转，这次的写法却不同……难道不是一个人写的
      Param paramAnnotation = (Param) annotation;
      String name = paramAnnotation.value();
      checkState(emptyToNull(name) != null, "Param annotation was empty on param %s.", paramIndex);
      // 将注解里的值塞到nameToIndex Map中
      nameParam(data, name, paramIndex);
      Class<? extends Param.Expander> expander = paramAnnotation.expander();
      //如果重写了Expander，同理塞入
      if (expander != Param.ToStringExpander.class) {
        data.indexToExpanderClass().put(paramIndex, expander);
      }
      // 注解声明参数的值是否已经编码过了
      data.indexToEncoded().put(paramIndex, paramAnnotation.encoded());
      // flag设置为true
      isHttpAnnotation = true;
      String varName = '{' + name + '}';
      // 如果url中包含了我们方法上的参数名字 作为占位符，或者queries参数，或者headers头里面，将此值放入formParams
      if (!data.template().url().contains(varName) &&
          !searchMapValuesContainsSubstring(data.template().queries(), varName) &&
          !searchMapValuesContainsSubstring(data.template().headers(), varName)) {
        data.formParams().add(name);
      }
    } else if (annotationType == QueryMap.class) {
      // QueryMap的作用就是为我们提供一个无限量塞URL查询参数的地方，所以也保存他的元数据，同时设置flag为true
      checkState(data.queryMapIndex() == null, "QueryMap annotation was present on multiple parameters.");
      data.queryMapIndex(paramIndex);
      data.queryMapEncoded(QueryMap.class.cast(annotation).encoded());
      isHttpAnnotation = true;
    } else if (annotationType == HeaderMap.class) {
      // 同理设置HeaderMap，flag
      checkState(data.headerMapIndex() == null, "HeaderMap annotation was present on multiple parameters.");
      data.headerMapIndex(paramIndex);
      isHttpAnnotation = true;
    }
  }
  // 所以只有跟Feign相关的三个注解才叫isHttpAnnotation=true。并且处理相关元数据，其他的注解则啥也不干，返回false
  return isHttpAnnotation;
}
```

### 处理类注解

```java
protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
	if (clz.getInterfaces().length == 0) {
		// 检查接口上是否有RequestMapping注解
		RequestMapping classAnnotation = findMergedAnnotation(clz, RequestMapping.class);
		if (classAnnotation != null) {
			if (classAnnotation.value().length > 0) {
				String pathValue = emptyToNull(classAnnotation.value()[0]);
				pathValue = resolve(pathValue);
				if (!pathValue.startsWith("/")) {
					pathValue = "/" + pathValue;
				}
				// 将RequestMapping里的信息解析到RequestTemplate中
				data.template().insert(0, pathValue);
			}
		}
	}
}
```

### 处理方法注解

```java
protected void processAnnotationOnMethod(MethodMetadata data, Annotation methodAnnotation, Method method) {
    // ...
    RequestMapping methodMapping = findMergedAnnotation(method, RequestMapping.class);
    // HTTP Method
    RequestMethod[] methods = methodMapping.method();
    if (methods.length == 0) {
      methods = new RequestMethod[] { RequestMethod.GET };
    }
    checkOne(method, methods, "method");
    data.template().method(methods[0].name());

    // path
    checkAtMostOne(method, methodMapping.value(), "value");
    if (methodMapping.value().length > 0) {
      String pathValue = emptyToNull(methodMapping.value()[0]);
      if (pathValue != null) {
        pathValue = resolve(pathValue);
        // Append path from @RequestMapping if value is present on method
        if (!pathValue.startsWith("/") && !data.template().toString().endsWith("/")) {
          pathValue = "/" + pathValue;
        }
        data.template().append(pathValue);
      }
    }

    // produces
    parseProduces(data, method, methodMapping);
    // consumes
    parseConsumes(data, method, methodMapping);
    // headers
    parseHeaders(data, method, methodMapping);
    data.indexToExpander(new LinkedHashMap<Integer, Param.Expander>());
}
```

