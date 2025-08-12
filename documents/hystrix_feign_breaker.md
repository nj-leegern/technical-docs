# 自定义基于Hystrix的Feign全局断路器扩展



## 背景

在使用Feign的时候，每一个Feign接口都需要定义fallback或者fallbackFactory来处理熔断，遇到庞大的系统时工作量不小，所以需要定义全局的熔断器以便减轻业务开发负担， 我的思路就是使用动态代理（cglib）来代理各个Feign处理熔断。



## 流程分析

首先需要了解Feign集成Hystrix以及自动配置过程：

1. FeignAutoConfiguration自动配置类

   ```java
       @Configuration
       @ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
       protected static class HystrixFeignTargeterConfiguration {
   
           @Bean
           @ConditionalOnMissingBean
           public Targeter feignTargeter() {
               return new HystrixTargeter();
           }
   
       }
   
       @Configuration
       @ConditionalOnMissingClass("feign.hystrix.HystrixFeign")
       protected static class DefaultFeignTargeterConfiguration {
   
           @Bean
           @ConditionalOnMissingBean
           public Targeter feignTargeter() {
               return new DefaultTargeter();
           }
   
       }
   ```

   如果存在'feign.hystrix.HystrixFeign'类，就使用HystrixTargeter，否则使用DefaultTargeter

   

2. HystrixTargeter类

   ```java
       @Override
       public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
               FeignContext context, Target.HardCodedTarget<T> target) {
           if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
               return feign.target(target);
           }
           feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
           SetterFactory setterFactory = getOptional(factory.getName(), context,
                   SetterFactory.class);
           if (setterFactory != null) {
               builder.setterFactory(setterFactory);
           }
           Class<?> fallback = factory.getFallback();
           if (fallback != void.class) {
               return targetWithFallback(factory.getName(), context, target, builder,
                       fallback);
           }
           Class<?> fallbackFactory = factory.getFallbackFactory();
           if (fallbackFactory != void.class) {
               return targetWithFallbackFactory(factory.getName(), context, target, builder,
                       fallbackFactory);
           }
   
           return feign.target(target);
       }
   ```

   1>. 通过以上代码可以看出，优先使用Feign主动配置的fallback和fallbackFactory

   2>. 如果没配置的话，调用HystrixFeign.Builder的target(Target<T> target)方法，即不降级处理。

   

3. HystrixFeign.Builder类

   ```java
    public static final class Builder extends Feign.Builder {
   
       private Contract contract = new Contract.Default();
       private SetterFactory setterFactory = new SetterFactory.Default();
   
       /**
        * Allows you to override hystrix properties such as thread pools and command keys.
        */
       public Builder setterFactory(SetterFactory setterFactory) {
         this.setterFactory = setterFactory;
         return this;
       }
   
       /**
        * @see #target(Class, String, Object)
        */
       public <T> T target(Target<T> target, T fallback) {
         return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null)
             .newInstance(target);
       }
   
       /**
        * @see #target(Class, String, FallbackFactory)
        */
       public <T> T target(Target<T> target, FallbackFactory<? extends T> fallbackFactory) {
         return build(fallbackFactory).newInstance(target);
       }
   
       // ...........
   
       @Override
       public Feign build() {
         return build(null);
       }
   
       /** Configures components needed for hystrix integration. */
       Feign build(final FallbackFactory<?> nullableFallbackFactory) {
         super.invocationHandlerFactory(new InvocationHandlerFactory() {
           @Override
           public InvocationHandler create(Target target,
                                           Map<Method, MethodHandler> dispatch) {
             return new HystrixInvocationHandler(target, dispatch, setterFactory,
                 nullableFallbackFactory);
           }
         });
         super.contract(new HystrixDelegatingContract(contract));
         return super.build();
       }
   
   ```

   1>. HystrixFeign.Builder继承了Feign.Builder类，并且target(Target<T> target)方法继承自父类Feign.Builder

   2>. 父类Feign.Builder中target(Target<T> target)调用的是build().newInstance(target)方法，子类HystrixFeign.Builder重写了

   ​       build()方法。

   3>. 最终创建Feign实例的是子类的build(final FallbackFactory<?> nullableFallbackFactory)方法。

   4>. 通过build(final FallbackFactory<?> nullableFallbackFactory)，最后实例化HystrixInvocationHandler处理熔断。



