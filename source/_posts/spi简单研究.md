---
title: spi简单研究
date: 2018-05-10 13:41:00
tags: [spi]
---

SPI是Service Provider Interfaces的简称。根据Java的SPI规范，我们可以定义一个服务接口，具体的实现由对应的实现者去提供，即Service Provider（服务提供者）。然后在使用的时候只要根据SPI的规范去获取对应的服务提供者的服务实现即可。   
SPI我的简单理解为服务提供方提供接口，调用方通过接口实现自己的扩展，然后修改服务提供方的jar包或者其他交付方式的META-INF/services中对应的接口配置指定实现类。   
SPI机制的约定：   
> * 在META-INF/services/目录中创建以接口全限定名命名的文件该文件内容为Api具体实现类的全限定名
> * 使用ServiceLoader类动态加载META-INF中的实现类    
> * 如SPI的实现类为Jar则需要放在主程序classPath中 

结构图如下：       
![](https://raw.githubusercontent.com/panhb/images/master/res/spi/structure.png)    

App.java   
```java
package spi;

/**
 * @author panhb
 *
 */
public interface App {

    String say(String msg) throws Exception;
}
```

AppImpl1.java
```java
package spi;

/**
 * @author panhb
 */
public class AppImpl1 implements App {

    @Override
    public String say(String msg) throws Exception {
        return "AppImpl1 " + msg;
    }

}
```
AppImpl2.java
```java
package spi;

/**
 * @author panhb
 */
public class AppImpl2 implements App {

    @Override
    public String say(String msg) throws Exception {
        return "AppImpl2 " + msg;
    }

}
```

SpiFactory.java
```java
package spi;

import java.util.Iterator;
import java.util.ServiceLoader;

/**
 * @author panhb
 */
public class SpiFactory<T> {

    private Class<T> domainClass;

    public SpiFactory(Class<T> domainClass) {
        this.domainClass = domainClass;
    }

    public T getSpi() throws Exception {
        return getSpi(domainClass);
    }

    public T getSpi(String impl) throws Exception {
        return getSpi(Class.forName(impl));
    }

    public T getSpi(Class<?> impl) throws Exception {
        ServiceLoader<T> s = ServiceLoader.load(domainClass);
        Iterator<T> iterator = s.iterator();
        while (iterator.hasNext()) {
            T t = iterator.next();
            if (impl.isInstance(t)) {
                return t;
            }
        }
        return null;
    }
}
```

AppImpl3.java
```java
package spitest;

import spi.App;

/**
 * @author panhb
 */
public class AppImpl3 implements App {

    @Override
    public String say(String msg) throws Exception {
        return "AppImpl3 " + msg;
    }

}
```

AppTest.java
```java
import spi.App;
import spi.AppImpl2;
import spi.SpiFactory;

/**
 * @author panhb
 */
public class AppTest {

    @Test
    public void sayTest() throws Exception {
        SpiFactory<App> spiFactory = new SpiFactory(App.class);
        App app = spiFactory.getSpi();
        String msg = app.say("hello world");
        System.out.println(msg);

        App app1 = spiFactory.getSpi(AppImpl2.class);
        String msg1 = app1.say("hello world");
        System.out.println(msg1);

        App app2 = spiFactory.getSpi("spi.AppImpl1");
        String msg2 = app2.say("hello world");
        System.out.println(msg2);
    }

}
```

META-INF/services目录中的文件：     
文件名：   
**spi.App**  
内容：     
**spitest.AppImpl3**   
**spi.AppImpl1**    
**spi.AppImpl2**    

运行结果：   
![](https://raw.githubusercontent.com/panhb/images/master/res/spi/result.png)   



