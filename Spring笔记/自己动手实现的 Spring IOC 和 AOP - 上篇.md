[TOC]

## 一、简单的IOC和AOP实现

### 1.1 简单的IOC

先从简单的 IOC 容器实现开始，最简单的 IOC 容器只需4步即可实现，如下：

1. 加载 xml 配置文件，遍历其中的标签
2. 获取标签中的 id 和 class 属性，加载 class 属性对应的类，并创建 bean
3. 遍历标签中的标签，获取属性值，并将属性值填充到 bean 中
4. 将 bean 注册到 bean 容器中

代码结构介绍：

```java
SimpleIOC     // IOC 的实现类，实现了上面所说的4个步骤
SimpleIOCTest    // IOC 的测试类
Car           // IOC 测试使用的 bean
Wheel         // 同上 
ioc.xml       // bean 配置文件
```

容器实现类 SimpleIOC 的代码：

```java
public class SimpleIOC {
	//使用Map数据结构做为容器
    private Map<String, Object> beanMap = new HashMap<>();

    public SimpleIOC(String location) throws Exception {
        loadBeans(location);
    }

    public Object getBean(String name) {
        Object bean = beanMap.get(name);
        if (bean == null) {
            throw new IllegalArgumentException("there is no bean with name " + name);
        }

        return bean;
    }

    private void loadBeans(String location) throws Exception {
        // 加载 xml 配置文件
        InputStream inputStream = new FileInputStream(location);
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        DocumentBuilder docBuilder = factory.newDocumentBuilder();
        Document doc = docBuilder.parse(inputStream);
        Element root = doc.getDocumentElement();
        NodeList nodes = root.getChildNodes();

        // 遍历 <bean> 标签
        for (int i = 0; i < nodes.getLength(); i++) {
            Node node = nodes.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                String id = ele.getAttribute("id");
                String className = ele.getAttribute("class");
                
                // 加载 beanClass
                Class beanClass = null;
                try {
                    beanClass = Class.forName(className);
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                    return;
                }

                   // 创建 bean
                Object bean = beanClass.newInstance();

                // 遍历 <property> 标签
                NodeList propertyNodes = ele.getElementsByTagName("property");
                for (int j = 0; j < propertyNodes.getLength(); j++) {
                    Node propertyNode = propertyNodes.item(j);
                    if (propertyNode instanceof Element) {
                        Element propertyElement = (Element) propertyNode;
                        String name = propertyElement.getAttribute("name");
                        String value = propertyElement.getAttribute("value");

                            // 利用反射将 bean 相关字段访问权限设为可访问
                        Field declaredField = bean.getClass().getDeclaredField(name);
                        declaredField.setAccessible(true);

                        if (value != null && value.length() > 0) {
                            // 将属性值填充到相关字段中
                            declaredField.set(bean, value);
                        } else {
                            String ref = propertyElement.getAttribute("ref");
                            if (ref == null || ref.length() == 0) {
                                throw new IllegalArgumentException("ref config error");
                            }
                            
                            // 将引用填充到相关字段中
                            declaredField.set(bean, getBean(ref));
                        }

                        // 将 bean 注册到 bean 容器中
                        registerBean(id, bean);
                    }
                }
            }
        }
    }

    private void registerBean(String id, Object bean) {
        beanMap.put(id, bean);
    }
}
```

容器测试使用的 bean 代码：

```java
public class Car {
    private String name;
    private String length;
    private String width;
    private String height;
    private Wheel wheel;
    
    // 省略其他不重要代码
}

public class Wheel {
    private String brand;
    private String specification ;
    
    // 省略其他不重要代码
}
```

bean 配置文件 ioc.xml 内容:

```xml
<beans>
    <bean id="wheel" class="com.titizz.simulation.toyspring.Wheel">
        <property name="brand" value="Michelin" />
        <property name="specification" value="265/60 R18" />
    </bean>

    <bean id="car" class="com.titizz.simulation.toyspring.Car">
        <property name="name" value="Mercedes Benz G 500"/>
        <property name="length" value="4717mm"/>
        <property name="width" value="1855mm"/>
        <property name="height" value="1949mm"/>
        <property name="wheel" ref="wheel"/>
    </bean>
</beans>
```

IOC 测试类 SimpleIOCTest：

