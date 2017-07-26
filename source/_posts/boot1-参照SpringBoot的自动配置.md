id: boot1
title: 参照SpringBoot的自动配置
categories: web java
tags: [boot,spring,java]
---

> “无感”就是最棒的设计

## 问题
~~想直接解决问题不愿听故事的，进屋坐坐看第二节。~~

最近在公司领到一个任务，优化公司的缓存框架的使用体验。公司的缓存框架封装成一个JAR文件，提供基于注解的对几种缓存的一致性使用。覆盖的缓存方案有`ehcache` `memcached` `redis`，先向公司大牛致敬。
要使用缓存框架，需要做这么几件事：

1. 引入缓存框架的JAR文件

2. 引入想要使用的缓存JAR，比如memcached.jar

3. 做相关的Spring Bean配置，比如要使用Memcached缓存就要有如下的配置：

   ```xml
   <!-- memcache配置示例 -->
   <bean name="ljmemcachedClient" class="net.rubyeye.xmemcached.utils.XMemcachedClientFactoryBean" destroy-method="shutdown">
     <!-- 多个地址空格分隔 -->
     <property name="servers" value="${cache.memcacheAddress}" />
     <!-- 连接池大小一般5已经足够用,根据项目组实际情况调整 -->
     <property name="connectionPoolSize" value="${cache.memcachePoolSize}" />
     <!-- 一致哈希分布 -->
     <property name="sessionLocator">
       <bean class="net.rubyeye.xmemcached.impl.KetamaMemcachedSessionLocator"></bean>
     </property>
     <!-- 二进制协议,提高数据传输效率,支持touch等 -->
     <property name="commandFactory">
       <bean class="net.rubyeye.xmemcached.command.BinaryCommandFactory"></bean>
     </property>
   </bean>
   ```

4. 必要的配置文件：

   ```properties
   cache.memcacheAddress=10.8.1.107:11211
   cache.memcachePoolSize=1
   ```

如果要使用redis缓存，则要引入另外一套配置。每个新项目都要配置，这显然是令人沮丧的。能不能简化这些配置呢？

- 方案一：在*缓存框架*中引入三种缓存的JAR包，并做三种缓存的`Bean`配置，不提供配置文件，由使用*缓存框架*的项目提供。
- 方案二：在*缓存框架*中提供三种缓存的“自动配置”，由使用*缓存框架*的项目提供配置文件和相关缓存JAR包，根据引入的缓存JAR包“自动配置”适合的`Bean`对象，完成缓存的使用。

来看下两种方案的优缺：

- 方案一
  - 优点：简单易实现
  - 缺点：JAR包臃肿，无论实际使用哪种缓存，都引入了三种缓存JAR，并且需要提供三种的配置文件。不符合程序员美学--简约
- 方案二：
  - 优点：“智能”发现缓存，按需引入，语义明确，使用方便
  - 缺点：程序实现复杂

我选择**方案二**。

然而**Spring Boot** 已经有成熟的自动配置实现方案，如果直接引入 *spring-boot-autoconfigure.jar* 会面临额外问题：

1. *spring-boot-autoconfigure.jar* 没有“直接”提供针对 `memcahced` 的自动配置；
2. *spring-boot-autoconfigure.jar* 提供了太多自动配置支持，而我只想要关于缓存的几个；

那就只好模仿**Spring Boot**搞一搞了。

---

## 终焉

> 撒花...完结

在开始赘述之前，先给出最终的成品。

如何使用自动配置？

1. 引入自动配置JAR包

2. 使用注解 `@EnableAutoConfigure` 开启自动配置。建议写在基础类上或者专门提供一个类，如：

   ```java
   @EnableAutoConfigure
   public class AutoConfigure {
   }
   ```

3. 根据需要选择具体缓存，引入JAR并给出配置信息。比如memcached.jar

   ```properties
   cache.memcached.servers=10.8.1.107:11211
   cache.memcached.connectionPoolSize=2
   ```

PS: 这里有一个纠结的地方，其实可以做到省略第2步，也就是在自动配置JAR包里把这事做了。但笔者感觉这样不好，好像是降低了使用的成本，但是损失了使用的自由。所以最终还是没有进一步封装。这里仅代表笔者个人感受，未必是对的。 

PS2：然而什么是对的呢？世界本来就不是0和1.



## 赘述

