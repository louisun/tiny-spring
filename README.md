# tiny-spring 源码分析

> tiny-spring 源码分析 by Louis

## 使用

`tiny-spring`是逐步进行构建的，里程碑版本我都使用了 git tag 来管理。例如，最开始的tag是`step-1-container-register-and-get`，那么可以使用下列命令获取各个版本

```
git checkout step-1-container-register-and-get
```

各版本见[`changelog.md`](https://github.com/louisun/tiny-spring/blob/master/changelog.md)



## 概览

`tiny-spring` 是一个用于学习 Spring 的开源项目， 可以说是精简版的 Spring，其核心功能是：

1. 支持 singleton 类型的 bean（可从 xml 读取），支持初始化、属性注入（包括依赖 bean 注入）。
2. 可以使用 `Aspectj` 的方式进行 AOP 编写，支持Java Proxy接口代理和 CGlib 的类代理。

##  

**代码目录树：**


```bash
.
└── us
    └── codecraft
        └── tinyioc
            ├── BeanReference.java
            ├── aop
            │   ├── AbstractAopProxy.java
            │   ├── AdvisedSupport.java
            │   ├── Advisor.java
            │   ├── AopProxy.java
            │   ├── AspectJAroundAdvice.java
            │   ├── AspectJAwareAdvisorAutoProxyCreator.java
            │   ├── AspectJExpressionPointcut.java
            │   ├── AspectJExpressionPointcutAdvisor.java
            │   ├── BeanFactoryAware.java
            │   ├── Cglib2AopProxy.java
            │   ├── ClassFilter.java
            │   ├── JdkDynamicAopProxy.java
            │   ├── MethodMatcher.java
            │   ├── Pointcut.java
            │   ├── PointcutAdvisor.java
            │   ├── ProxyFactory.java
            │   ├── ReflectiveMethodInvocation.java
            │   └── TargetSource.java
            ├── beans
            │   ├── AbstractBeanDefinitionReader.java
            │   ├── BeanDefinition.java
            │   ├── BeanDefinitionReader.java
            │   ├── BeanPostProcessor.java
            │   ├── PropertyValue.java
            │   ├── PropertyValues.java
            │   ├── factory
            │   │   ├── AbstractBeanFactory.java
            │   │   ├── AutowireCapableBeanFactory.java
            │   │   └── BeanFactory.java
            │   ├── io
            │   │   ├── Resource.java
            │   │   ├── ResourceLoader.java
            │   │   └── UrlResource.java
            │   └── xml
            │       └── XmlBeanDefinitionReader.java
            └── context
                ├── AbstractApplicationContext.java
                ├── ApplicationContext.java
                └── ClassPathXmlApplicationContext.java
```

## 1. 文件资源

`Resource` 接口标识一个“外部资源”，通过`getInputStream()` 获得输入流。

`UrlResource` 实现 `Resource` 接口，通过 URL 来 `getInputStream()`。

`ResourceLoader` 是一个资源加载类，用 `getResource(String location)` 返回一个 `UrlResource`。

```java
public Resource getResource(String location){
    // 资源加载
    URL resource = this.getClass().getClassLoader().getResource(location);
    return new UrlResource(resource);
}
```

## 2. Bean 的定义、加载


### 2.1 BeanDefinition 对象

`BeanDefinition` 保存了所谓的`bean`的元数据，`bean`实际上就是任意的 `Object`。

元数据有 `bean` 对象本身（也就是 Object） ，bean 的类 `beanClass`，bean 的类名 `beanClassName`（用于反射），以及一个专门的 **“属性对”** 列表 `propertyValues`，里面放的是 `PropertyValue`：其实就是一个属性名和数值型。

```java
public class BeanDefinition {

	private Object bean;

	private Class beanClass;

	private String beanClassName;

	private PropertyValues propertyValues = new PropertyValues();

	public BeanDefinition() {
	}

	public void setBean(Object bean) {
		this.bean = bean;
	}

	public Class getBeanClass() {
		return beanClass;
	}

	public void setBeanClass(Class beanClass) {
		this.beanClass = beanClass;
	}

	public String getBeanClassName() {
		return beanClassName;
	}

	public void setBeanClassName(String beanClassName) {
		this.beanClassName = beanClassName;
		try {
			this.beanClass = Class.forName(beanClassName);
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}

	public Object getBean() {
		return bean;
	}

	public PropertyValues getPropertyValues() {
		return propertyValues;
	}

	public void setPropertyValues(PropertyValues propertyValues) {
		this.propertyValues = propertyValues;
	}
}

public class PropertyValue {

    private final String name;

    private final Object value;

    public PropertyValue(String name, Object value) {
        this.name = name;
        this.value = value;
    }

    public String getName() {
        return name;
    }

    public Object getValue() {
        return value;
    }
}

public class PropertyValues {

	private final List<PropertyValue> propertyValueList = new ArrayList<PropertyValue>();

	public PropertyValues() {
	}

	public void addPropertyValue(PropertyValue pv) {
        // TODO:这里可以对于重复propertyName进行判断，直接用 List 没法做到
		this.propertyValueList.add(pv);
	}

	public List<PropertyValue> getPropertyValues() {
		return this.propertyValueList;
	}

}
```



### 2.2 BeanDefinitionReader 接口



这里主要是用 XML 来读取 bean 的定义并加载到 registry 的字典里面，其他直接定义 bean 对象的情况不需要这个 `BeanDefinitionReader`。

`BeanDefinitionReader`  是一个通过 localtion 地址来加载 `BeanDefinition` 的接口

```java
public interface BeanDefinitionReader {
    void loadBeanDefinitions(String location) throws Exception;
}
```



### 2.3 AbstractBeanDefinitionReader 抽象类实现

`AbstractBeanDefinitionReader` 实现上述接口的抽象类，实际上它并未具体实现了如何加载 `BeanDefinition`，而是规范了加载的流程：

- 我们还是需要用一个 `ResourceLoader` 来获得 `Resource` 再得到输入流；
- 我们要加载多个 `BeanDefinition`，用一个 `Map<String, BeanDifinition>` 的 registry 来保存读入的结果。



### 2.4 XmlBeanDefinitionReader 具体实现



具体实现如何从 XML 中解析生成 Bean 并保存到 registry 



```java
@Override
public void loadBeanDefinitions(String location) throws Exception {
    InputStream inputStream = getResourceLoader().getResource(location).getInputStream();
    doLoadBeanDefinitions(inputStream);
}
```



XML 中 bean 的定义如下，bean 要注明 id 即 bean 的 nane，以及对应的类

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean id="outputService" class="us.codecraft.tinyioc.OutputServiceImpl">
    </bean>

    <bean id="helloWorldService" class="us.codecraft.tinyioc.HelloWorldServiceImpl">
        <property name="text" value="Hello World!"></property>
        <property name="outputService" ref="outputService"></property>
    </bean>

    <bean id="autoProxyCreator" class="us.codecraft.tinyioc.aop.AspectJAwareAdvisorAutoProxyCreator"></bean>

    <bean id="timeInterceptor" class="us.codecraft.tinyioc.aop.TimerInterceptor"></bean>

    <bean id="aspectjAspect" class="us.codecraft.tinyioc.aop.AspectJExpressionPointcutAdvisor">
        <property name="advice" ref="timeInterceptor"></property>
        <property name="expression" value="execution(* us.codecraft.tinyioc.*.*(..))"></property>
    </bean>
</beans>
```



每读入一个 bean 条目，创建一个 `BeanDefinition`:

```java
protected void processBeanDefinition(Element ele) {
    String name = ele.getAttribute("id");
    String className = ele.getAttribute("class");
    // 创建新的 BeanDefinition
    BeanDefinition beanDefinition = new BeanDefinition();
    // 处理属性
    processProperty(ele, beanDefinition);
    beanDefinition.setBeanClassName(className);
    // 注册到 registry
    getRegistry().put(name, beanDefinition);
}
```

在处理 bean 属性的时候，如果出现了 `ref`，即要注入其他的 bean，这里特别定义了一个 `BeanReference` 类保存被引用的 bean，这里创建一个保存 bean 名称的 `BeanReference`，最后 `getBean()` 会从这个类里调。



```java
public class BeanReference {
    private String name;
    private Object bean;

    public BeanReference(String name) {
        this.name = name;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Object getBean() {
        return bean;
    }
    public void setBean(Object bean) {
        this.bean = bean;
    }
}


// XmlBeanDefinitionReader 处理 ref 属性
String ref = propertyEle.getAttribute("ref");
if (ref == null || ref.length() == 0) {
    throw new IllegalArgumentException("...");
}
BeanReference beanReference = new BeanReference(ref); // 用 ref 的 bean name 创建 BeanReference
beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, beanReference));
```



## 3. BeanFactory 工厂



### 3.1 BeanFactory 接口



`BeanFactory` 是通过 bean 的 name 调用 `getBean(String name)` 返回 bean 对象的一个接口。



### 3.2 AbstractBeanFactory 抽象类实现



`AbstractBeanFactory` 没有具体实现怎么生成 bean `Object`，只是规范了工厂生成 Bean、初始化、实例化、注入属性等一系列流程。

首先，`AbstractBeanFactory` 有一个 `beanDefinitionMap`，用来保存将要生成的 bean 类的定义信息：

```java
private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();
```

工厂的 `getBean()` 方法实现了单例模式：如果从类定义信息 `beanDefination` 中 `getBean()`得到的 bean 实例不存在，那么要新创建一个 bean，如果存在则直接返回。

```java
@Override
public Object getBean(String name) throws Exception {
    BeanDefinition beanDefinition = beanDefinitionMap.get(name);
    if (beanDefinition == null) {
        throw new IllegalArgumentException("No bean named " + name + " is defined");
    }
    Object bean = beanDefinition.getBean();
    if (bean == null) {
        bean = doCreateBean(beanDefinition);  // 1. 创建 bean 实例
        bean = initializeBean(bean, name);    // 2. 初始化 bean 实例
        beanDefinition.setBean(bean);		  // 3. bean 实例放回 beanDefinition 中
    }
    return bean;
}
```

虽然 `doCreateBean` 已经在 `beanDefinition` 中设置了 bean，但是后面初始化 bean 的时候，对 bean 又进行了改动，所以上面（第3步）还要 `setBean` 一次。



创建 bean 实例的过程由 `doCreateBean(BeanDefinition beanDefinition)`完成：

其又分为创建实例、定义中写入实例

```java
protected Object doCreateBean(BeanDefinition beanDefinition) throws Exception {
   Object bean = createBeanInstance(beanDefinition);
   beanDefinition.setBean(bean);
   applyPropertyValues(bean, beanDefinition);
   return bean;
}
```



`doCreateBean` 也分为 3 步：

1. 用 `createBeanInstance` 实例化 bean，是通过  `beanDefinition` 中的 `beanClass` 的 `newInstance()`方法
2. 还是要为 `beanDefinition` 更新 bean 实例：`setBean(bean)`
3. 调用 `applyPropertyValues` 方法 将 `beanDefinition` 中的属性对应用到 bean 实例中（这一步由子类实现）



这个第 3 步是如何注入属性，在此处也是这个 `AbstractBeanFactory` 未完成的功能，项目中由子类`AutowireCapableBeanFactory`实现。



类中还存在其他 AOP 相关的内容，后面再讲。



### 3.3 AutowireCapableBeanFactory 具体实现



这里注入属性只有一种方法，也就是自动装配内容，从 `BeanDefinition` 的 `propertyValueList` 读取属性名和属性值的键值对 `PropertyValue`，用反射的方法注入。这里优先使用属性对应的`set`方法注入属性，没有此方法才直接对属性赋值（因为通常是私有域，按理是不能去访问的）。



还有一种情况是属性是其他 bean 对象，那么获得的值是保存了bean name 的  `BeanReference`，还要用本 Factory 的 `getBean()` 方法找到这个 bean 再作为属性值。

```java
public class AutowireCapableBeanFactory extends AbstractBeanFactory {

