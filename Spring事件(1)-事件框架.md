## Spring内置事件和自定义事件



### 事件传递对象（ApplicationEvent）

![ApplicationEnvironmentPreparedEvent](Spring事件(1)-事件框架.assets\ApplicationEnvironmentPreparedEvent.png)

`ApplicationEvent`是事件接口，继承自`EventObject`(Java规范要求事件对象都需要继承该类)。

内置事件对象包括两部分：

 	1. `ApplicationContextEvent`是Spring Framework需要的。
 	2. `SpringApplicationEvent`该类在Spring boot包下。

### 事件监听（ApplicationListener）

![MultiServerUserRegistry](Spring事件(1)-事件框架.assets\MultiServerUserRegistry.png)

`ApplicationListener`是Spring的事件监听接口，实现了jdk中的`EventListener`，该接口只提供了一个抽象方法`void onApplicationEvent(E event)`用于处理接收到的事件。

子接口`SmartApplicationListener`增加了两个抽象方法（`supportsEventType`，`supportsSourceType`）并且还继承了`Ordered`接口提供监听器排序。

`supportsEventType`和`supportsSourceType`两个方法都是过滤方法返回事件类型和事件源类型是否支持。

### 事件发布

#### ApplicationEventPublisher

![AbstractApplicationContext](Spring事件(1)-事件框架.assets\AbstractApplicationContext.png)

`ApplicationEventPublisher`事件发布接口，`ApplicationContext`接口继承了该接口，这也就说明Spring Framework的核心`IoC`容器都具有发布事件的能力，接口方法在`AbstractApplicationContext`中实现，来看一下实现代码：

```java
/**
* Publish the given event to all listeners.
* @param event the event to publish (may be an {@link ApplicationEvent}
* or a payload object to be turned into a {@link PayloadApplicationEvent})
* @param eventType the resolved event type, if known
* @since 4.2
*/
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");
    if (logger.isTraceEnabled()) {
        logger.trace("Publishing event in " + getDisplayName() + ": " + event);
    }

    // Decorate event as an ApplicationEvent if necessary
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent) event;
    }
    else {
        applicationEvent = new PayloadApplicationEvent<>(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
        }
    }

    // Multicast right now if possible - or lazily once the multicaster is initialized
    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
    }
    else {
        //事件发布委派给ApplicationEventMulticaster类来完成
        getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
    }

    // Publish event via parent context as well...
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
        }
        else {
            this.parent.publishEvent(event);
        }
    }
}
```

从代码中可以看出事件发布实际上是委派给`ApplicationEventMulticaster`实例来做的，下面我们来看`ApplicationEventMulticaster`。

#### ApplicationEventMulticaster

![SimpleApplicationEventMulticaster](Spring事件(1)-事件框架.assets\SimpleApplicationEventMulticaster.png)

`ApplicationEventMulticaster`接口有一个抽象实现和一个简单实现，通过查看`ApplicationEventMulticaster`接口规范我们会发现改接口提供注册`Listener`和通知`Listener`的接口契约。

Spring内部的事件发布是通过`ApplicationEvent#publishEvent`再委派给`ApplicationEventMulticaster`实际去发送事件的。我们来看一下`ApplicationEventMulticaster`的实现逻辑。

1. `ApplicationEventMulticaster`的实例化

   ```java
   /**
    * 初始化ApplicationEventMulticaster.
        * 如果在IoC容器上下文中不存在以applicationEventMulticaster命名的ApplicationEventMulticaster Bean
    * 就使用新创建一个SimpleApplicationEventMulticaster，并注册到IoC容器中。
    *
    * Initialize the ApplicationEventMulticaster.
    * Uses SimpleApplicationEventMulticaster if none defined in the context.
    * @see org.springframework.context.event.SimpleApplicationEventMulticaster
    */
   protected void initApplicationEventMulticaster() {
       ConfigurableListableBeanFactory beanFactory = getBeanFactory();
       if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
           this.applicationEventMulticaster =
               beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
           if (logger.isDebugEnabled()) {
               logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
           }
       } else {
           this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
           beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
           if (logger.isDebugEnabled()) {
               logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
                            APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
                            "': using default [" + this.applicationEventMulticaster + "]");
           }
       }
   }
   ```