```java
public class SimpleIOCTest {
    @Test
    public void getBean() throws Exception {
        String location = SimpleIOC.class.getClassLoader().getResource("spring-test.xml").getFile();
        SimpleIOC bf = new SimpleIOC(location);
        Wheel wheel = (Wheel) bf.getBean("wheel");
        System.out.println(wheel);
        Car car = (Car) bf.getBean("car");
        System.out.println(car);
    }
}
```

### 1.2 简单的 AOP 实现

AOP 的实现是基于代理模式的，这一点相信大家应该都知道。代理模式是AOP实现的基础，代理模式不难理解，这里就不花篇幅介绍了。在介绍 AOP 的实现步骤之前，先引入 Spring AOP 中的一些概念，接下来我们会用到这些概念。

通知（Advice）

```tex
通知定义了要织入目标对象的逻辑，以及执行时机。
Spring 中对应了 5 种不同类型的通知：
· 前置通知（Before）：在目标方法执行前，执行通知
· 后置通知（After）：在目标方法执行后，执行通知，此时不关系目标方法返回的结果是什么
· 返回通知（After-returning）：在目标方法执行后，执行通知
· 异常通知（After-throwing）：在目标方法抛出异常后执行通知
· 环绕通知（Around）: 目标方法被通知包裹，通知在目标方法执行前和执行后都被会调用
```

切点（Pointcut）

```tex
如果说通知定义了在何时执行通知，那么切点就定义了在何处执行通知。所以切点的作用就是
通过匹配规则查找合适的连接点（Joinpoint），AOP 会在这些连接点上织入通知。
```

切面（Aspect）

```tex
切面包含了通知和切点，通知和切点共同定义了切面是什么，在何时，何处执行切面逻辑。
```

说完概念，接下来我们来说说简单 AOP 实现的步骤。这里 AOP 是基于 JDK 动态代理实现的，只需3步即可完成：

1. 定义一个包含切面逻辑的对象，这里假设叫 logMethodInvocation
2. 定义一个 Advice 对象（实现了 InvocationHandler 接口），并将上面的 logMethodInvocation 和 目标对象传入
3. 将上面的 Adivce 对象和目标对象传给 JDK 动态代理方法，为目标对象生成代理

上面步骤比较简单，不过在实现过程中，还是有一些难度的，这里要引入一些辅助接口才能实现。接下来就来介绍一下简单 AOP 的代码结构：

```tex
MethodInvocation 接口  // 实现类包含了切面逻辑，如上面的 logMethodInvocation
Advice 接口        // 继承了 InvocationHandler 接口
BeforeAdvice 类    // 实现了 Advice 接口，是一个前置通知
SimpleAOP 类       // 生成代理类
SimpleAOPTest      // SimpleAOP 从测试类
HelloService 接口   // 目标对象接口
HelloServiceImpl   // 目标对象
```

MethodInvocation 接口代码：

```java
public interface MethodInvocation {
    void invoke();
}
```

Advice 接口代码：

```java
public interface Advice extends InvocationHandler {}
```

BeforeAdvice 实现代码：

```java
public class BeforeAdvice implements Advice {
    private Object bean;
    private MethodInvocation methodInvocation;

    public BeforeAdvice(Object bean, MethodInvocation methodInvocation) {
        this.bean = bean;
        this.methodInvocation = methodInvocation;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 在目标方法执行前调用通知
        methodInvocation.invoke();
        return method.invoke(bean, args);
    }
}
```

SimpleAOP 实现代码：

```java
public class SimpleAOP {
    public static Object getProxy(Object bean, Advice advice) {
        return Proxy.newProxyInstance(SimpleAOP.class.getClassLoader(), 
                bean.getClass().getInterfaces(), advice);
    }
}
```

HelloService 接口，及其实现类代码：

```java
public interface HelloService {
    void sayHelloWorld();
}

public class HelloServiceImpl implements HelloService {
    @Override
    public void sayHelloWorld() {
        System.out.println("hello world!");
    }
}
```

SimpleAOPTest 代码:

```java
public class SimpleAOPTest {
    @Test
    public void getProxy() throws Exception {
        // 1. 创建一个 MethodInvocation 实现类
        MethodInvocation logTask = () -> System.out.println("log task start");
        HelloServiceImpl helloServiceImpl = new HelloServiceImpl();
        
        // 2. 创建一个 Advice
        Advice beforeAdvice = new BeforeAdvice(helloServiceImpl, logTask);
        
        // 3. 为目标对象生成代理
        HelloService helloServiceImplProxy = (HelloService) SimpleAOP.getProxy(helloServiceImpl,beforeAdvice);
        
        helloServiceImplProxy.sayHelloWorld();
    }
}
```

以上实现了简单的 IOC 和 AOP，不过实现的 IOC 和 AOP 还很简单，且只能独立运行。IOC和AOP两个模块没有整合在一起，且IOC在加载bean的过程中，AOP不能对beanzhi织入通知。下面将详细说一下升级版IOC和AOP。

## 二、升级版IOC和AOP实现

### 2.1 IOC实现

#### 2.1.1 BeanFactory的生命流程：

1. BeanFactory加载Bean配置文件，将读取到的Bean配置封装成BeanDefinition对象。
2. 将封装好的BeanDefinition对象注册到BeanDefinition容器中。
3. 注册BeanPostProcessor相关实现类到BeanPostProcessor容器中。
4. BeanFactory进入就绪状态。
5. 外部调用BeanFactory的`getBean(String name)`方法，BeanFactory着手实例化相应的bean。
6. 重复步骤3和4，直至程序退出，BeanFactory被销毁。

#### 2.1.2 BeanDefinition 及 其他一些类的介绍

在详细介绍 IOC 容器的工作原理前，这里先介绍一下实现 IOC 所用到的一些辅助类，包括BeanDefinition、BeanReference、PropertyValues、PropertyValue。

BeanDefinition，从字面意思上翻译成中文就是 “Bean 的定义”。从翻译结果中就可以猜出这个类的用途，即根据 Bean 配置信息生成相应的 Bean 详情对象。举个例子，如果把 Bean 比作是电脑 💻，那么 BeanDefinition 就是这台电脑的配置清单。我们从外观上无法看出这台电脑里面都有哪些配置，也看不出电脑的性能咋样。但是通过配置清单，我们就可了解这台电脑的详细配置。我们可以知道这台电脑是不是用了牙膏厂的 CPU，BOOM 厂的固态硬盘等。透过配置清单，我们也就可大致评估出这台电脑的性能。

上面那个例子还是比较贴切的，但是只是个例子，和实际还是有差距的。那么在具体实现中，BeanDefinition 和 xml 是怎么对应的呢？答案在下面：



看完上图，我想大家对 BeanDefinition 的用途有了更进一步的认识。接下来我们来说说上图中的 ref 对应的 BeanReference 对象。BeanReference 对象保存的是 bean 配置中 ref 属性对应的值，在后续 BeanFactory 实例化 bean 时，会根据 BeanReference 保存的值去实例化 bean 所依赖的其他 bean。

接下来说说 PropertyValues 和 PropertyValue 这两个长的比较像的类，首先是PropertyValue。PropertyValue 中有两个字段 name 和 value，用于记录 bean 配置中的标签的属性值。然后是PropertyValues，PropertyValues 从字面意思上来看，是 PropertyValue 复数形式，在功能上等同于 List。那么为什么 Spring 不直接使用 List，而自己定义一个新类呢？答案是要获得一定的控制权，看下面的代码：

```java
public class PropertyValues {

    private final List<PropertyValue> propertyValueList = new ArrayList<PropertyValue>();

    public void addPropertyValue(PropertyValue pv) {
        // 在这里可以对参数值 pv 做一些处理，如果直接使用 List，则就不行了
        this.propertyValueList.add(pv);
    }

    public List<PropertyValue> getPropertyValues() {
        return this.propertyValueList;
    }

}
```

好了，辅助类介绍完了，接下来我们继续 BeanFactory 的生命流程探索。

#### 2.1.3 xml的解析

BeanFactory 初始化时，会根据传入的 xml 配置文件路径加载并解析配置文件。但是加载和解析 xml 配置文件这种脏活累活，BeanFactory 可不太愿意干，它只想高冷的管理容器中的 bean。于是 BeanFactory 将加载和解析配置文件的任务委托给专职人员 BeanDefinitionReader 的实现类 XmlBeanDefinitionReader 去做。那么 XmlBeanDefinitionReader 具体是怎么做的呢？XmlBeanDefinitionReader 做了如下几件事情：