   protected void applyPropertyValues(Object bean, BeanDefinition mbd) throws Exception {
      if (bean instanceof BeanFactoryAware) {
         ((BeanFactoryAware) bean).setBeanFactory(this);
      }
      for (PropertyValue propertyValue : mbd.getPropertyValues().getPropertyValues()) {
         Object value = propertyValue.getValue();
         if (value instanceof BeanReference) {
            BeanReference beanReference = (BeanReference) value;
            value = getBean(beanReference.getName());
         }

         try {
            Method declaredMethod = bean.getClass().getDeclaredMethod(
                  "set" + propertyValue.getName().substring(0, 1).toUpperCase()
                        + propertyValue.getName().substring(1), value.getClass());
            declaredMethod.setAccessible(true);

            declaredMethod.invoke(bean, value);
         } catch (NoSuchMethodException e) {
            Field declaredField = bean.getClass().getDeclaredField(propertyValue.getName());
            declaredField.setAccessible(true);
            declaredField.set(bean, value);
         }
      }
   }
}
```



## 4. ApplicationContext 更方便的接口



### 4.1 ApplicationContext 接口



`ApplicationContext` 只是简单继承了 `BeanFactory`，什么都没做，是一个标记接口。



注意上面的 `BeanFactory` 只是直接通过 BeanDefinition 实现了 bean 的装载、获取，bean 的来源（BeanDefinition是 怎么来的）没有指定。现在我们顺便结合 `BeanDefinitionReader` 获取 BeanDefinition 的 registry，再对 bean 进行实例化。



### 4.2 AbstractApplicationContext 抽象实现类



是上面 `ApplicationContext` 接口的抽象实现类，内部包含了一个 `BeanFactory`，内部主要方法有 `refresh()` 和 `getBean()` 。

```java
public void refresh() throws Exception {
   loadBeanDefinitions(beanFactory);
   registerBeanPostProcessors(beanFactory); // 注册 bean 后处理器
   onRefresh();
}
```

`refresh()`方法先调用 `loadBeanDefinitions`来读取 `BeanDefinition`，然后`onRefresh()` 间接调用 `beanFactory` 对 bean 预实例化。

所谓预实例化 `preInstantiateSingletons` 就是对所有 bean 调用 `getBean()` 方法，就生成实例了。

`getBean()` 是直接调用工厂 `beanFactory` 的 `getBean()` 获取实例。





### 4.3 ClassPathXmlApplicationContext 具体实现类



毫无疑问 `ClassPathXmlApplicationContext` 要做的就是如何 `loadBeanDefinitions`，回到我们的 

 `XmlBeanDefinitionReader`，它需要一个`ResourceLoader`，然后调用 `ResourceLoader` 的 `loadBeanDefinitions`来获取 bean 的定义：`BeanDefinition`。由于 `XmlBeanDefinitionReader` 是把`BeanDefinition`放在其 `registry` 字典中的，我们要调用 `beanFactory.registerBeanDefinition` 将它注册到 `beanFactory`。



除了 `BeanFactory`之外，还要给 `ClassPathXmlApplicationContext` 指定一个 location，即 XML 文件的路径。

```java
public class ClassPathXmlApplicationContext extends AbstractApplicationContext {

