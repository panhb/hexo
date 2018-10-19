---
title: AOP自定义注解浅析
date: 2018-10-19 12:00:00
tags: [AOP,自定义注解,spring]
---

spring aop我的理解是通过代理类执行切面方法，其中代理方式有两种，一种是jdk动态代理，一种是cglib动态代理（通过修改字节码来实现代理）。当需要被代理的类有接口时，会采用jdk动态代理实现接口。如果没有接口时，会采用cglib动态代理生成一个子类。通常aop会配合自定义注解实现某些功能，比如日志等。昨天我实现了一个自定义注解，将参数值自动赋值到要返回的bean中，具体实现如下   
自定义注解ArgToFiled   
```java
package com.panhb.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author hongbo.pan
 * @date 2018/10/18
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ArgToFiled {

    /**
     * 方法中的参数名
     */
    String[] argNames() default {"status"};

    /**
     * 返回VO中需要赋值的字段名
     */
    String[] filedNames() default {"status"};

}
```

实现注解的切面类ArgToFiledAspect    
```java
package com.panhb.aspect;

import com.panhb.annotation.ArgToFiled;
import org.apache.commons.lang3.ArrayUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

/**
 * @author hongbo.pan
 * @date 2018/10/18
 */
@Aspect
@Component
public class ArgToFiledAspect {

    /**
     * 切点
     *
     * @param argToFiled
     */
    @Pointcut(value = "@annotation(argToFiled)")
    public void pointcut(ArgToFiled argToFiled) {
    }

    /**
     * 环绕
     *
     * @param pjp
     * @param argToFiled
     * @return
     * @throws Throwable
     */
    @Around("pointcut(argToFiled)")
    public Object around(ProceedingJoinPoint pjp, ArgToFiled argToFiled) throws Throwable {
        Object[] args = pjp.getArgs();
        Object voObject =  pjp.proceed(args);
        String[] argNames = argToFiled.argNames();
        String[] filedNames = argToFiled.filedNames();
        if (ArrayUtils.isNotEmpty(argNames) && ArrayUtils.isNotEmpty(filedNames)) {
            voObject = setFiledsValue(pjp, voObject, argNames, filedNames);
        }
        return voObject;
    }

    /**
     * 字段赋值
     *
     * @param pjp
     * @param voObject
     * @param argNames
     * @param filedNames
     * @return
     */
    private Object setFiledsValue(ProceedingJoinPoint pjp, Object voObject, String[] argNames, String[] filedNames) {
        Object[] args = pjp.getArgs();
        //获取方法参数
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        //获取方法参数名
        String[] paramNames = signature.getParameterNames();
        //获取方法参数类型
        Class[] classes = signature.getParameterTypes();
        for (int i = 0; i < argNames.length; i++) {
            //获取参数名的位置
            int paramIndex = ArrayUtils.indexOf(paramNames, argNames[i]);
            if (paramIndex > -1) {
                Object paramValue = args[paramIndex];
                if (paramValue != null) {
                    voObject = invokeSetMethodIgnoreException(voObject, classes[i], filedNames[i], paramValue);
                }
            }
        }
        return voObject;
    }

    private Object invokeSetMethodIgnoreException(Object voObject, Class paramType, String filedName, Object filedValue) {
        try {
            voObject = invokeSetMethod(voObject, paramType, filedName, filedValue);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return voObject;
    }

    /**
     * 反射调用set方法
     *
     * @param object
     * @param paramType 参数类型
     * @param filedName 参数名
     * @param filedValue 参数值
     * @return
     * @throws NoSuchMethodException
     * @throws InvocationTargetException
     * @throws IllegalAccessException
     */
    private Object invokeSetMethod(Object object, Class paramType, String filedName, Object filedValue) throws
            NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Class<?> clazz = object.getClass();
        String setMethodName = "set" +
                filedName.substring(0, 1).toUpperCase() + filedName.substring(1);
        Method method = clazz.getMethod(setMethodName, paramType);
        if (method != null) {
            method.invoke(object, filedValue);
        }
        return object;
    }

}
```

测试一下，首先定义一个TestVO
```java
package com.panhb.model;

import com.panhb.model.base.BaseModel;
import lombok.Data;

/**
 * @author hongbo.pan
 * @date 2018/10/19
 */
@Data
public class TestVO extends BaseModel {

    private String status;

}
```

创建一个测试controller
```java
package com.panhb.controller;

import com.panhb.annotation.ArgToFiled;
import com.panhb.controller.base.BaseController;
import com.panhb.model.Result;
import com.panhb.model.TestVO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author hongbo.pan
 * @date 2018/10/19
 */
@RestController
@Slf4j
public class TestController extends BaseController {

    @RequestMapping("/test")
    public Result<TestVO> test() {
        TestVO testVO = testArgToFiled("teststatus");
        return Result.succData(testVO);
    }

    @ArgToFiled
    public TestVO testArgToFiled(String status) {
        return new TestVO();
    }

}
```

启动后用postman模拟发送一个请求看看结果    
```java
{
    "code": 0,
    "data": {
        "status": null
    },
    "msg": "操作成功"
}
```

发现status并没有设置到vo中去，注解没有生效。后来找到原因，**同一个类中的方法调用，不是通过代理类去调用的**，导致注解没有生效。     
有两种解决办法   
1.将方法放到另一个类（需要spring注入）中调用    
2.同类中这样调用((XXX)AopContext.currentProxy()).xxx();     
按照第二种方式修改后    
```java
package com.panhb.controller;

import com.panhb.annotation.ArgToFiled;
import com.panhb.controller.base.BaseController;
import com.panhb.model.Result;
import com.panhb.model.TestVO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.aop.framework.AopContext;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author hongbo.pan
 * @date 2018/10/19
 */
@RestController
@Slf4j
public class TestController extends BaseController {

    @RequestMapping("/test")
    public Result<TestVO> test() {
        TestVO testVO =
                ((TestController)AopContext.currentProxy()).testArgToFiled("teststatus");
        return Result.succData(testVO);
    }

    @ArgToFiled
    public TestVO testArgToFiled(String status) {
        return new TestVO();
    }

}
```

发现后台报错   
```java   
java.lang.IllegalStateException: Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available. 
```

这里需要配置一下aop    
```java   
声明式
@EnableAspectJAutoProxy(exposeProxy = true)   

xml    
<aop:config expose-proxy="true" />
或者
<aop:aspectj-autoproxy expose-proxy="true"/>
```

重新发送模拟请求后返回结果    
```java
{
    "code": 0,
    "data": {
        "status": "teststatus"
    },
    "msg": "操作成功"
}
```
注解生效，参数值设置成功。