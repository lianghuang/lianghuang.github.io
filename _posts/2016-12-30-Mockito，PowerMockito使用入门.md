---
layout: post
title: "Mockito,PowerMockito使用入门"
date: 2016-12-28 17:42:32+00:00
tags:
    - java
    - mockito
    - powermockito
author: "huangliangliang"
---

后续为了提升研发效率，解决项目中新引入代码的质量问题，开发必须采用单元测试的方式对自身实现的方法进行测试。保证验证通过。在单元测试中，为了解决spring的依赖问题，我们选用Mockito作为Mock测试框架。

首先需要了解一些基本概念：

1. 什么是Mocking

Mocking is a way to test the functionality of a class in isolation. 

Mocking不要求例如数据库连接，文件系统等工程启动时的初始化信息。只专注于测试某个方法内部逻辑。我们之前做一些service的测试，可能需要先加载spring的各种配置，连接好各种中间件等。这种重量级的测试对我们的单元测试非常不友好，而且容易导致脏数据的产生。针对于这些问题，我们使用mocking，对某些外部依赖的服务，类等，使用预先设置的值，而不是真实的加载他们。从而使得单元测试成为一个轻量级的过程。

2. Mockito框架

Mockito就是用来进行mocking的框架。具体介绍请参考 https://github.com/mockito/mockito

3. PowerMockito框架

Mockito不能针对static方法进行mock，但程序中很多地方使用了static的方法。为了解决这一问题，我们引入PowerMockito框架。

4. Mockito,PowerMockito引入

Mockito maven依赖：http://mvnrepository.com/artifact/org.mockito/mockito-all

最新稳定版本为1.10.19

```
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-all</artifactId>
    <version>1.10.19</version>
    <scope>test</scope>
</dependency>

```
配合Junit食用效果更佳


```
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>2.2.0</version>
    <scope>test</scope>
</dependency>
```
PowerMockito maven依赖：


```
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-api-mockito</artifactId>
    <version>1.6.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.powermock</groupId>
    <artifactId>powermock-module-junit4</artifactId>
    <version>1.6.2</version>
    <scope>test</scope>
</dependency>
```

5. 使用说明

参考教程

1：Mockito https://www.tutorialspoint.com/mockito/index.htm

2.PowerMockito https://github.com/powermock/powermock/wiki/MockitoUsage


mockito框架创建mock对象不能对final,static,Anonymous,primitive类进行mock。
PowerMockito框架可以对final static方法进行mock。

贴一段写的针对于模拟支付链接生成的代码作为案例讲述。
```

package com.chinamobile.bcop.test;

import static org.mockito.Mockito.when;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.powermock.api.mockito.PowerMockito;
import org.powermock.core.classloader.annotations.PrepareForTest;
import org.powermock.modules.junit4.PowerMockRunner;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.chinamobile.bcop.constants.ProductConstants;
import com.chinamobile.bcop.order.rest.service.JournalAccountService;
import com.chinamobile.bcop.order.rest.service.boss.BossService;
import com.chinamobile.bcop.order.utils.Utils;
import com.chinamobile.bcop.si.common.config.OPOrderManageConfig;
import com.chinamobile.bcop.si.common.config.OPOuterApiConfig;
import com.chinamobile.bcop.si.common.rest.client.UserManageService;
import com.chinamobile.bcop.user.usermgt.Customer;

/**
 * @author Huang, Liangliang
 *
 */
@RunWith(PowerMockRunner.class)
@PrepareForTest({OPOuterApiConfig.class,OPOrderManageConfig.class})
public class ChargeTest {
    @InjectMocks 
    private BossService bossService;
    @Mock
    private UserManageService userManageService;
    @Mock
    private JournalAccountService journalAccountService;
    
    private static Logger log = LoggerFactory.getLogger(ChargeTest.class);
    @Before
    public void setup() throws Exception {
        MockitoAnnotations.initMocks(this);
        PowerMockito.mockStatic(OPOuterApiConfig.class);
        PowerMockito.mockStatic(OPOrderManageConfig.class);
        PowerMockito.when(OPOrderManageConfig.getTestValue()).thenReturn(new Boolean(false));
        when(OPOrderManageConfig.getBossVmOsAvailableName()).thenReturn("linux,windows");
//        when(OPOrderManageConfig.getTestValue()).thenReturn(new Boolean(true));

        when(journalAccountService.journalCharge(userId,36480)).thenReturn(Utils.getUUID());
        Customer customer=new Customer();
        customer.setBossId("something");

    }

    @Test
    public void chargeTest() throws Exception {
        String notifyUrl=someurl;
        String returnUrl=someurl;
       String resource=bossService.generateChargeUrl(userId, 36480, ProductConstants.Order.ORDER_SOURCE_OP, notifyUrl, returnUrl);
       Mockito.verify(journalAccountService,Mockito.times(0)).getWithdraw("");
       log.info(resource);
       log.info("Test value:{}",OPOrderManageConfig.getTestValue());
    }
}

```

@RunWith(PowerMockRunner.class)标签：
当需要mock静态方法的时候，必须加注解@PrepareForTest和@RunWith。注解@PrepareForTest里写的类是静态方法所在的类。由PowerMockito提供。

当不需要对static final方法进行mock时，使用mockito提供的标签即可。
@RunWith(MockitoJUnitRunner.class)

@InjectMocks, @Mock标签

@Mock标签等同于Mockito.mock(classToMock);用来初始化一个mock对象。@InjectMocks 标签标注的类可以通过反射使用@Mock标签标注的类（自动注入）。当代码使用spring的标签，想要初始化mock类时，采用@InjectMocks, @Mock标签更为方便（不需要实现get set方法）。

MockitoAnnotations.initMocks(this);

初始化Mock标签

PowerMockito.mockStatic(OPOuterApiConfig.class);

Call PowerMockito.mockStatic() to mock a static class (use PowerMockito.spy(class) to mock a specific method):

when(OPOrderManageConfig.getBossVmOsAvailableName()).thenReturn("linux,windows");

Mockito的条件注入，指定mock类在什么情况下，返回什么值。

Mockito.verify(journalAccountService,Mockito.times(0)).getWithdraw("");

verify:Verify that certain methods from the mock object are called.用来验证这个方法是否被调用到。可以指定次数。

