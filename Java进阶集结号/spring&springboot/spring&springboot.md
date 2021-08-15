### Spring 相关
1. BeanFactory和 ApplicationContext有什么区别？

>  ![img](https://img-blog.csdn.net/20150310105859070?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVkeWZ1bg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 
>
> 从上面的类结构图中可以看出来，ApplicationContext 是 BeanFactory接口的子接口
>
> 其中BeanFactory获得配置文件的实例是：
>
> <span style="white-space:pre">	</span>// 使用BeanFactory 读取配置文件
> 	BeanFactory beanFactory = new XmlBeanFactory(new ClassPathResource("applicationContext.xml"));
> 	HelloService helloService4 = (HelloService) beanFactory.getBean("helloService");
> 	helloService4.sayHello();
>
> ApplicationContext获取配置文件实例的方法是：
> <span style="white-space:pre">	</span>// 使用Spring Ioc 方式 获得HelloService 实例
> 	ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml"); // 获得工厂实例
> 	HelloService helloService2 = (HelloService) applicationContext.getBean("helloService"); // 通过id获得实例
> 	//helloService2.setInfo("itcast"); // 已经配置依赖注入 
> 	helloService2.sayHello();
>
> 其实两个在代码看来就是在获取配置文件的时候 的差异，他们还有其他的差异：
> 1）BeanFactory 采用的是延迟加载，第一次getBean的时候才会初始化Bean
>
> 2）ApplicationContext是对BeanFactory的扩展，提供了更多的功能
>
> 国际化处理
> 事件传递
> Bean自动装配
> 各种不同应用层的Context实现
>
> 结论：开发中尽量使用ApplicationContext 就可以了

1. Spring IOC 的理解，其初始化过程

2. Spring Bean 的生命周期

3. Spring MVC 的工作原理？

4. Spring 循环注入的原理？

5. Spring 中用到了那些设计模式？

6. Spring AOP的理解，各个术语，他们是怎么相互工作的？

7. Spring框架中的单例bean是线程安全的吗?

8. Spring @ Resource和Autowired有什么区别？

9. Spring 的不同事务传播行为有哪些，有什么作用？

10. Spring Bean 的加载过程是怎样的？

11. 请举例说明@Qualifier注解

12. Spring 是如何管理事务的，事务管理机制？

13. 使用Spring框架的好处是什么？

14. Spring由哪些模块组成？

15. ApplicationContext通常的实现是什么？

16. 什么是Spring的依赖注入？

17. 你怎样定义类的作用域？

18. Spring框架中的单例bean是线程安全的吗？

19. 你可以在Spring中注入一个null 和一个空字符串吗？

20. 你能说下 Spring Boot 与 Spring 的区别吗

21. SpringBoot 的自动配置是怎么做的？

22. @RequestMapping 的作用是什么？

23. spring boot 有哪些方式可以实现热部署？

24. 说说Ioc容器的加载过程

25. 为什么 Spring 中的 bean 默认为单例？

26. 说说Spring中的@Configuration

27. FileSystemResource 和ClassPathResource 有何区别？

28. 什么是 Swagger？你用 Spring Boot 实现了它吗？

29. spring的controller是单例还是多例，怎么保证并发的安全。

30. 说一下Spring的核心模块

31. 如何向 Spring Bean 中注入一个 Java.util.Properties

32. 如何给Spring 容器提供配置元数据?

33. 如何在Spring中如何注入一个java集合，实现过吗？

34. 什么是基于Java的Spring注解配置? 举几个例子？

35. 怎样开启注解装配？

36. Spring支持哪些事务管理类型

37. 在Spring AOP 中，关注点和横切关注的区别是什么？

38. spring 中有多少种IOC 容器？

39. 描述一下 DispatcherServlet 的工作流程

40. 介绍一下 WebApplicationContext吧

41. Spring Boot 的配置文件有哪几种格式？它们有什么区别？

42. Spring Boot 需要独立的容器运行吗？

43. Spring Boot 自动配置原理是什么？

44. RequestMapping 和 GetMapping 的不同之处在哪里？

45. 如何使用Spring Boot实现异常处理？

46. Spring Boot 中如何解决跨域问题 ?

47. Spring Boot 如何实现热部署 ?

48. Spring Boot打成的 jar 和普通的jar有什么区别呢?

49. bootstrap.properties 和 application.properties 有何区别 ?

50. springboot启动机制  

51. spring和springboot常见问题

    > https://xiaozhuanlan.com/topic/4923687015