接下来详细说说如何实现的。

### 有条件的创建

> 除非满足条件，不然。。。哼！

最本质的需求是根据条件加载Bean。 `Spring4` 提供了实现方案 — `@Conditional` ，可以通过条件判断创建 `Bean` 。

示例：

```java
// java config
@Configuration
public class TestBeanConfig {

    // 根据条件创建， 条件写在TestConditional类里
    @Bean
    @Conditional(TestConditional.class)
    public TestBean createTestBean() {
        return new TestBean();
    }
}

// 配套的条件类实现
public class TestConditional implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return false;
    }
}
```

只有当 `TestConditional.matches()` 结果为 **true** 时才会创建 `TestBean` 。

#### 注解 @Conditional

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Conditional {

    Class<? extends Condition>[] value();
}
```

- 作用范围： 类，方法
- 包含参数： value, 接口`Condition`的实现类数组。数组内所有的条件都要满足哟！

如果value中的条件都满足，就创建接下来的Bean。没啥好说的了；

#### 接口 Condition

```java
public interface Condition {

    // 当返回值为true时，创建Bean；否则忽略。
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
接口很简单只有一个matches方法，可以通过context和metadata获得相应判断信息，辅助判断；

##### 参数 ConditionContext

```java
public interface ConditionContext {
    // 检查Bean定义
    BeanDefinitionRegistry getRegistry();
    // 检查Bean是否存在，甚至探查Bean的属性
    ConfigurableListableBeanFactory getBeanFactory();
    // 检查环境变量是否存在以及它的值是什么
    Environment getEnvironment();
    // 检查加载的资源
    ResourceLoader getResourceLoader();
    // 加载并检查类是否存在
    ClassLoader getClassLoader();
}
```

##### AnnotatedTypeMetadata

```java
public interface AnnotatedTypeMetadata {

    boolean isAnnotated(String annotationName);
    Map<String, Object> getAnnotationAttributes(String annotationName);
    Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString);
    MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName);
    MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString);
}
```

- 借助 `isAnnotated()` 能够判断带有 `@Bean` 注解的方法是不是还有其他特定注解；
- 借助其他方法可以检查 `@Bean` 注解的方法上其他注解的属性；

### 尝试自动配置

#### 判断条件

至此，基础技能已经 *GET* ，要解决第一节的问题，只要做判断：当前类路径下是否存在某类，有就创建无则忽略。
**Spring Boot** 在 `@Conditional` 基础上扩展了几个注解方便做此类判断：`@ConditionalOnClass`,  `@ConditionalOnMissingClass` ,  `@ConditionalOnBean` ,  `@ConditionalOnMissingBean` ;
这里就不贴源码了，想看的朋友去这里 **org.springframework.boot.autoconfigure.condition**;
当然可以自行编写条件类实现判断，我这里就懒省事了，直接使用 **Spring Boot** 的实现。~~（才没有这么简单）~~

#### 编写自动配置类

编写 `memcached` 和 `redis` 的Java Config类。核心代码：
```java
/**
 * memcached 缓存自动配置
 * 在引入googlecode-xmemcached.jar时启用
 * Created by mw4157 on 16/7/6.
 */
@Configuration
@ConditionalOnClass(MemcachedClient.class)
public class MemcachedAutoConfiguration {
    /**
     * 创建一个工厂Bean
     */
    @Bean
    @ConditionalOnMissingBean
    public XMemcachedClientFactoryBean memcachedClientFactory() {
        return ...;
    }
}

/**
 * redis 自动配置
 * 在引入redis-clients.jar和spring-data-redis.jar时启用
 * Created by mw4157 on 16/7/8.
 */
@Configuration
@ConditionalOnClass({ JedisConnection.class, RedisOperations.class, Jedis.class })
public class RedisAutoConfiguration {

    /**
     * Redis 连接配置
     */
    @Bean
    @ConditionalOnMissingBean(RedisConnectionFactory.class)
    public JedisConnectionFactory redisConnectionFactory() throws UnknownHostException {
        return ...;
    }
}
```
代码的核心功能是创造需要的Bean，同时辅以`@Conditional`相关的注解，就可以在大意上解决问题。

## 赘述2.0

之所以说大意上，是因为还有其他内容没有注意到：
1. 如何将配置文件与自动配置类结合起来？
2. `@Configuration` 注解本身继承于 `@Component` ，可以被 **component-scan** 扫描到，如果项目配置的扫描根路径范围过大包括了自动配置类，则可能会引发错误（这其实是不可避免的“缺陷”，读者可以尝试在自己的项目中扫描 `org.spring` 会发现项目无法启动）。以 `MemcachedAutoConfiguration` 为例，如果项目没有引入 *googlecode-xmemcached.jar*，这个类上的 `MemcachedClient.class` 一定是无法识别的，此时被 **Spring** 的 **component-scan** 直接扫描到，会在加载类（load class)阶段报错*NoClassDefFoundError*。