	private String configLocation;

	public ClassPathXmlApplicationContext(String configLocation) throws Exception {
		this(configLocation, new AutowireCapableBeanFactory());
	}

	public ClassPathXmlApplicationContext(String configLocation, AbstractBeanFactory beanFactory) throws Exception {
		super(beanFactory);
		this.configLocation = configLocation;
		refresh();
	}

	@Override
	protected void loadBeanDefinitions(AbstractBeanFactory beanFactory) throws Exception {
		XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(new ResourceLoader());
		xmlBeanDefinitionReader.loadBeanDefinitions(configLocation);
		for (Map.Entry<String, BeanDefinition> beanDefinitionEntry : xmlBeanDefinitionReader.getRegistry().entrySet()) {
			beanFactory.registerBeanDefinition(beanDefinitionEntry.getKey(), beanDefinitionEntry.getValue());
		}
	}

}
```



> 注 1：在 Spring 的实现中，对 `ApplicatinoContext` 的分层更为细致。`AbstractApplicationContext` 中为了实现 **不同来源** 的 **不同类型** 的资源加载类定义，把这两步分层实现。以“从类路径读取 XML 定义”为例，首先使用 `AbstractXmlApplicationContext` 来实现 **不同类型** 的资源解析，接着，通过 `ClassPathXmlApplicationContext` 来实现 **不同来源** 的资源解析。 



> 注 2：在 tiny-spring 的实现中，先用 `BeanDefinitionReader` 读取 `BeanDefiniton` 后，保存在内置的 `registry`（键值对为 `String` - `BeanDefinition` 的哈希表，通过 `getRigistry()` 获取）中，然后由 `ApplicationContext` 把 `BeanDefinitionReader` 中 `registry` 的键值对一个个赋值给 `BeanFactory` 中保存的 `beanDefinitionMap`。**而在 Spring 的实现中**，`BeanDefinitionReader` 直接操作 `BeanDefinition` ，它的 `getRegistry()` 获取的不是内置的 `registry`，而是 `BeanFactory` 的实例。如何实现呢？以 `DefaultListableBeanFactory` 为例，它实现了一个 `BeanDefinitonRigistry` 接口，该接口把 `BeanDefinition` 的 **注册** 、**获取** 等方法都暴露了出来，这样，`BeanDefinitionReader` 可以直接通过这些方法把 `BeanDefiniton` 直接加载到 `BeanFactory` 中去。





---



下面说说 AOP 实现部分



## 5. AOP 的实现说明





### 5.1 JDK 动态代理

JDK 的动态代理主要有两个关键类：`Proxy` 和 `InvocationHandler`，都在 `java.lang.reflect` 中。

`Proxy` 可以通过: `Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)` 返回一个代理对象。

`InvocationHandler` 可以通过`Object invoke(Object proxy, Method method,Object[] args)` 来处理代理对象具体的方法调用。

```java
public class JdkDynamicAopProxy extends AbstractAopProxy implements InvocationHandler {