2. `SimpleApplicationEventMulticaster#multicastEvent()`的实现

   ```java
   @Override
   public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
       ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
       for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
           Executor executor = getTaskExecutor();
           //如果Executor如果不为空，那么将调用Executor的execute方法，发起异步通知
           if (executor != null) {
               executor.execute(() -> invokeListener(listener, event));
           }
           //否则同步通知
           else {
               invokeListener(listener, event);
           }
       }
   }
   ```

   从代码中可以看出，只要Executor不为空，通知将异步执行，否则同步执行通知，所以我们可以传递Executor对象让通知异步执行。

   * SpringBoot方式

     ```java
     @Configuration
     @EnableAsync
     public class MulticasterAsyncConfig {
     
         @Bean
         public ThreadPoolTaskExecutor threadPoolTaskExecutor() {
             ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
             executor.setCorePoolSize(10);
             executor.setMaxPoolSize(100);
             executor.setQueueCapacity(2000);
             return executor;
         }
     
         @Bean
         public ApplicationEventMulticaster applicationEventMulticaster(BeanFactory beanFactory) {
             SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster(beanFactory);
             multicaster.setTaskExecutor(threadPoolTaskExecutor());
             return multicaster;
         }
     }
     ```

     

   * Spring XML方式

     ```xml
     <task:executor id="executor" pool-size="10" />  
     <!-- 名字必须是applicationEventMulticaster和messageSource是一样的，默认找这个名字的对象 -->  
     <!-- 名字必须是applicationEventMulticaster，因为AbstractApplicationContext默认找个 -->  
     <!-- 如果找不到就new一个，但不是异步调用而是同步调用 -->  
     <bean id="applicationEventMulticaster" class="org.springframework.context.event.SimpleApplicationEventMulticaster">  
         <!-- 注入任务执行器 这样就实现了异步调用（缺点是全局的，要么全部异步，要么全部同步（删除这个属性即是同步））  -->  
         <property name="taskExecutor" ref="executor"/>  
     </bean>
     ```

   但是这种方式有个弊端就是通知要么是全部是同步要么全部是异步，不能根据需要决定是否异步，所以我们使用另一种方式实现通知消息异步。我们会在下一篇**自定义事件**中讲解

### 自定义事件

有了Spring的支持我们自定义事件是非常简单的。

1. 自定义事件

   ```java
   public class CustomEvent<T> extends ApplicationEvent {
   
       public CustomEvent(T source) {
           super(source);
       }
   }
   ```

   

2. 自定义事件监听继承自`ApplicationListener`

   ```java
   @Component
   public class CustomEventListener implements ApplicationListener<CustomEvent> {
   
       @Override
       public void onApplicationEvent(CustomEvent customEvent) {
           System.out.println("得到事件内容：" + customEvent.getSource());
       }
   }
   ```

   

3. 自定义事件监听实现`SmartApplicationListener`

   * `CustomSmartEventListener1` order=1

   ```java
   @Component
   public class CustomSmartEventListener1 implements SmartApplicationListener {
       @Override
       public boolean supportsEventType(Class<? extends ApplicationEvent> aClass) {
           return StringUtils.equals(aClass.getName(),CustomEvent.class.getName());
       }
   
       @Override
       public boolean supportsSourceType(Class<?> aClass) {
           return StringUtils.equals(String.class.getName(),aClass.getName());
       }
   
       @Override
       public void onApplicationEvent(ApplicationEvent applicationEvent) {
           System.out.println("接收到通知1：" + applicationEvent.getSource());
       }
   
       @Override
       public int getOrder() {
           return 1;
       }
   }
   ```

   * `CustomSmartEventListener2` order=2

   ```java
   @Component
   public class CustomSmartEventListener2 implements SmartApplicationListener {
       @Override
       public boolean supportsEventType(Class<? extends ApplicationEvent> aClass) {
           return StringUtils.equals(aClass.getName(),CustomEvent.class.getName());
       }
   
       @Override
       public boolean supportsSourceType(Class<?> aClass) {
           return StringUtils.equals(String.class.getName(),aClass.getName());
       }
   
       @Override
       public void onApplicationEvent(ApplicationEvent applicationEvent) {
           System.out.println("接收到通知2：" + applicationEvent.getSource());
       }
   
       @Override
       public int getOrder() {
           return 2;
       }
   }
   ```

   

4. 事件发布

   ```java
   @Autowired
   private ApplicationContext applicationContext;
   
   @GetMapping("/")
   public String index(){
       applicationContext.publishEvent(new CustomEvent<>("收到消息了吗？"));
       return "/index";
   }
   ```

5. 输出结果：

   ```
   得到事件内容：收到消息了吗？
   接收到通知1：收到消息了吗？
   接收到通知2：收到消息了吗？
   ```

   从输出接口可以看到order越小监听器越早被调用。

## 参考

https://jinnianshilongnian.iteye.com/blog/1902886