### 问题1.配置文件

**Spring** 提供了一个注解用于导入配置文件中的数据 —  `@Value` 。
基本用法：
```java
@Value("${name}")
private String address;
@Value("${name:defaultValue}")
private String address2;
```
`name` 对应 *properties* 文件里的 **key** ; `defaultValue` 是参数的默认值，在 **key** 不存在时生效。有一点需要注意的是：如何默认null？
```java
@Value("${name:null}")
```
这是不对的，注入的是字符串“null”，正确的做法是：
```java
@Value("${name:#{null}}")
```

### 问题2.自动配置的类加载

要解决问题2，首先要避免自动配置类被项目的 **component-scan** 直接发现，其次我们要在合适的时候让 **Spring** 发现自动配置类，并在加载前判断 `@Conditional` 是否满足，不满足就直接跳过。老实说，我没有信心说明白这个问题。

下面介绍几个相关知识点：

#### @Configuration

```JAVA
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    String value() default "";
}
```
可以注意到 `@Configuration` 是被 `@Component` 修饰的，所以可以被 **component-scan** 发现。

#### component-scan

```xml
<!-- 自动扫描且不扫描@Controller-->
<context:component-scan base-package="org.msdg">
	<context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller" />
	<context:exclude-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice" />
	<context:exclude-filter type="regex" expression="org.msdg.core.conf.autoconfigure.*"/>
</context:component-scan>
```
**Spring** 会自动扫描 *base-package* 下的Java类，如果发现带有 `@Component` (包括用其修饰的注解 `@Repository`   `@Service`   `@Controller`   `@RestController`   `@Configuration` 等)的类，将会放入Spring容器中管理，成为众多Bean里的一员。
**component-scan** 还包括两个子标签 *<context:include-filter>* 和 *<context:exclude-filter>* ,注意两个不是必须的，但如果同时出现，要保证 *include-filter* 出现在前面，并且如果范围重合，重合部分被后续的 *exclude-filter* 排除掉。两个子标签都具有 `type` 和 `expression` 两个属性：
- type
    - annotation 	拥有指定注解的类
    - aspectj            AspectJ语法
    - assignable     指定class或interface的全名
    - regex		表达式，可以写类似org.spring.\*,org.spring包下的类都被包括
    - custom           Spring自定义类型，并没有研究过
- expression 就是表达式了，可以写全限定类名、符合规范的正则表达等

在必要的时候可以通过 *<context:exclude-filter>* 来避免扫描自动配置类。~~但真要这么做的话只能说明 **component-scan** 的范围过大了，不够精细，不优雅。~~

#### 使用Selector加载Bean

在避免了基础扫描后，我们要在适当的时候引入自动配置的类。这里需要用到接口 `DeferredImportSelector` 。
```JAVA
public interface DeferredImportSelector extends ImportSelector {
}
```
这是一个典型的“标识型”接口，没有任何需要实现的方法。来看下官方注释：
> A variation of ImportSelector that runs after all @Configuration beans have been processed. This type of selector can be particularly useful when the selected imports are @Conditional.
> Implementations can also extend the Ordered interface or use the Order annotation to indicate a precedence against other DeferredImportSelectors.

大意是这个接口的实现类将会在 `@Configuration` 的Bean处理后执行。并且 `DeferredImportSelector` 是专门用来选择导入 `@Conditional` 修饰的Bean。我们也可以用 `@Ordered` 决定多个 `DeferredImportSelector` 的执行顺序。
突破口就在这里， 让我们更详尽的了解一下接口 `ImportSelector` 的作用。
```JAVA
public interface ImportSelector {

    /**
     * Select and return the names of which class(es) should be imported based on
     * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
     */
    String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```