    public JdkDynamicAopProxy(AdvisedSupport advised) {
        super(advised);
    }

	@Override
	public Object getProxy() {
		return Proxy.newProxyInstance(getClass().getClassLoader(), advised.getTargetSource().getInterfaces(), this);
	}

	@Override
	public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
		MethodInterceptor methodInterceptor = advised.getMethodInterceptor();
		if (advised.getMethodMatcher() != null
				&& advised.getMethodMatcher().matches(method, advised.getTargetSource().getTarget().getClass())) {
			return methodInterceptor.invoke(new ReflectiveMethodInvocation(advised.getTargetSource().getTarget(),
					method, args));
		} else {
			return method.invoke(advised.getTargetSource().getTarget(), args);
		}
	}

}
```

当然这里先得有个 `AopProxy` 接口，可以 `getProxy()`，然后 `AbstractAopProxy` 实现 `AopProxy` 并且设置了名为 advised 的`AdvisedSupport`属性， 

`AdvisedSupport` 是存放拦截的内容 `TargetSource` 、拦截器 `MethodInterceptor` 以及拦截匹配 `MethodMatcher`的类，因为这三个东西是同时处理的。

上面的 `invoke(Object proxy, Method method, Object[] args)` 的拦截逻辑是 advised 中的 `MethodMatcher` 如果能够 match 我们的目标类、方法和参数，那么就调用 `advised` 中的 `MethodInterceptor` 来 invoke 操作，否则是用原生的 `method.invoke(object, args)`，即不拦截，调用原对象的方法。

再介绍下 `MethodInterceptor`，这是 `org.aopalliance.inercept` 中的对象

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-18-22-47_r15.png)

`MethodInterceptor` 对象调用 `invoke` 要传入一个实现`MethodInvocation`接口的对象（有个`getMethod()`方法），注释中写道：method invocation 就是能被方法拦截器拦截的连接点，最重要的其实是 JoinPoint 中的 `proceed()` 方法。

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-18-27-26_r84.png)

看 `MethodInterceptor` 是如何 invoke 调用 `MethodInvocation` 的：

```java
public class TimerInterceptor implements MethodInterceptor {

	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
    // 方法前操作
		long time = System.nanoTime();
		System.out.println("Invocation of Method " + invocation.getMethod().getName() + " start!");
    // invocation 继续执行原先的代码
		Object proceed = invocation.proceed();
    // 方法后操作
		System.out.println("Invocation of Method " + invocation.getMethod().getName() + " end! takes " + (System.nanoTime() - time)
				+ " nanoseconds.");
		return proceed;
	}

}
```

看 JdkDynamicAopProxyTest：

```java
public class JdkDynamicAopProxyTest {

