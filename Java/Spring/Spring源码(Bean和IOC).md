# Spring源码(Bean和IOC)

首次看Spring源码, 可能会有一些错误.

## 创建Spring启动项目
在Spring源码中新建一个模块`spring-hpc`, 并添加简单的依赖和启动程序.
1. 添加依赖(`build.gradle`文件)
	```gradle
	plugins {  
	 id 'java'  
	}  

	group 'org.springframework'  
	version '5.3.5-SNAPSHOT'  

	repositories {  
	 mavenCentral()  
	}  

	dependencies {  
	 compile project(":spring-web")  
		compile project(":spring-webmvc")  

		testCompile group: 'junit', name: 'junit', version: '4.12'  
	}
	```
	
2. 创建项目的配置文件`MainConfig.java`
	```java
	@Configuration  
	@ImportResource(locations = "classpath:Beans.xml")  
	@ComponentScan(basePackages = {"com.spring.hpc.circle"})  
	public class MainConfig {  

	}

	```
3. 创建`Bean.xml`文件
	
	```xml
	<?xml version="1.0" encoding="UTF-8"?>  
	<beans xmlns="http://www.springframework.org/schema/beans"  
	 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
	 xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">  

	 <bean id="instanceA" class="com.spring.hpc.circle.InstanceA" >  
	 	<property name="instanceB" ref="instanceB"></property>  
	 </bean>  
	 <bean id="instanceB" class="com.spring.hpc.circle.InstanceB" >  
	 	<property name="instanceA" ref="instanceA"></property>  
	 </bean>  
	</beans>
	```
	
4. 创建启动类
	```java
	public class MainClass {  

		public static void main(String\[\] args) {  
			//创建IOC容器  
			 AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);  

			 //去容器的缓存中直接拿  
			 InstanceA instanceA = ctx.getBean(InstanceA.class);  
			 ctx.close();  
		 }  
	}
	```
	
5. 创建其它实体类, 其中`InstanceA`,`InstanceB`是循环依赖的关系, 
    ```java
    /**
     * InstanceA
     **/
    public class InstanceA {  

        private InstanceB instanceB;  

        public InstanceB getInstanceB() {  
        	return instanceB;  
        }  

        public void setInstanceB(InstanceB instanceB) {  
            this.instanceB = instanceB;  
     	}  

        public InstanceA(InstanceB instanceB) {  
            this.instanceB = instanceB;  
     	}  
    
        public InstanceA() {  
            System.out.println("InstanceA实例化");  
     	}  
    }
    
    /**
     * InstanceB
     **/
    public class InstanceB {  
    
        private InstanceA instanceA;  
    
     	public InstanceA getInstanceA() {  
            return instanceA;  
     	}  
        public void setInstanceA(InstanceA instanceA) {  
            this.instanceA = instanceA;  
     	}  

        public InstanceB(InstanceA instanceA) {  
            this.instanceA = instanceA;  
     	}  
    
        public InstanceB() {  
            System.out.println("InstanceB实例化");  
     	}  
    }
    
    /**
     * Person
     **/
    @Component  
    public class Person {  
    
        @Value("zhangsan")  
        private String name;  
    
        @Value("18")  
        private String age;  
    
        public String getAge() {  
                return age;  
        }  
    
        public void setAge(String age) {  
                this.age = age;  
        }  
    
        public String getName() {  
    
            return name;  
        }  
    
        public void setName(String name) {  
            this.name = name;  
        }  
    }
    ```

## Spring源码解析

当Spring启动项目创建好后, 便可以从`main()`方法中的`AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class)` 进入代码.

### 加载Spring启动时所必须的beanDefinition

1. 首先代码进入到`AnnotationConfigApplicationContext`的构造方法中

   ```java
   public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   		this();
   		register(componentClasses);
   		refresh();
   }
   ```

   通过调用`this()`进入上述构造方法的重载构造方法中

   ```java
   public AnnotationConfigApplicationContext() {
       StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
       this.reader = new AnnotatedBeanDefinitionReader(this);
       createAnnotatedBeanDefReader.end();
       this.scanner = new ClassPathBeanDefinitionScanner(this);
   }
   ```

   在上述代码中第2行是标记应用程序启动的标记, 比较重要的代码在第3行和第5行

2. 进入`this.reader = new AnnotatedBeanDefinitionReader(this);`中进行查看, 进入到`AnnotatedBeanDefinitionReader`类中

   ```java
   public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
   		this(registry, getOrCreateEnvironment(registry));
   }
   ```

   其中`getOrCreateEnvironment()`方法是获取应用程序启动的环境, 然后通过`this()`方法进入了重载类中

   ```java
   public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
         Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
         Assert.notNull(environment, "Environment must not be null");
         this.registry = registry;
         this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
         AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
   }
   ```

   在这个方法体中, 第2,3,5行进行的是判断代码, 而关键的逻辑在第6行.