根据 *importingClassMetadata* 的值，从带有注解 `@Configuration` 的类中选择并返回合适的类命名数组，将其导入 **Spring** 容器。执行顺序是在 `@Configuration` 引入的Bean处理之后。接口 `ImportSelector` 的作用类似于注解 `@Import` 。

一个 `ImportSelector` 通常应该实现任意一个或多个 `Aware` 接口，比如：

- `EnvironmentAware`
- `BeanFactoryAware`
- `BeanClassLoaderAware`
- `ResourceLoaderAware`

#### 编写我们的`ImportSelector`

~~参考了Spring Boot的实现，哦哈哈，别告我抄袭~~
*代码是部分代码，完全体见文章结尾*
```JAVA
@Order(Ordered.LOWEST_PRECEDENCE - 1)
@Configuration
public class EnableAutoConfigurationImportSelector implements DeferredImportSelector,
		BeanClassLoaderAware, ResourceLoaderAware, BeanFactoryAware, EnvironmentAware {

	@Override
	public String[] selectImports(AnnotationMetadata metadata) {
		List<String> configurations = getCandidateConfigurations();
		configurations = removeDuplicates(configurations);
		return configurations.toArray(new String[configurations.size()]);
	}

	/**
	 * 核心方法,引入需要的类
	 */
	protected List<String> getCandidateConfigurations() {
		return SpringFactoriesLoader.loadFactoryNames(this.getClass(), getBeanClassLoader());
	}

	/**
	 * 删除重复引用
	 */
	protected final <T> List<T> removeDuplicates(List<T> list) {
		return new ArrayList<T>(new LinkedHashSet<T>(list));
	}

	...
}
```
为了方便多方面装配，四大接口全部实现了，~~但这不重要~~。核心方法是 *getCandidateConfigurations()* ，通过 `SpringFactoriesLoader` 加载自动配置类。**Spring Boot** 是这么用的，不要问我为什么，我也不知道有没有别的替代方案。T_T
`SpringFactoriesLoader` 会加载并实例化写在文件 "META-INF/spring.factories" 中的类。*key-value* 形式，需要写清全限定类名。根据官方解释是用来表达工厂类和多种实现类的管理，应该是类似父子关系，但这里并没有强制限定。所以，我们的 **spring.factories** 是这样的：
```properties
# Auto Configure
org.msdg.core.conf.autoconfigure.EnableAutoConfigurationImportSelector=\
  org.msdg.core.conf.autoconfigure.memcached.MemcachedAutoConfiguration,\
  org.msdg.core.conf.autoconfigure.redis.RedisAutoConfiguration
```
至此突破口已经准备好了。

#### 使用@Import注册Bean
现在只需要将 `ImportSelector` 放入Spring容器中就可以了。
因此我们要引入 `@Import` , 这个注解可以导入一个配置类到另一个配置类中。在 **Spring4.2** 中对这个注解进行了加强，可以直接将一个类加入Spring容器。那么只要写上 *@Import(EnableAutoConfigurationImportSelector.class)* 即可。
为了便于使用，这里自定义一个注解：
```JAVA
@Configuration
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfigure {
}
```

## 结语
先告一段落吧。如有表述不清，欢迎讨论。



## PS:

#### Memchached配置辅助类

```java
import org.springframework.beans.factory.annotation.Value;

/**
 * Created by mw4157 on 16/7/9.
 */
public class MemcachedProperties {

    @Value("${cache.memcached.servers:localhost}")
    private String servers;
    @Value("${cache.memcached.connectionPoolSize:1}")
    private int connectionPoolSize;
    @Value("${cache.memcached.sessionLocatorKind:ketama}")
    private String sessionLocatorKind;
    @Value("${cache.memcached.commandFactory:binary}")
    private String commandFactory;
	
  	// getter and setter
}
```

#### Memcached自动配置类