	@Test
	public void testInterceptor() throws Exception {
		// --------- helloWorldService without AOP
		ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
		HelloWorldService helloWorldService = (HelloWorldService) applicationContext.getBean("helloWorldService");
		helloWorldService.helloWorld();

		// --------- helloWorldService with AOP
		// 1. 设置被代理对象(Joinpoint)
		AdvisedSupport advisedSupport = new AdvisedSupport();
		TargetSource targetSource = new TargetSource(helloWorldService, HelloWorldServiceImpl.class,
				HelloWorldService.class);
		advisedSupport.setTargetSource(targetSource);

		// 2. 设置拦截器(Advice)
		TimerInterceptor timerInterceptor = new TimerInterceptor();
		advisedSupport.setMethodInterceptor(timerInterceptor);

        // 3. 这里我们忽略 advisedSupport 中的 MethodMatcher，因为它是后面 AspectJ 才用到的，这里只做测试
        
		// 3. 创建代理(Proxy)
		JdkDynamicAopProxy jdkDynamicAopProxy = new JdkDynamicAopProxy(advisedSupport);
		HelloWorldService helloWorldServiceProxy = (HelloWorldService) jdkDynamicAopProxy.getProxy();
        

		// 4. 基于AOP的调用
		helloWorldServiceProxy.helloWorld();

	}
}
```



### 5.2 bean 初始化的后处理器

工厂方法中的 `getBean()`的两个步骤：

1. `doCreateBean` 创建 bean 实例，注入属性
2. `initializeBean` 初始化 bean 实例



**第一步**在创建实例的时候，有个 `applyProperties`，具体操作在唯一的具体 Factory 类 `AutowireCapableBeanFactory` 的 `applyPropertyValues` 方法（为了给 bean 注入属性）中，我们有这么一段：

```java
if (bean instanceof BeanFactoryAware) {
    ((BeanFactoryAware) bean).setBeanFactory(this);
}
```

也就是如果 bean 实现了 `BeanFactoryAware` 接口，就转换成 `BeanFactoryAware` 调用 `setBeanFactory`，将工厂容器的引用传回 bean 中，这样 bean 就可以获得对容器操作的权限，允许了编写扩展 IoC 容器功能的 bean。

> 注：上面没有说明，`AbstractBeanFactory` 里面除了 beanDefinationMap 之外，还有 beanDefinationNames 的 List 存放每个 BeanDefiniton 的名称，更重要的是还有一个`beanPostProcessors`的列表（存放实现`BeanPostProcessor`接口的对象）



对于**第二步**在代码中其实没有写初始化 bean 的逻辑，这里初始化逻辑之前和之后，从 `beanPostProcessor`  列表中获得每个后处理器，对 bean 进行处理。

```java
protected Object initializeBean(Object bean, String name) throws Exception {
		for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {
			bean = beanPostProcessor.postProcessBeforeInitialization(bean, name);
		}

		// TODO:call initialize method
		for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {
            bean = beanPostProcessor.postProcessAfterInitialization(bean, name);
		}
    return bean;
}
```

当然，经过 `BeanPostProcessor` 处理之后的 bean **可以是动态代理返回的代理类**，比如我们在`postProcessAfterInitialization(bean, name)`中去调用 `JdkDynamicAopProxy`对象的 `getProxy`来返回一个新的 代理bean，这样就在初始化 bean 的过程中，动态代理了 bean 对象。

由于 `ApplicationContext` 有个 `refresh` 方法，里面有 `registerBeanPostProcessors` 方法，其实就是加到 `BeanFactory` 的 `beanPostProcessors` 列表里面。

XML 的 bean 定义中，加上这个 BeanPostProcessor 的 bean 就可以了

```xml
<bean id="beanInitializeLogger" class="us.codecraft.tinyioc.BeanInitializeLogger">
```

```java
// 当然这里并没有创建代理对象，还是返回原理的bean
public class BeanInitializeLogger implements BeanPostProcessor {
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws Exception {
		System.out.println("Initialize bean " + beanName + " start!");
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws Exception {
		System.out.println("Initialize bean " + beanName + " end!");
		return bean;
	}
}
```

在初始化 bean 时调用后处理器，这里是默认所有 bean 都会被所有 `BeanPostProcessor` 处理。





### 5.3 AspectJ 管理切面



#### 5.3.1 Pointcut、ClassFilter、MethodMatcher



先介绍一下`Pointcut`接口，即切点，提供用于类过滤的 `ClassFilter`、方法匹配的`MethodMatcher`。

这三者是紧密相关的。

```java
public interface Pointcut {
    ClassFilter getClassFilter();
    MethodMatcher getMethodMatcher()
}

public interface ClassFilter {
    boolean matches(Class targetClass);
}

public interface MethodMatcher {
    boolean matches(Method method, Class targetClass);
}

```



#### 5.3.2 PointcutAdvisor 组合 Pointcut 和 Advice



接下来`PointcutAdvisor`是切点通知器，可以`getPointcut()`得到一个切点，又因为继承自 `Advisor`可以得到一个 `Advice`增强：`Advice`就是一个通知对象，用于实现具体的方法拦截。

```java
public interface PointcutAdvisor extends Advisor{
   Pointcut getPointcut();
}
```

之前这部分内容我们是在 `AdvisedSupport` 中保存。



---

#### 5.3.2 AspectJExpressionPointcut 同时实现三个接口来match



我们看具体的 `AspectJExpressionPointcut`，它既是一个 `Pointcut`，又自己实现了 `ClassFilter`和`MethodMatcher`，所以它的 `getPointcut()`就返回自己了，另外还可以 match 类和方法。

**这个Pointcut需要传入一个 expression 匹配表达式**，用 `PointcutParser`结合这个 expression 来构造 `PointcutExpression`，接下来就可以用``PointcutExpression` 来 match 匹配类和方法了。

```java
@Override
public boolean matches(Class targetClass) {
    checkReadyToMatch();
    return pointcutExpression.couldMatchJoinPointsInType(targetClass);
}

@Override
public boolean matches(Method method, Class targetClass) {
    checkReadyToMatch();
    ShadowMatch shadowMatch = pointcutExpression.matchesMethodExecution(method);
    if (shadowMatch.alwaysMatches()) {
        return true;
    } else if (shadowMatch.neverMatches()) {
        return false;
    }
    // TODO:其他情况不判断了！见org.springframework.aop.aspectj.RuntimeTestWalker
    return false;
}
```



#### 5.3.3 AspectJExpressionPointcutAdvisor 封装上面的 Pointcut 和 Advice

实现 `PointcutAdvisor` 接口。

上面 `AspectJExpressionPointcut` 是一个传入 expression 然后 match 的功能，我们新的 Advisor 组合了  Pointcut 和 Advice。

```java
public class AspectJExpressionPointcutAdvisor implements PointcutAdvisor {

    private AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();

    private Advice advice;

    public void setAdvice(Advice advice) {
        this.advice = advice;
    }

    public void setExpression(String expression) {
        this.pointcut.setExpression(expression);
    }

	@Override
	public Advice getAdvice() {
		return advice;
	}

    @Override
	public Pointcut getPointcut() {
		return pointcut;
	}
}
```



#### 5.3.4 AspectJAwareAdvisorAutoProxyCreator （ AOP 实现关键类）



