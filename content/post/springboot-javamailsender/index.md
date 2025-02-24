---
title: 在Spring Boot中使用 JavaMailSender
description: Spring Boot 中使用 JavaMailSender 发送邮件
date: 2025-02-24
timezone: UTC+8
image: springboot.png
slug: springboot-javamailsender
categories:
    - Java
    - Spring Boot
---

# 在Spring Boot中实现邮件发送

**本文主要通过部分简单的发送验证码的例子来介绍在Spring Boot中实现邮件发送。**

## 1. 添加依赖

在`pom.xml`文件中添加JavaMailSender的依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

## 2. 在`application.yaml`中配置邮件发送信息

```yaml
spring:
  mail:
    host: smtp.qq.com
    port: 465
    username: xxx@qq.com
    password: xxxxxxxxx # 授权码
    # 下方的属性必须带着，不然会报错
    properties:
      mail:
        smtp:
          auth: true
          socketFactory:
            class: javax.net.ssl.SSLSocketFactory
          starttls:
           enable: true
```

## 3.创建Controller

```java
package cn.programcx.springbootinit.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;

import cn.programcx.springbootinit.model.Tester;

import cn.programcx.springbootinit.services.*;
@Controller
public class TestController {
    @Autowired
    private VerificationService service;

//当用户点击发送验证码时，调用此方法
    @PostMapping("verifycallback")
    @ResponseBody
    public  void sendVerifyCode(@RequestParam("email") String email){
        service.sendVerificationCode(email);
    }
}

## 4.创建Service

```java
package cn.programcx.springbootinit.services;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

import java.util.Random;

@Service
public class VerificationService {
    @Value("${spring.mail.username}")
    private String userName;

    @Value("${spring.mail.host}")
    private String host;

    @Value("${spring.mail.password}")
    private String password;

    @Value("${spring.mail.port}")
    private  String port;

    @Autowired
    private JavaMailSender mailSender;

    public void sendVerificationCode(String toAddress){
        Random random =new Random(System.currentTimeMillis());
        int code = random.nextInt((999999 - 100000) + 1);
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(userName);
        message.setTo(toAddress);
        message.setSubject("验证码登录");
        message.setText("欢迎注册程旭工作室官网，验证码: "+Integer.toString(code)+" 10分钟内有效.请勿泄露。注册后您将成为程旭工作室终身SSSSSVIP。");
        Logger logger= LoggerFactory.getLogger(getClass());
        try {
            mailSender.send(message);

            logger.info("已发送验证码"+Integer.toString(code)+"邮件地址"+toAddress);

        }
        catch (Exception e){
            logger.error(e.toString());
            logger.error("未成功验证码"+Integer.toString(code)+"邮件地址"+toAddress,e.getMessage());
        }
    }

}
```