```java
import org.msdg.core.cache.impl.MemCacheManager;
import org.msdg.core.conf.autoconfigure.condition.ConditionalOnMissingBean;
import net.rubyeye.xmemcached.MemcachedClient;
import net.rubyeye.xmemcached.command.BinaryCommandFactory;
import net.rubyeye.xmemcached.command.KestrelCommandFactory;
import net.rubyeye.xmemcached.command.TextCommandFactory;
import net.rubyeye.xmemcached.impl.ArrayMemcachedSessionLocator;
import net.rubyeye.xmemcached.impl.ElectionMemcachedSessionLocator;
import net.rubyeye.xmemcached.impl.KetamaMemcachedSessionLocator;
import net.rubyeye.xmemcached.utils.XMemcachedClientFactoryBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import org.msdg.core.conf.autoconfigure.condition.ConditionalOnClass;

/**
 * memcached 缓存自动配置
 * 当类路径中存在MemcachedClient.class时,加载配置
 * Created by mw4157 on 16/7/6.
 */
@Configuration
@ConditionalOnClass(MemcachedClient.class)
public class MemcachedAutoConfiguration {

    /**
     * memcached配置实体
     * 用于引入相关的配置, 具体参见 {@link MemcachedProperties}
     */
    @Bean(name = "org.msdg.core.conf.autoconfigure.cache.MemcachedProperties")
    @ConditionalOnMissingBean
    public MemcachedProperties memcachedProperties() {
        return new MemcachedProperties();
    }

    /**
     * 抽象配置memcahced的逻辑
     * 具体配置可继承此类
     */
    protected abstract static class AbstractMemcachedConfiguration {
        @Autowired
        private MemcachedProperties properties;

        /**
         * 应用配置信息, 创建工厂类
         */
        protected final XMemcachedClientFactoryBean applyProperties(XMemcachedClientFactoryBean clientFactory) {
            // 设置服务器
            clientFactory.setServers(properties.getServers());
            // 连接池大小一般5已经足够用,根据项目组实际情况调整
            clientFactory.setConnectionPoolSize(properties.getConnectionPoolSize());

            // 分布策略
            switch (properties.getSessionLocatorKind()) {
                case "ketama":
                    // 一致性哈希(默认)
                    clientFactory.setSessionLocator(new KetamaMemcachedSessionLocator());
                    break;
                case "array":
                    // 余数哈希
                    clientFactory.setSessionLocator(new ArrayMemcachedSessionLocator());
                    break;
                case "election":
                    // 选举哈希
                    clientFactory.setSessionLocator(new ElectionMemcachedSessionLocator());
                    break;
            }

            // 谁来补充这里的注释, 我不造啊
            switch (properties.getCommandFactory()) {
                case "binary":
                    // 二进制协议,提高数据传输效率,支持touch等(默认)
                    clientFactory.setCommandFactory(new BinaryCommandFactory());
                    break;
                case "kestrel":
                    clientFactory.setCommandFactory(new KestrelCommandFactory());
                    break;
                case "text":
                    clientFactory.setCommandFactory(new TextCommandFactory());
                    break;
            }

            return clientFactory;
        }
    }

    /**
     * 创建一个工厂类
     */
    @Configuration
    protected static class MemcachedConnectionConfiguration extends AbstractMemcachedConfiguration{

        @Bean
        @ConditionalOnMissingBean
        public XMemcachedClientFactoryBean memcachedClientFactory() {
            return applyProperties(new XMemcachedClientFactoryBean());
        }
    }

    /**
     * 创建管理器, 用于配合公司基于注解的那套缓存使用方案
     */
    @Configuration
    protected static class MemcahchedConfiguration {

        @Bean
        @ConditionalOnMissingBean
        public MemCacheManager memcachedManager() {
            return new MemCacheManager();
        }
    }

    @Configuration
    protected static class MemcahchedConfiguration1 {

        @Bean
        @ConditionalOnMissingBean
        public MemCacheManager memcachedManager() {
            return new MemCacheManager();
        }
    }
}
```

#### Redis配置辅助类

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;

public class RedisProperties {

   /**
    * Database index used by the connection factory.
    */
   @Value("${cache.redis.database:0}")
   private int database;

   /**
    * Redis server host.
    */
   @Value("${cache.redis.host:localhost}")
   private String host;

   /**
    * Login password of the redis server.
    */
   @Value("${cache.redis.password:#{null}}")
   private String password;

   /**
    * Redis server port.
    */
   @Value("${cache.redis.port:6379}")
   private int port;

   /**
    * Connection timeout in milliseconds.
    */
   @Value("${cache.redis.timeout:300000}")
   private int timeout;

   @Autowired
   private Pool pool;

   @Autowired
   private Sentinel sentinel;

   public int getDatabase() {
      return this.database;
   }

   // getter and setter

   /**
    * Pool properties.
    */
   public static class Pool {