 `AspectJAwareAdvisorAutoProxyCreator` 类（以下简称 `AutoProxyCreator`）是实现 AOP 植入的关键类，它实现了两个接口：

1. `BeanPostProcessor` ：在 `postProcessorAfterInitialization` 方法中，使用动态代理的方式，返回一个对象的代理对象。解决了 **在 IoC 容器的何处植入 AOP** 的问题。
2. `BeanFactoryAware` ：这个接口提供了对 `BeanFactory` 的感知，这样，尽管它是容器中的一个 `Bean`，却可以获取容器的引用，进而获取容器中所有的切点对象，决定对哪些对象的哪些方法进行代理。解决了 **为哪些对象提供 AOP 的植入** 的问题。



```java
public class AspectJAwareAdvisorAutoProxyCreator implements BeanPostProcessor, BeanFactoryAware {

    // AspectJAwareAdvisorAutoProxyCreator 是一个 bean，但是实现了 BeanFactoryAware，可以保存工厂
	private AbstractBeanFactory beanFactory;

    // 初始化前什么都不作
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws Exception {
		return bean;
	}

    // 初始化后
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws Exception {
 		// 注意，这里的 bean 是doCreation中 initializeBean 初始化的时候传进来的
        
        // 是否要进行拦截处理
        // 一些 Adivisor 增强类，不需要增强自身，排除
		if (bean instanceof AspectJExpressionPointcutAdvisor) {
			return bean;
		}
        // 一些方法拦截器，排除
        if (bean instanceof MethodInterceptor) {
            return bean;
        }
        // 通过 bean factory 找到所有 AspectJExpression 的 PointcutAdvisor(保存了pointcut和advisor)
		List<AspectJExpressionPointcutAdvisor> advisors = beanFactory
				.getBeansForType(AspectJExpressionPointcutAdvisor.class);
        // 遍历所有的 pointcut & advisor
		for (AspectJExpressionPointcutAdvisor advisor : advisors) {
            // 类匹配与否
			if (advisor.getPointcut().getClassFilter().matches(bean.getClass())) {
                // 还是用之前的 AdvisedSupport 来保存拦截类、方法匹配、拦截器
				AdvisedSupport advisedSupport = new AdvisedSupport();
                // 拦截器从 advisor.getAdvice() 得到：MethodInterceptor 继承自 Advice
				advisedSupport.setMethodInterceptor((MethodInterceptor) advisor.getAdvice());
				advisedSupport.setMethodMatcher(advisor.getPointcut().getMethodMatcher());

				TargetSource targetSource = new TargetSource(bean, bean.getClass().getInterfaces());
				advisedSupport.setTargetSource(targetSource);
				// 返回新的 JDK 动态代理
				return new JdkDynamicAopProxy(advisedSupport).getProxy();
			}
		}
        // 匹配不了就直接返回
		return bean;
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws Exception {
		this.beanFactory = (AbstractBeanFactory) beanFactory;
	}
}
```



XML 注入：

```xml
 <bean id="autoProxyCreator" class="us.codecraft.tinyioc.aop.AspectJAwareAdvisorAutoProxyCreator"></bean>

<bean id="aspectjAspect" class="us.codecraft.tinyioc.aop.AspectJExpressionPointcutAdvisor">
    <property name="advice" ref="timeInterceptor"></property>
    <property name="expression" value="execution(* us.codecraft.tinyioc.*.*(..))"></property>
</bean>
```



总而言之：`AspectJAwareAdvisorAutoProxyCreator`只要注入后，会自动在实例初始化阶段收集所有的 `AspectJExpressionPointcutAdvisor`(属性如上只需要提供 advice ref是什么，以及 expression 就好了)，实现 AspectJ 的动态代理。





## 6. CGLib 动态代理



Java 的 JDK 动态代理只能代理接口，即我们的代理对象 Target Class 只能是接口，而不能是 Impl 实现类。

CGLib 可以实现对类的代理，是对 ASM 的封装，Spring 中使用了 CGLib。

可以定义一个工厂类`ProxyFactory`，用于根据 TargetSource 类型自动创建代理，这样就需要在调用者代码中去进行判断。



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-21-15-30_r28.png)