## 实现方案 

   在了解完具体过程之后，我们只需要将自定义的全局熔断器FallbackFactory传递给build(final FallbackFactory<?> fallbackFactory)就

   可以了，顺着这个思路想到了如下方案：

1. 首先想到的是继承HystrixFeign.Builder重写build()，但是HystrixFeign.Builder类使用final关键字修饰，此方案行不通。

2. 通过参考Spring Cloud Alibaba中的熔断器Sentinel与Feign集成方式（com.alibaba.cloud.sentinel.feign.SentinelFeign），自定

   ​      义Feign.Builder子类重写target(Target<T> target)方法，默认使用自定义的FallbackFactory来处理熔断。

   

## 具体实现

1. 自定义Feign熔断代理Fallback

   ```java
   @Slf4j
   public class XcloudFeignFallback<T> implements org.springframework.cglib.proxy.MethodInterceptor {
   
       private Class<T> targetType;
       private String targetName;
       private Throwable cause;
   
       public XcloudFeignFallback(Class<T> targetType, String targetName, Throwable cause) {
           this.targetType = targetType;
           this.targetName = targetName;
           this.cause = cause;
       }
   
       @Nullable
       @Override
       public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws 	
           																					  Throwable {
           FeignException exception = (FeignException) cause;
   
           log.error("XcloudFeignFallback:[{}.{}] serviceId:[{}] message:[{}]", targetType.getName(), 	
                                                     method.getName(), targetName, exception.contentUTF8());
           //  告警代码实现 ......
   
           return null;
       }
   
       @Override
       public boolean equals(Object o) {
           if (this == o) {
               return true;
           }
           if (o == null || getClass() != o.getClass()) {
               return false;
           }
           XcloudFeignFallback<?> that = (XcloudFeignFallback<?>) o;
           return targetType.equals(that.targetType);
       }
   
       @Override
       public int hashCode() {
           return Objects.hash(targetType);
       }
   }
   ```

   

2. 自定义全局熔断FallbackFactory实现类

   ```java
   @AllArgsConstructor
   public class GlobalFallbackFactory<T> implements FallbackFactory<T> {
   
       /* 被代理目标对象 */
       private Target<T> target;
   
       @Override
       public T create(Throwable cause) {
           // 获取@FeignClient注解的接口类型
           Class<T> targetType = target.type();
           String targetName = target.name();
           Enhancer enhancer = new Enhancer();
           enhancer.setSuperclass(targetType);
           enhancer.setUseCache(true);
           enhancer.setCallback(new XcloudFeignFallback<>(targetType, targetName, cause));
           return (T) enhancer.create();
       }
   }
   ```

   