      /**
       * Max number of "idle" connections in the pool. Use a negative value to indicate
       * an unlimited number of idle connections.
       */
      @Value("${cache.redis.pool.maxIdle:8}")
      private int maxIdle;

      /**
       * Target for the minimum number of idle connections to maintain in the pool. This
       * setting only has an effect if it is positive.
       */
      @Value("${cache.redis.pool.minIdle:0}")
      private int minIdle;

      /**
       * Max number of connections that can be allocated by the pool at a given time.
       * Use a negative value for no limit.
       */
      @Value("${cache.redis.pool.maxActive:8}")
      private int maxActive;

      /**
       * Maximum amount of time (in milliseconds) a connection allocation should block
       * before throwing an exception when the pool is exhausted. Use a negative value
       * to block indefinitely.
       */
      @Value("${cache.redis.pool.maxWait:-1}")
      private int maxWait;

      // getter and setter
   }

   /**
    * Redis sentinel properties.
    */
   public static class Sentinel {

      /**
       * Name of Redis server.
       */
      @Value("${cache.redis.sentinel.master:#{null}}")
      private String master;

      /**
       * Comma-separated list of host:port pairs.
       */
      @Value("${cache.redis.sentinel.nodes:#{null}}")
      private String nodes;

      // getter and setter
   }
}
```

#### Redis自动配置类

```java
import org.msdg.core.cache.impl.RedisCacheManager;
import org.msdg.core.conf.autoconfigure.condition.ConditionalOnClass;
import org.msdg.core.conf.autoconfigure.condition.ConditionalOnMissingBean;
import org.msdg.core.conf.autoconfigure.condition.ConditionalOnMissingClass;
import org.apache.commons.pool2.impl.GenericObjectPool;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisNode;
import org.springframework.data.redis.connection.RedisSentinelConfiguration;
import org.springframework.data.redis.connection.jedis.JedisConnection;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.RedisOperations;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.util.Assert;
import org.springframework.util.StringUtils;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPoolConfig;

import java.net.UnknownHostException;
import java.util.ArrayList;
import java.util.List;

/**
 * Created by mw4157 on 16/7/8.
 */
@Configuration
@ConditionalOnClass({ JedisConnection.class, RedisOperations.class, Jedis.class })
public class RedisAutoConfiguration {

    @Bean(name = "org.springframework.autoconfigure.redis.RedisProperties")
    @ConditionalOnMissingBean
    public RedisProperties redisProperties() {
        return new RedisProperties();
    }

    @Bean(name = "org.springframework.autoconfigure.redis.RedisProperties.Pool")
    @ConditionalOnMissingBean
    public RedisProperties.Pool redisPoolProperties() {
        return new RedisProperties.Pool();
    }

    @Bean(name = "org.springframework.autoconfigure.redis.RedisProperties.Sentinel")
    @ConditionalOnMissingBean
    public RedisProperties.Sentinel redisSentinelProperties() {
        return new RedisProperties.Sentinel();
    }

    /**
     * Base class for Redis configurations.
     */
    protected abstract class AbstractRedisConfiguration {

        @Autowired
        protected RedisProperties properties;

        @Autowired(required = false)
        private RedisSentinelConfiguration sentinelConfiguration;

        protected final JedisConnectionFactory applyProperties(JedisConnectionFactory factory) {
            factory.setHostName(this.properties.getHost());
            factory.setPort(this.properties.getPort());
            if (this.properties.getPassword() != null) {
                factory.setPassword(this.properties.getPassword());
            }
            factory.setDatabase(this.properties.getDatabase());
            if (this.properties.getTimeout() > 0) {
                factory.setTimeout(this.properties.getTimeout());
            }
            return factory;
        }

        protected final RedisSentinelConfiguration getSentinelConfig() {
            if (this.sentinelConfiguration != null) {
                return this.sentinelConfiguration;
            }
            RedisProperties.Sentinel sentinelProperties = this.properties.getSentinel();
            if (sentinelProperties.getMaster() != null) {
                RedisSentinelConfiguration config = new RedisSentinelConfiguration();
                config.master(sentinelProperties.getMaster());
                config.setSentinels(createSentinels(sentinelProperties));
                return config;
            }
            return null;
        }