1. 将 xml 配置文件加载到内存中
2. 获取根标签下所有的标签
3. 遍历获取到的标签列表，并从标签中读取 id，class 属性
4. 创建 BeanDefinition 对象，并将刚刚读取到的 id，class 属性值保存到对象中
5. 遍历标签下的标签，从中读取属性值，并保持在 BeanDefinition 对象中
6. 将 <id, BeanDefinition> 键值对缓存在 Map 中，留作后用
7. 重复3、4、5、6步，直至解析结束

#### 2.1.4 注册BeanPostProcessor

BeanPostProcessor 接口是 Spring 对外拓展的接口之一，其主要用途提供一个机会，让开发人员能够插手 bean 的实例化过程。通过实现这个接口，我们就可在 bean 实例化时，对bean 进行一些处理。比如，我们所熟悉的 AOP 就是在这里将切面逻辑织入相关 bean 中的。正是因为有了 BeanPostProcessor 接口作为桥梁，才使得 AOP 可以和 IOC 容器产生联系。

接下来说说 BeanFactory 是怎么注册 BeanPostProcessor 相关实现类的。

XmlBeanDefinitionReader 在完成解析工作后，BeanFactory 会将它解析得到的 <id, BeanDefinition> 键值对注册到自己的 beanDefinitionMap 中。BeanFactory 注册好 BeanDefinition 后，就立即开始注册 BeanPostProcessor 相关实现类。这个过程比较简单：

1. 根据 BeanDefinition 记录的信息，寻找所有实现了 BeanPostProcessor 接口的类。
2. 实例化 BeanPostProcessor 接口的实现类
3. 将实例化好的对象放入 List中
4. 重复2、3步，直至所有的实现类完成注册

#### 2.1.5 getBean 过程解析

在完成了 xml 的解析、BeanDefinition 的注册以及 BeanPostProcessor 的注册过程后。BeanFactory 初始化的工作算是结束了，此时 BeanFactory 处于就绪状态，等待外部程序的调用。

外部程序一般都是通过调用 BeanFactory 的 getBean(String name) 方法来获取容器中的 bean。BeanFactory 具有延迟实例化 bean 的特性，也就是等外部程序需要的时候，才实例化相关的 bean。这样做的好处是比较显而易见的，第一是提高了 BeanFactory 的初始化速度，第二是节省了内存资源。下面我们就来详细说说 bean 的实例化过程：

1. 实例化 bean 对象，类似于 new XXObject()
2. 将配置文件中配置的属性填充到刚刚创建的 bean 对象中。
3. 检查 bean 对象是否实现了 Aware 一类的接口，如果实现了则把相应的依赖设置到 bean 对象中。比如如果 bean 实现了 BeanFactoryAware 接口，Spring 容器在实例化bean的过程中，会将 BeanFactory 容器注入到 bean 中。
4. 调用 BeanPostProcessor 前置处理方法，即 postProcessBeforeInitialization(Object bean, String beanName)。
5. 检查 bean 对象是否实现了 InitializingBean 接口，如果实现，则调用 afterPropertiesSet 方法。或者检查配置文件中是否配置了 init-method 属性，如果配置了，则去调用 init-method 属性配置的方法。
6. 调用 BeanPostProcessor 后置处理方法，即 postProcessAfterInitialization(Object bean, String beanName)。我们所熟知的 AOP 就是在这里将 Adivce 逻辑织入到 bean 中的。
7. 注册 Destruction 相关回调方法。
8. bean 对象处于就绪状态，可以使用了。
9. 应用上下文被销毁，调用注册的 Destruction 相关方法。如果 bean 实现了 DispostbleBean 接口，Spring 容器会调用 destroy 方法。如果在配置文件中配置了 destroy 属性，Spring 容器则会调用 destroy 属性对应的方法。

上述流程从宏观上对 Spring 中 singleton 类型 bean 的生命周期进行了描述，接下来说说所上面流程中的一些细节问题。
先看流程中的第二步 -- 设置对象属性。在这一步中，对于普通类型的属性，例如 String，Integer等，比较容易处理，直接设置即可。但是如果某个 bean 对象依赖另一个 bean 对象，此时就不能直接设置了。Spring 容器首先要先去实例化 bean 依赖的对象，实例化好后才能设置到当前 bean 中。大致流程如下：