3. 自定义Feign.Builder，重写target(Target<T> target)，处理全局熔断

   ```java
   public final class XcloudFeign {
   
       private XcloudFeign() {}
   
       public static Builder builder() {
           return new Builder();
       }
   
       public static final class Builder extends Feign.Builder implements ApplicationContextAware {
   
           private Contract contract = new Contract.Default();
   
           private SetterFactory setterFactory = new SetterFactory.Default();
   
           private ApplicationContext applicationContext;
   
           private FeignContext feignContext;
   
           @Override
           public Feign.Builder invocationHandlerFactory(InvocationHandlerFactory invocationHandlerFactory) 
           {
               throw new UnsupportedOperationException();
           }
   
           @Override
           public Builder contract(Contract contract) {
               this.contract = contract;
               return this;
           }
   
           public Builder setterFactory(SetterFactory setterFactory) {
               this.setterFactory = setterFactory;
               return this;
           }
   
           @Override
           public <T> T target(Target<T> target) {
               // 自定义熔断器
               return (T) this.target(target, new GlobalFallbackFactory(target));
           }
   
           /**
            * @see #target(Class, String, Object)
            */
           public <T> T target(Target<T> target, T fallback) {
               return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : 
            																	   null).newInstance(target);
           }
   
           /**
            * @see #target(Class, String, FallbackFactory)
            */
           public <T> T target(Target<T> target, FallbackFactory<? extends T> fallbackFactory) {
               return build(fallbackFactory).newInstance(target);
           }
   
           /**
            * Like {@link Feign#newInstance(Target)}
            * fallback support.
            * @see #target(Target, Object)
            */
           public <T> T target(Class<T> apiType, String url, T fallback) {
               return target(new Target.HardCodedTarget<T>(apiType, url), fallback);
           }
   
           /**
            * Same as {@link #target(Class, String, T)}, except you can inspect a source exception before
            * creating a fallback object.
            */
           public <T> T target(Class<T> apiType, String url,
                               FallbackFactory<? extends T> fallbackFactory) {
               return target(new Target.HardCodedTarget<T>(apiType, url), fallbackFactory);
           }
   
           @Override
           public Feign build() {
               return this.build(null);
           }
   
           /** Configures components needed for hystrix integration. */
           private Feign build(final FallbackFactory<?> defaultFallbackFactory) {
               super.invocationHandlerFactory(new InvocationHandlerFactory() {
                   @Override
                   public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
                       // using reflect get fallback and fallbackFactory properties from
                       // FeignClientFactoryBean because FeignClientFactoryBean is a package
                       // level class, we can not use it in our package
                       Object feignClientFactoryBean = Builder.this.applicationContext.getBean("&" +                                                                                   target.type().getName());
   
                       Class fallback = (Class) getFieldValue(feignClientFactoryBean, "fallback");
                       Class fallbackFactory = (Class) getFieldValue(feignClientFactoryBean, 	
                                                                     "fallbackFactory");
                       String beanName = (String) getFieldValue(feignClientFactoryBean, "contextId");
                       if (! StringUtils.hasText(beanName)) {
                           beanName = (String) getFieldValue(feignClientFactoryBean, "name");
                       }
   
                       // check fallback
                       if (void.class != fallback) {
                           Object fallbackInstance = getFromContext(beanName, "fallback", fallback, 
                                                                    target.type());
                           return new HystrixInvocationHandler(target, dispatch, setterFactory, new 
                                                               FallbackFactory.Default(fallbackInstance));
                       }
   
                       // check fallbackFactory
                       if (void.class != fallbackFactory) {
                           FallbackFactory fallbackFactoryInstance = (FallbackFactory) 
                               getFromContext(beanName, "fallbackFactory", fallbackFactory, 
                                              FallbackFactory.class);
                           return new HystrixInvocationHandler(target, dispatch, setterFactory, 
                                                               fallbackFactoryInstance);
                       }
   
                       // default fallbackFactory
                       return new HystrixInvocationHandler(target, dispatch, setterFactory, 
                                                           defaultFallbackFactory);
                   }
   
                   /* Get fallback or fallbackFactory from Context */
                   private Object getFromContext(String name, String type, Class fallbackType, Class 
                                                 targetType) {
                       Object fallbackInstance = feignContext.getInstance(name, fallbackType);
                       if (fallbackInstance == null) {
                           throw new IllegalStateException(String.format("No %s instance of type %s found 
                                                                         for feign client %s", type, 
                                                                         fallbackType, name));
                       }
   
                       if (! targetType.isAssignableFrom(fallbackType)) {
                           throw new IllegalStateException(String.format(
                                   "Incompatible %s instance. Fallback/fallbackFactory of type %s is not 
                                    assignable to %s for feign client %s", type, fallbackType, targetType, 
                           name));
                       }
                       return fallbackInstance;
                   }
               });
               super.contract(new HystrixDelegatingContract(contract));
               return super.build();
           }
   
           private Object getFieldValue(Object instance, String fieldName) {
               Field field = ReflectionUtils.findField(instance.getClass(), fieldName);
               field.setAccessible(true);
               try {
                   return field.get(instance);
               }
               catch (IllegalAccessException e) {
                   // ignore
               }
               return null;
           }
   
           @Override
           public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
               this.applicationContext = applicationContext;
               feignContext = this.applicationContext.getBean(FeignContext.class);
           }
       }
   }
   
   ```

   

4. 自定义Feign配置类，初始化自定义的Feign.Builder

   ```java
   @Configuration
   public class DefaultFeignConfiguration {
   
       @Bean
       @Primary
       @Scope("prototype")
       @ConditionalOnProperty(name = "feign.hystrix.enabled", havingValue = "true")
       public Feign.Builder xcloudFeignHystrixBuilder() {
           return XcloudFeign.builder();
       }
   }
   ```

   