        private List<RedisNode> createSentinels(RedisProperties.Sentinel sentinel) {
            List<RedisNode> sentinels = new ArrayList<RedisNode>();
            String nodes = sentinel.getNodes();
            for (String node : StringUtils.commaDelimitedListToStringArray(nodes)) {
                try {
                    String[] parts = StringUtils.split(node, ":");
                    Assert.state(parts.length == 2, "Must be defined as 'host:port'");
                    sentinels.add(new RedisNode(parts[0], Integer.valueOf(parts[1])));
                }
                catch (RuntimeException ex) {
                    throw new IllegalStateException(
                            "Invalid redis sentinel " + "property '" + node + "'", ex);
                }
            }
            return sentinels;
        }

    }

    /**
     * Redis connection configuration.
     */
    @Configuration
    @ConditionalOnMissingClass("org.apache.commons.pool2.impl.GenericObjectPool")
    protected class RedisConnectionConfiguration extends AbstractRedisConfiguration {

        @Bean
        @ConditionalOnMissingBean(RedisConnectionFactory.class)
        public JedisConnectionFactory redisConnectionFactory() throws UnknownHostException {
            return applyProperties(new JedisConnectionFactory(getSentinelConfig()));
        }

    }

    /**
     * Redis pooled connection configuration.
     */
    @Configuration
    @ConditionalOnClass(GenericObjectPool.class)
    protected class RedisPooledConnectionConfiguration extends AbstractRedisConfiguration {

        @Bean
        @ConditionalOnMissingBean(RedisConnectionFactory.class)
        public JedisConnectionFactory redisConnectionFactory() throws UnknownHostException {
            return applyProperties(createJedisConnectionFactory());
        }

        private JedisConnectionFactory createJedisConnectionFactory() {
            if (this.properties.getPool() != null) {
                return new JedisConnectionFactory(getSentinelConfig(), jedisPoolConfig());
            }
            return new JedisConnectionFactory(getSentinelConfig());
        }

        private JedisPoolConfig jedisPoolConfig() {
            JedisPoolConfig config = new JedisPoolConfig();
            RedisProperties.Pool props = this.properties.getPool();
            config.setMaxTotal(props.getMaxActive());
            config.setMaxIdle(props.getMaxIdle());
            config.setMinIdle(props.getMinIdle());
            config.setMaxWaitMillis(props.getMaxWait());
            return config;
        }
    }

    /**
     * Standard Redis configuration.
     */
    @Configuration
    protected class RedisConfiguration {

        @Bean
        @ConditionalOnMissingBean(name = "redisTemplate")
        public RedisTemplate<Object, Object> redisTemplate(
                RedisConnectionFactory redisConnectionFactory)
                throws UnknownHostException {
            RedisTemplate<Object, Object> template = new RedisTemplate<Object, Object>();
            template.setConnectionFactory(redisConnectionFactory);
            return template;
        }

        @Bean
        @ConditionalOnMissingBean(RedisCacheManager.class)
        public RedisCacheManager redisCacheManager() {
            return new RedisCacheManager();
        }
    }
}
```

#### ImportSelector

```java
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanClassLoaderAware;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.context.EnvironmentAware;
import org.springframework.context.ResourceLoaderAware;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DeferredImportSelector;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.io.support.SpringFactoriesLoader;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.util.Assert;

@Order(Ordered.LOWEST_PRECEDENCE - 1)
@Configuration
public class EnableAutoConfigurationImportSelector implements DeferredImportSelector,
      BeanClassLoaderAware, ResourceLoaderAware, BeanFactoryAware, EnvironmentAware {

   private ConfigurableListableBeanFactory beanFactory;

   private Environment environment;

   private ClassLoader beanClassLoader;

   private ResourceLoader resourceLoader;

   @Override
   public String[] selectImports(AnnotationMetadata metadata) {
      List<String> configurations = getCandidateConfigurations();
      configurations = removeDuplicates(configurations);
      return configurations.toArray(new String[configurations.size()]);
   }

   protected List<String> getCandidateConfigurations() {
      return SpringFactoriesLoader.loadFactoryNames(this.getClass(), getBeanClassLoader());
   }

   protected final <T> List<T> removeDuplicates(List<T> list) {
      return new ArrayList<T>(new LinkedHashSet<T>(list));
   }

   @Override
   public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
      Assert.isInstanceOf(ConfigurableListableBeanFactory.class, beanFactory);
      this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
   }

   @Override
   public void setBeanClassLoader(ClassLoader classLoader) {
      this.beanClassLoader = classLoader;
   }

   protected ClassLoader getBeanClassLoader() {
      return this.beanClassLoader;
   }

   @Override
   public void setEnvironment(Environment environment) {
      this.environment = environment;
   }

   @Override
   public void setResourceLoader(ResourceLoader resourceLoader) {
      this.resourceLoader = resourceLoader;
   }
}
```

#### ConditionalOnClass

```java
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.context.annotation.Conditional;

