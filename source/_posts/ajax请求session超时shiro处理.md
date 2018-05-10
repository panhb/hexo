---
title: ajax请求session超时shiro处理
date: 2018-05-10 13:41:00
tags: [shiro]
---

问题产生背景：之前项目用过shiro做权限控制，新项目决定延用shiro做权限控制，但是新项目采用前后分离，session超时后，需要返回json结构给前端去处理，shiro默认处理方案是session超时后重定向到登陆链接，修改如下   

### 1.创建SessionExpiredFilter  
```java
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.web.filter.authc.FormAuthenticationFilter;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import java.io.IOException;
import java.io.PrintWriter;

/**
 * @author panhb
 */
@Slf4j
public class SessionExpiredFilter extends FormAuthenticationFilter {

	@Override
	protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
		if (isLoginRequest(request, response)) {
			if (isLoginSubmission(request, response)) {
				return executeLogin(request, response);
			} else {
				return true;
			}
		} else {
			//原处理逻辑
//			saveRequestAndRedirectToLogin(request, response);
            response.setContentType("application/json;charset=utf-8");
            try (PrintWriter pw = response.getWriter()) {
                pw.write("{\"code\":17918}");
                pw.flush();
            } catch (IOException e) {
                log.error(e.getMessage(), e);
            }
			return false;
		}
	}

}
```   

### 2.将创建的SessionExpiredFilter放入ShiroFilterFactoryBean   
```java
ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
Map<String, Filter> filters = new LinkedHashMap<>();
SessionExpiredFilter sessionExpiredFilter = new SessionExpiredFilter();
filters.put("authc", sessionExpiredFilter);
shiroFilterFactoryBean.setFilters(filters);

Map<String, String> chains = new LinkedHashMap<>();
chains.put("/**", "authc");
shiroFilterFactoryBean.setFilterChainDefinitionMap(chains);
```