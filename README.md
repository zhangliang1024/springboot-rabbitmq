# SpringBoot+RabbitMQ 项目整合中遇到的问题
> SpringBoot+RabbitMQ 演示消息发送+手动ACK+失败重试机制

### 一、自定义构建线程池
> TaskScheduler 任务调度器: ThreadPoolTaskScheduler是一个专门用于调度任务的类
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import java.util.concurrent.ThreadPoolExecutor;

@Configuration
@EnableScheduling
public class TaskSchedulerConfig implements AsyncConfigurer,SchedulingConfigurer {
 
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskScheduler());
    }
     
    @Bean
    public ThreadPoolTaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler  executor = new ThreadPoolTaskScheduler ();
        executor.setPoolSize(10);
        executor.setThreadNamePrefix("MyExecutor-");
        // rejection-policy：当pool已经达到max size的时候，如何处理新任务
        // CALLER_RUNS：不在新线程中执行任务，而是由调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
 
}
```

> 报错：java.lang.IllegalArgumentException: Unsupported scheduler type: class org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;

@Configuration
@EnableScheduling
public class TaskSchedulerConfig implements AsyncConfigurer,SchedulingConfigurer {
 
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskScheduler());
    }
     
    /**
     * ThreadPoolTaskExecutor是一个专门用于执行任务的类
    @Bean
    public Executor taskScheduler() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setKeepAliveSeconds(3000);
        executor.setThreadNamePrefix("MyExecutor-");

        // rejection-policy：当pool已经达到max size的时候，如何处理新任务
        // CALLER_RUNS：不在新线程中执行任务，而是由调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
 
}
```

### 二、自定义mapper文件的扫描
> `classpath*` ：不仅包含class路径，还包括jar文件中(class路径)进行查找: `classpath*:com/cj/springboot/mapping/*.xml`
```java
SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
bean.setDataSource(dataSource);
// 添加XML目录
ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
try {
    bean.setMapperLocations(resolver.getResources("classpath*:com/cj/springboot/mapping/*.xml"));
    SqlSessionFactory sqlSessionFactory = bean.getObject();
    sqlSessionFactory.getConfiguration().setCacheEnabled(Boolean.TRUE);
    return sqlSessionFactory;
} catch (Exception e) {
    throw new RuntimeException(e);
}
```
> [解决src/main/java目录下mapper.xml文件不被扫描的问题](https://blog.csdn.net/sdx15701674113/article/details/73330401)
```xml
<build>
<resources>
    <resource>
        <directory>src/main/resources</directory>
    </resource>
    <resource>
        <directory>src/main/java</directory>
        <includes>
            <include>**/*.xml</include>
        </includes>
        <filtering>false</filtering>
    </resource>
</resources>
<plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
</plugins>
</build>
```

### 三、数据库连接
> 1.数据库连接时区的选择不同有时会导致启动报错
>
> 2.`5.+`的数据库版本要求SSL连接，如果没有加：useSSL=false
> 
> Thu Jul 09 15:03:35 CST 2020 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
> 
> spring.datasource.url=jdbc:mysql:///test?characterEncoding=UTF-8&serverTimezone=UTC&useSSL=false&autoReconnect=true
> 
> spring.datasource.url=jdbc:mysql:///test?characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true

### 四、RabbitMQ连接
> 错误使用连接：spring.rabbitmq.address=192.168.80.130:5672
```yaml
spring:
  rabbitmq:
    addresses: 192.168.80.130:5672
```


