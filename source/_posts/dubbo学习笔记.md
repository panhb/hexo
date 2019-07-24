---
title: Dubbo学习笔记
date: 2016-02-04 12:00:00
tags: [dubbo,java,学习笔记]
---

今年呢，也没干什么大事，就做了一点微小的工作，做了一个dubbo的监控平台，专门监视使用了dubbo框架的系统。这个项目本身也在使用dubbo，主要是在使用dubbo框架的项目里加上我们的filter，通过filter获取到监控数据，以及访问控制。那说了半天，dubbo是个什么鬼呢。dubbo是阿里巴巴的一个开源的分布式服务框架，致力于提供高性能和透明化的**RPC**远程服务调用方案（以上摘抄自[*DUBBO*](http://dubbo.io/)）。github地址是[*GITHUB*](https://github.com/dubbo)。其他的我就不多说了，具体说说dubbo的泛化调用。      
<!-- more -->         
起因很简单，我们奇葩的架构师（可以这么叫吧，其实我更想叫他项目经理）要我们开发一个通用的consumer端调用。那么这个通用的consumer怎么做呢，如果是HTTP的话，无非就是你把调用的地址和参数告诉我，我发一个http请求，获取到对应的数据，返回给你就完了咯。但是这个RPC就比较蛋疼了，木有调用地址，而是要实例化对应的服务，然后调用服务的方法，方法参数也要实例化，那么我不可能把所有的服务都引入到我们的项目中，毕竟这样不太现实，工作量也太大。**当你没什么头绪的时候，看官方文档是最吼的。**我就去dubbo的官网看api文档，翻着翻着，突然看到了这个[*泛化引用*](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E6%B3%9B%E5%8C%96%E5%BC%95%E7%94%A8)，这尼玛心里那个激动啊，赶紧写了段测试代码压压惊。具体代码如下         
    
```       
     
import java.io.IOException;
import java.util.List;
import java.util.Map;
import java.util.Properties;

import com.alibaba.dubbo.config.ApplicationConfig;
import com.alibaba.dubbo.config.ReferenceConfig;
import com.alibaba.dubbo.config.RegistryConfig;
import com.alibaba.dubbo.rpc.service.GenericService;

public class ServiceRefFactory {
	private ApplicationConfig application;
	private RegistryConfig registry;
	private static String ADDRESS;	
	static {
		Properties prop = new Properties();
		ClassLoader loader =ServiceRefFactory.class.getClassLoader();
		try {
			prop.load(loader.getResourceAsStream("dubbo.properties"));
		} catch (IOException e) {
			e.printStackTrace();
		}
		ADDRESS = prop.getProperty("dubbo.registry.address");
	}	
	public synchronized static ServiceRefFactory getInstance(String applicationName){
		ApplicationConfig application = new ApplicationConfig();
		application.setName(applicationName);
		RegistryConfig registry = new RegistryConfig();
		registry.setAddress(ADDRESS);
		return new ServiceRefFactory(application, registry);
	}	
	private ServiceRefFactory(ApplicationConfig application, RegistryConfig registry){
		this.application = application;
		this.registry = registry;
	}
	/*
	 * dobbo泛化引用实现
	 * interfaceClass  接口服务
	 * methodName  方法名
	 * parameters 参数
	 */
	public Object applyServiceRef(String interfaceClass,String methodName,List<Map<String,Object>> parameters){
		ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>(); // 此实例很重，封装了与注册中心的连接以及与提供者的连接，请自行缓存，否则可能造成内存和连接泄漏
		reference.setApplication(application);
		reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
		reference.setInterface(interfaceClass); // 弱类型接口名 
		reference.setGeneric(true); // 声明为泛化接口 
		
		GenericService genericService = reference.get(); // 用com.alibaba.dubbo.rpc.service.GenericService可以替代所有接口引用 
		int length = parameters.size();
		String[] classStr = new String[length];  
		Object[] objStr = new Object[length];
		for(int i=0;i<parameters.size();i++){
			classStr[i] = parameters.get(i).get("ParamType")+"";//参数类型
			objStr[i] = parameters.get(i).get("Object");//参数值
		}
		Object result = genericService.$invoke(methodName, classStr , objStr);
		return result;
	}
}
```          

先写这么多吧，其他什么provider怎么配置啊，consumer怎么配置啊，官方文档都写的很清楚，我这里写了也没啥意义，以上代码已在生产中运行，但是**仅供参考，慨不负责，蛤蛤**
              