3. 第2步中关键的代码在第6行, 然后进入第6行查看

      ```java
      public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
      		registerAnnotationConfigProcessors(registry, null);
      }
      ```

      该方法继续调用它的重载方法, 这一步方法是加载Spring项目启动所必须的Spring内部的`BeanDefinition`

      ```java
      public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
      			BeanDefinitionRegistry registry, @Nullable Object source) {
      
          // 获取默认的bean工厂(DefaultListableBeanFactory)
          DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
          if (beanFactory != null) {
              if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
                  beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
              }
              if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
                  beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
              }
          }
      
          // 具有名称和别名的beanDefinition
          Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
      
          // 添加BeanDefinition : internalConfigurationAnnotationProcessor
          if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
              RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
              def.setSource(source);
              beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
          }
      
          // 添加BeanDefinition : internalAutowiredAnnotationProcessor
          if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
              RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
              def.setSource(source);
              beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
          }
      
          // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
          if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
              RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
              def.setSource(source);
              beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
          }
      
          // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
          if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
              RootBeanDefinition def = new RootBeanDefinition();
              try {
                  def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                                                      AnnotationConfigUtils.class.getClassLoader()));
              }
              catch (ClassNotFoundException ex) {
                  throw new IllegalStateException(
                      "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
              }
              def.setSource(source);
              beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
          }
      
          // 添加BeanDefinition :  internalEventListenerProcessor
          if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
              RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
              def.setSource(source);
              beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
          }
      
          // 添加BeanDefinition :  internalEventListenerFactory
          if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
              RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
              def.setSource(source);
              beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
          }
      
          return beanDefs;
      }
      ```

      当该方法执行完后, 能够在调试界面发现如下图, 在`beanDefinitionMap`中添加了上述代码执行注册的`BeanDefinition`

      ![image-20210311162701315](/home/hpc/GitCode/JavaKnowledge/Java/Spring/Spring源码(Bean和IOC).assets/image-20210311162701315.png)

4. 然后继续执行代码, 知道回到第2步的重载构造方法中, 然后看到`this.scanner = new ClassPathBeanDefinitionScanner(this);`, 根据代码的类名可以大体看成该类的作用是在ClassPath路径下进行BeanDefinitioin的扫描. 执行该行代码, 进入到`ClassPathBeanDefinitionScanner`类的构造方法中

      ```java
      public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
          this(registry, true);
      }
      ```

      然后继续执行`this()`方法, 进入到该构造方法中的重载方法中

      ```java
      public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
          this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
      }
      ```

      其中`getOrCreateEnvironment()`方法与第2步中的作用相同, 然后继续执行`this()`方法, 进入到另一个重载方法中

      ```java
      public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                            Environment environment) {
      
          this(registry, useDefaultFilters, environment,
               (registry instanceof ResourceLoader ? (ResourceLoader) registry : null));
      }
      ```

      接着继续执行, 进入到另一个重载的构造方法中

      ```java
      public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                            Environment environment, @Nullable ResourceLoader resourceLoader) {
      
          Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
          this.registry = registry;
      
          if (useDefaultFilters) {
              registerDefaultFilters();
          }
          setEnvironment(environment);
          setResourceLoader(resourceLoader);
      }
      ```

5. 根据上面的代码, 首先进入的第8行`registerDefaultFilters();`

      ```java
      protected void registerDefaultFilters() {
      		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
      		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
      		try {
      			this.includeFilters.add(new AnnotationTypeFilter(
      					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
      			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
      		}
      		catch (ClassNotFoundException ex) {
      			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
      		}
      		try {
      			this.includeFilters.add(new AnnotationTypeFilter(
      					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
      			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
      		}
      		catch (ClassNotFoundException ex) {
      			// JSR-330 API not available - simply skip.
      		}
      	}
      ```

      该方法的作用是注册默认过滤器, 这将默认注册所有具有`@Component`元注解的所有注解, 例如`@Controller`, `@Service`等. 然后根据项目是否支持`JSR-250`, `JSR-330`来添加相关的注解.
      
6. 然后进入到第4步`setEnvironment(environment);` 这行代码的意义便是设置环境, 并不涉及核心逻辑. 然后代码继续执行, 进入`setResourceLoader(resourceLoader);` 中, 该方法源码如下:

      ```java
      @Override
      public void setResourceLoader(@Nullable ResourceLoader resourceLoader) {
          this.resourcePatternResolver = ResourcePatternUtils.getResourcePatternResolver(resourceLoader);
          this.metadataReaderFactory = new CachingMetadataReaderFactory(resourceLoader);
          this.componentsIndex = CandidateComponentsIndexLoader.loadIndex(this.resourcePatternResolver.getClassLoader());
      }
      ```

      该方法注释上描述是通过设置ResourceLoader, 以便于后面解析资源(这里面我看得不太清楚)