上面图片描述的依赖比较简单，就是 BeanA 依赖 BeanB。现在考虑这样一种情况，BeanA 依赖 BeanB，BeanB 依赖 BeanC，BeanC 又依赖 BeanA。三者形成了循环依赖，如下所示：



对于这样的循环依赖，根据依赖注入方式的不同，Spring 处理方式也不同。如果依赖靠构造器方式注入，则无法处理，Spring 直接会报循环依赖异常。这个理解起来也不复杂，构造 BeanA 时需要 BeanB 作为构造器参数，此时 Spring 容器会先实例化 BeanB。构造 BeanB 时，BeanB 又需要 BeanC 作为构造器参数，Spring 容器又不得不先去构造 BeanC。最后构造 BeanC 时，BeanC 又依赖 BeanA 才能完成构造。此时，BeanA 还没构造完成，BeanA 要等 BeanB 实例化好才能完成构造，BeanB 又要等 BeanC，BeanC 等 BeanA。这样就形成了死循环，所以对于以构造器注入方式的循环依赖是无解的，Spring 容器会直接报异常。对于 setter 类型注入的循环依赖则可以顺利完成实例化并依次注入。

循环依赖问题说完，接下来 bean 实例化流程中的第6步 -- 调用 BeanPostProcessor 后置处理方法。先介绍一下 BeanPostProcessor 接口，BeanPostProcessor 接口中包含了两个方法，其定义如下：

```java
public interface BeanPostProcessor {

    Object postProcessBeforeInitialization(Object bean, String beanName) throws Exception;

    Object postProcessAfterInitialization(Object bean, String beanName) throws Exception;
}
```

BeanPostProcessor 是一个很有用的接口，通过实现接口我们就可以插手 bean 的实例化过程，为拓展提供了可能。我们所熟知的 AOP 就是在这里进行织如入，具体点说是在 postProcessAfterInitialization(Object bean, String beanName) 执行织入逻辑的。下面就来说说 Spring AOP 织入的流程，以及 AOP 是怎样和 IOC 整合的。先说 Spring AOP 织入流程，大致如下：

1. 查找实现了 PointcutAdvisor 类型的切面类，切面类包含了 Pointcut 和 Advice 实现类对象。
2. 检查 Pointcut 中的表达式是否能匹配当前 bean 对象。
3. 如果匹配到了，表明应该对此对象织入 Advice。
4. 将 bean，bean class对象，bean实现的interface的数组，Advice对象传给代理工厂 ProxyFactory。代理工厂创建出 AopProxy 实现类，最后由 AopProxy 实现类创建 bean 的代理类，并将这个代理类返回。此时从 postProcessAfterInitialization(Object bean, String beanName) 返回的 bean 此时就不是原来的 bean 了，而是 bean 的代理类。原 bean 就这样被无感的替换掉了，是不是有点偷天换柱的感觉。

大家现在应该知道 AOP 是怎样作用在 bean 上的了，那么 AOP 是怎样和 IOC 整合起来并协同工作的呢？下面就来简单说一下。

Spring AOP 生成代理类的逻辑是在 AbstractAutoProxyCreator 相关子类中实现的，比如 DefaultAdvisorAutoProxyCreator、AspectJAwareAdvisorAutoProxyCreator 等。上面说了 BeanPostProcessor 为拓展留下了可能，这里 AbstractAutoProxyCreator 就将可能变为了现实。AbstractAutoProxyCreator 实现了 BeanPostProcessor 接口，这样 AbstractAutoProxyCreator 可以在 bean 初始化时做一些事情。光继承这个接口还不够，继承这个接口只能获取 bean，要想让 AOP 生效，还需要拿到切面对象（包含 Pointcut 和 Advice）才行。所以 AbstractAutoProxyCreator 同时继承了 BeanFactoryAware 接口，通过实现该接口，AbstractAutoProxyCreator 子类就可拿到 BeanFactory，有了 BeanFactory，就可以获取 BeanFactory 中所有的切面对象了。有了目标对象 bean，所有的切面类，此时就可以为 bean 生成代理对象了。