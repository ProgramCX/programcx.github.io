---
title: 在Spring Boot中使用定时任务
description: Spring Boot 中使用 Quartz 实现定时任务
date: 2025-02-22
timezone: UTC+8
image: springboot.png
slug: springboot-quartz-simple-job
categories:
    - Java
    - Spring Boot
---

# 在Spring Boot中实现简单定时任务

**本文主要介绍在Spring Boot中使用Quartz实现定时任务的方法。**

## 1. 添加依赖

在`pom.xml`文件中添加Quartz的依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

## 2. 编写定时任务代码

1. 创建一个继承`QuartzJobBean`的类，实现`executeInternal`方法，编写定时任务的逻辑。

```java
package cn.programcx.springbootinit.jobs;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.quartz.*;

import java.util.Date;

public class MyJob extends QuartzJobBean {

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        Logger logger = LoggerFactory.getLogger(getClass());
        logger.info(new Date().toString()); // 打印当前时间
    }
}
```

2.在容器中注册JobDetail和Trigger，并将其与MyJob关联。

```java
package cn.programcx.springbootinit.config;


import cn.programcx.springbootinit.jobs.MyJob;
import org.quartz.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CustomConfiguration {
    @Bean
    public JobDetail printTimeJobDetail() {
        // 指定具体的Job类
        return JobBuilder.newJob(MyJob.class).withIdentity("MyJobDetail").storeDurably().build();
    }

    @Bean
    public Trigger printTimeJobtrigger(){
        // 指定具体的触发器
        CronScheduleBuilder builder = CronScheduleBuilder.cronSchedule("0/1 * * * * ?"); // 每秒执行一次

        // 指定触发器关联的JobDetail
        return (Trigger) TriggerBuilder.newTrigger().forJob("MyJobDetail").withIdentity("quartzTaskService").withSchedule(builder).build();
    }
}

```

## 3. 启动定时任务

在Spring Boot的启动类中添加`@EnableScheduling`注解，启用定时任务。

```java
package cn.programcx.springbootinit;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class SpringBootInitApplication {

    public static void main(String[] args) {
        // 启动Spring Boot应用
        SpringApplication.run(SpringBootInitApplication.class, args);
    }

}
```


## 4. 测试

启动Spring Boot应用，可以看到控制台每秒输出一次当前时间。

![控制台输出](image.png)





