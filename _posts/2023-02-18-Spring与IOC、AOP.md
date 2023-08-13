---
category: Java
---

## Spring与IOC

Spring是Java中最为常用的框架，其目的是简化复杂应用程序的开发，降低代码间的耦合性，提高代码的可维护性和可扩展性。其最为重要的核心思想就是控制反转，即IOC；面向切面编程，即AOP。

### 控制反转(IOC)

#### 什么是IOC

Java是一门面向对象的语言。甚至号称一切皆对象，当我们的程序越来越复杂，创建和需要管理的对象也就越来越多，这在极大程度上降低了我们开发的效率，试想一下，你创建了一个Service类，他有很多个属性，每个属性的类又有很多属性.....当我们想要使用这个Service类的时候，得new一个该类的对象，为了让其能够工作，还必须得填充其属性，因此又得new很多其属性对应类的对象，一层套一层，不仅写起来麻烦，可维护性还不高，而且还十分冗余。

而IOC核心目标就是解决这一问题，由它来负责对象的创建、管理，而开发人员则专心于代码逻辑即可。这一操作将对象的控制权从开发人员交到了Spring的手中，即为控制反转。

#### Spring是如何实现IOC的

**1、识别用户交由IOC管理的对象**：在编写代码时，通过注解、配置等操作对需要交由IOC管理的对象做一个标记，这样在项目启动时IOC就会知道需要创建那些对象、需要将那些对象放到自己的容器里面。典型的注解有: @Bean、@Component。@Bean注解修饰方法，将方法的返回值放入到IOC容器中，值得一提的是，Spring将由IO容器管理的对象称为Bean。@Component，一般用于修饰类，Spring会调用类的无参构造方法，为该类创建一个对象实例，值得一提的是，Spring为许多功能不同的Component起了新名字，如@service、@Configuration、@Controller。

**2、为这些Bean设置属性(依赖注入，DI):** 即在IOC容器中找到适合的对象，用来作为当前对象的某个属性(个人理解)。一般使用@Autowired来标识需要被依赖注入的属性，这个注解会优先根据Bean的类型来进行注入，即在IOC容器中找到对应类型的Bean来作为当前对象的这个属性, 当同一个类型存在多个Bean时，使用@Autowired就需要指明Bean的名称了。设置好属性后就可以使用这些Bean了，使用方法也是依赖注入。例如：

```java
@Service
class SumService{
    public int sum(int a, int b){
        //code...
    }
}

@RestController
class SumController{
    @Autowired
    private SumService sumService;
    
    @GetMapping("/getSum")
    public Integer getSum(Integer a, Integer b){
        return sumService.sum(a, b);
    }
}

```

**3、销毁这些对象：**在应用程序关闭或Bean不再需要时，Spring容器会销毁Bean实例。这可以通过容器的关闭或销毁方法来触发。

####  Bean的生命周期

Bean的生命周期从代码来看比较复杂，但其实换个角度，从人的生命周期去理解，就会好很多。

人的一辈子，从出生，成长，然后死亡。

出生就对应Bean的实例化；成长就对应Bean的初始化；死亡对应Bean被销毁的阶段。

实例化阶段即创建对象实例；

初始化阶段即为对象注入各种属性，进行一些资源分配、数据校验，检查Bean是否准备就绪等。

死亡就是销毁对象，是否其占用的资源等。

同常在生命周期的相邻两个阶段之间又会有一些操作，如在实例化之前应该做些什么、在初始化之前应该做些什么、在销毁之前又应该做些什么。

下面的完整的Bean的生命周期：

1. 如果创建了一个类继承了InstantiationAwareBeanPostProcessorAdapter接口，并在配置文件中配置了该类的注入，即InstantiationAwareBeanPostProcessorAdapter和bean关联，则Spring将调用该接口的postProcessBeforeInstantiation（）方法。
2. 根据配置情况调用 Bean 构造方法或工厂方法实例化 Bean。
3. 如果InstantiationAwareBeanPostProcessorAdapter和bean关联，则Spring将调用该接口的postProcessAfterInstantiation（）方法。
4. 利用依赖注入完成 Bean 中所有属性值的配置注入。
5. 如果 Bean 实现了 BeanNameAware 接口，则 Spring 调用 Bean 的 setBeanName() 方法传入当前 Bean 的 id 值。
6. 如果 Bean 实现了 BeanFactoryAware 接口，则 Spring 调用 setBeanFactory() 方法传入当前工厂实例的引用。
7. 如果 Bean 实现了 ApplicationContextAware 接口，则 Spring 调用 setApplicationContext() 方法传入当前 ApplicationContext 实例的引用。
8. 如果 BeanPostProcessor 和 Bean 关联，则 Spring 将调用该接口的预初始化方法 postProcessBeforeInitialzation() 对 Bean 进行加工操作，此处非常重要，Spring 的 AOP 就是利用它实现的。
9. 如果 Bean 实现了 InitializingBean 接口，则 Spring 将调用 afterPropertiesSet() 方法。
10. 如果在配置文件中通过 init-method 属性指定了初始化方法，则调用该初始化方法。
11. 如果 BeanPostProcessor 和 Bean 关联，则 Spring 将调用该接口的初始化方法 postProcessAfterInitialization()。
12. 注意：以上工作完后才能以后就可以应用这个bean了，那这个bean是一个singleton的，所以一般这种情况下我们调用同一个id的bean会是在内容地址相同的实例，当然在spring配置文件中也可以配置非Singleton。
13. 如果 Bean 实现了 DisposableBean 接口，则 Spring 会调用 destory() 方法将 Spring 中的 Bean 销毁；如果在配置文件中通过 destory-method 属性指定了 Bean 的销毁方法，则 Spring 将调用该方法对 Bean 进行销毁。