```java
// AopProxy 可以是 JdkDynamicAopProxy 或者 Cglib2AopProxy
public class ProxyFactory extends AdvisedSupport implements AopProxy {

   @Override
   public Object getProxy() {
      return createJdkDynamicAopProxy
      // return createAopProxy().getProxy();
   }

   protected final AopProxy createCglibAopProxy() {
      return new Cglib2AopProxy(this);
   }
    
   protected final AopProxy createJdkDynamicAopProxy() {
      return new JdkDynamicAopProxy(this);
   } 
}
```



### Cglib2AopProxy



```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

public class Cglib2AopProxy extends AbstractAopProxy {

   public Cglib2AopProxy(AdvisedSupport advised) {
      super(advised);
   }

   // 返回一个 proxy : enhanced
   @Override
   public Object getProxy() {
      Enhancer enhancer = new Enhancer();
      enhancer.setSuperclass(advised.getTargetSource().getTargetClass());
      enhancer.setInterfaces(advised.getTargetSource().getInterfaces());
      enhancer.setCallback(new DynamicAdvisedInterceptor(advised));  // 设置回调的 MethodInterceptor（主要是 里面的 intercept 函数返回的 MethodInvocation 对象）
      Object enhanced = enhancer.create();
      return enhanced;
   }

   private static class DynamicAdvisedInterceptor implements MethodInterceptor {

      private AdvisedSupport advised;

      private org.aopalliance.intercept.MethodInterceptor delegateMethodInterceptor;

      private DynamicAdvisedInterceptor(AdvisedSupport advised) {
         this.advised = advised;
         this.delegateMethodInterceptor = advised.getMethodInterceptor();
      }

      // 什么情况下拦截
      @Override
      public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
         if (advised.getMethodMatcher() == null
               || advised.getMethodMatcher().matches(method, advised.getTargetSource().getTargetClass())) {
            return delegateMethodInterceptor.invoke(new CglibMethodInvocation(advised.getTargetSource().getTarget(), method, args, proxy));
         }
         return new CglibMethodInvocation(advised.getTargetSource().getTarget(), method, args, proxy).proceed();
      }
   }

   // MethodInvocation 如何拦截，主要是 preceed()
    // 主要这个 proceed 方法，内部是用 这个 getProxy() 返回的Enhancer代理来 invoke
   private static class CglibMethodInvocation extends ReflectiveMethodInvocation {

      private final MethodProxy methodProxy;

      public CglibMethodInvocation(Object target, Method method, Object[] args, MethodProxy methodProxy) {
         super(target, method, args);
         this.methodProxy = methodProxy;
      }

      @Override
      public Object proceed() throws Throwable {
         return this.methodProxy.invoke(this.target, this.arguments);
      }
   }

}
```



##  7. 类图



### Beans 部分

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-21-29-05_r30.png)



### Context 部分

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-21-28-02_r89.png)

### AOP部分

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-09-21-24-11_r43.png)