/**
 * {@link Conditional} that only matches when the specified classes are on the classpath.
 *
 * @author Phillip Webb
 */
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {

   /**
    * The classes that must be present. Since this annotation parsed by loading class
    * bytecode it is safe to specify classes here that may ultimately not be on the
    * classpath.
    * @return the classes that must be present
    */
   Class<?>[] value() default { };

   /**
    * The classes names that must be present.
    * @return the class names that must be present.
    */
   String[] name() default { };

}
```

#### OnClassCondition

```java
import java.util.Collections;
import java.util.Iterator;
import java.util.LinkedList;
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
import org.springframework.util.ClassUtils;
import org.springframework.util.MultiValueMap;
import org.springframework.util.StringUtils;

/**
 * {@link Condition} that checks for the presence or absence of specific classes.
 *
 * @author Phillip Webb
 * @see ConditionalOnClass
 * @see ConditionalOnMissingClass
 */
class OnClassCondition implements Condition {

    private Logger logger = LoggerFactory.getLogger(OnClassCondition.class);

    @Override
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {
        MultiValueMap<String, Object> onClasses = getAttributes(annotatedTypeMetadata,
                ConditionalOnClass.class);
        if (onClasses != null) {
            List<String> missing = getMatchingClasses(onClasses, MatchType.MISSING, conditionContext);
            if (!missing.isEmpty()) {

                logger.debug("required @ConditionalOnClass classes not found: {}",
                        StringUtils.collectionToCommaDelimitedString(missing));
                return false;
            }
            logger.debug("@ConditionalOnClass classes found: {}",
                    StringUtils.collectionToCommaDelimitedString(
                            getMatchingClasses(onClasses, MatchType.PRESENT, conditionContext)));
        }

        MultiValueMap<String, Object> onMissingClasses = getAttributes(annotatedTypeMetadata,
                ConditionalOnMissingClass.class);
        if (onMissingClasses != null) {
            List<String> present = getMatchingClasses(onMissingClasses, MatchType.PRESENT, conditionContext);
            if (!present.isEmpty()) {
                logger.debug("required @ConditionalOnMissing classes found: {}",
                        StringUtils.collectionToCommaDelimitedString(present));
                return false;
            }
            logger.debug("@ConditionalOnMissing classes not found: {}",
                    StringUtils.collectionToCommaDelimitedString(
                            getMatchingClasses(onMissingClasses, MatchType.MISSING, conditionContext)));
        }

        return true;
    }

    private MultiValueMap<String, Object> getAttributes(AnnotatedTypeMetadata metadata,
                                                        Class<?> annotationType) {
        return metadata.getAllAnnotationAttributes(annotationType.getName(), true);
    }

    private List<String> getMatchingClasses(MultiValueMap<String, Object> attributes,
                                            MatchType matchType, ConditionContext context) {
        List<String> matches = new LinkedList<String>();
        addAll(matches, attributes.get("value"));
        addAll(matches, attributes.get("name"));
        Iterator<String> iterator = matches.iterator();
        while (iterator.hasNext()) {
            if (!matchType.matches(iterator.next(), context)) {
                iterator.remove();
            }
        }
        return matches;
    }

    private void addAll(List<String> list, List<Object> itemsToAdd) {
        if (itemsToAdd != null) {
            for (Object item : itemsToAdd) {
                Collections.addAll(list, (String[]) item);
            }
        }
    }

    private enum MatchType {
        PRESENT {
            @Override
            public boolean matches(String className, ConditionContext context) {
                return ClassUtils.isPresent(className, context.getClassLoader());
            }
        },

        MISSING {
            @Override
            public boolean matches(String className, ConditionContext context) {
                return !ClassUtils.isPresent(className, context.getClassLoader());
            }
        };

        public abstract boolean matches(String className, ConditionContext context);
    }
}
```