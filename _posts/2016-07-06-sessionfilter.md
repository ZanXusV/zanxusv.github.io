---
layout:     post
title:      "SessionFilter"
subtitle:   "Java login session filter"
date:       2016-07-06 18:54:00
author:     "ZanXus"
header-img: "img/blog/header/post-bg-01.jpg"
thumbnail: /img/blog/thumbs/thumb01.png
tags: [session,filter]
category: [java]
comments: true
share: true
---

# This filter is made to make sure a  request is legal. 

## It's a  manage system in the project,when a client send a request to server,the server need to detect if the request includes  legal session.

 
> ### Firstly,add properties in **_web.xml_**{: style="color: red"}.

~~~ xml
<filter>
    <filter-name>SessionFilter</filter-name>
    <filter-class>com.ucf.filter.SessionFilter
    </filter-class>
    <init-param>
        <description>
        The url that will redirect to if session not detected.
        And if this param not configed,it will redirect to 
        the root path(/) of the web application. 
        </description>
        <param-name>redirectUrl</param-name>
        <param-value>/login</param-value>
    </init-param>
    <init-param>
        <description>
        The regex of url that won't be intercepted.The value 
        of redirectUrl does not need to add to the regex 
        because the redirectUrl will not be intercepted,besides 
        the contextPath of the web application is not included.
        </description>
        <param-name>excepUrlRegex</param-name>
        <!--the requestmapping /loginPost and /registPost 
        will not be intercept, and the <param-value>
         can be multiple to match more request url-->
        <param-value>/(login|regist)Post/</param-value>
    </init-param>
</filter>

<filter-mapping>
    <filter-name>SessionFilter</filter-name>
    <!--all url under the url-pattern will be filtered,and the 
    filter mapping can also be multiple-->
    <url-pattern>/*</url-pattern>
</filter-mapping>
~~~
<p></p>

> ### Secondly,Create the **_SessionFilter_**{: style="color: red"} class.

~~~ java
import java.io.IOException;
import java.util.regex.Pattern;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import com.ucf.staging.model.commodity.vo.CommodityOrgVo;
import org.apache.commons.lang3.StringUtils;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.ucf.staging.common.RedisUtils;
import com.ucf.staging.consts.GlobalConsts;
import com.ucf.staging.model.commodity.CommodityOrg;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SessionFilter implements Filter {

    private String redirectUrl;

    private Pattern excepUrlPattern;

    private static final Logger logger= LoggerFactory.getLogger(MerchantPcSessionFilter.class);

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        //get values of init params in web.xml
        redirectUrl = filterConfig.getInitParameter("redirectUrl");
        String excepUrlRegex = filterConfig.getInitParameter("excepUrlRegex");
        if (StringUtils.isNotBlank(excepUrlRegex)) {
            excepUrlPattern = Pattern.compile(excepUrlRegex);
        }
    }
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        // * while requesting http://127.0.0.1:8080/webApp/home.jsp?&a=1&b=2 
        // * request.getRequestURL()： http://127.0.0.1:8080/webApp/home.jsp
        // * request.getContextPath()： /webApp
        // * request.getServletPath()：/home.jsp
        // * request.getRequestURI()： /webApp/home.jsp
        // * request.getQueryString()：a=1&b=2

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;
        String requestURI = request.getRequestURI();
        
        //if a requst uri matches the excepetUrl pattern then let it through 
        if (excepUrlPattern.matcher(requestURI).matches()) {
                chain.doFilter(req, res); 
                return; 
         }
         
        /*sometimes the contextpath will added in the web server instead of  project,and the request will be transfered by nginx so the request uri will 
         perform as no contextpath included in browser while the web server does have the contextpath,in such situation the above code need to be replaced 
         with the follow or replace matches() with find().
        if (requestURI.indexOf(RequestMappingWithoutContextPath) >=0) {
            chain.doFilter(req, res);
            return;
        }*/
        
        //get sessionId form cookie
        Cookie[] cookies = request.getCookies();
        String sessionId = null;
        if (cookies==null){
            response.sendRedirect(redirectUrl);
            return;
        }
        for (Cookie cookie : cookies) {
            //the cookie name was set as "merSessionId" when logged in successfully,and the cookie value was used setting as key in redis.
            if ("merSessionId".equals(cookie.getName())) {
                sessionId = cookie.getValue();
            }
        }
        if (sessionId != null) {
            final String key = sessionId;
            String session = RedisUtils.executeResult((x) -> x.get(key), GlobalConsts.REDIS_INDEX);
            logger.info("session content"+session);
            if (StringUtils.isBlank(session)) {
                response.sendRedirect(redirectUrl);
                return;
            } else {
                CommodityOrgVo orgVo = JSON.toJavaObject(JSONObject.parseObject(session), CommodityOrgVo.class);
                request.setAttribute("merchantSession", orgVo);
                RedisUtils.execute((x) -> x.expire(key, GlobalConsts.REDIS_LIVETIME), GlobalConsts.REDIS_INDEX);
                chain.doFilter(req, res);
            }
        } else {
            response.sendRedirect(redirectUrl);
        }
    }

    @Override
    public void destroy() {

    }

}
~~~


> ### Here's the login code segment.

~~~ java
    String sessionId = DigestUtils.md5Hex(orgId + password);
    CommodityOrgVo orgVo = orgService.toCorgVo(org);
    RedisUtils.execute((x) -> x.set(sessionId, JSON.toJSONString(orgVo)),GlobalConsts.REDIS_INDEX);//set session to redis
    RedisUtils.execute((x) -> x.expire(sessionId, GlobalConsts.REDIS_LIVETIME),GlobalConsts.REDIS_INDEX);//set session expire
    Cookie cookie = new Cookie("merSessionId", sessionId);//create cookie 
    String requestUrl=request.getRequestURL().toString();
    //if it's a domain request not a ip request,set cookie's domain as main domain in order to make the cookie cross domain.
    String domain=MethodUtil.getDomain(requestUrl);
    if (StringUtils.isNotBlank(domain)){
        cookie.setDomain(domain);
    }
    cookie.setHttpOnly(true);
    cookie.setMaxAge(GlobalConsts.COOKIE_LIVETIME);
    cookie.setPath("/");
    response.addCookie(cookie);
~~~

> ### Here's the RedisUtil.

~~~ java
package com.ucf.staging.common;

import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.function.Consumer;
import java.util.function.Function;

import org.apache.commons.configuration.Configuration;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import com.webapp.utils.config.ConfigUtils;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisSentinelPool;
import redis.clients.util.Pool;

@Component
public final class RedisUtils {

	private static final Logger logger = LoggerFactory.getLogger(RedisUtils.class);
	private static final String REDIS_MODE = "redis.mode";
	private static final String MODE_SHARDED = "sharded";
	private static final String MODE_SENTINEL = "sentinel";
	private static final String REDIS_TIMEOUT = "redis.timeout";
	private static final String REDIS_PWD = "redis.password";
	private static final String REDIS_IP = "redis.ip";
	private static final String REDIS_PORT = "redis.port";
	private static final String REDIS_MASTER = "redis.master";
	private static final String REDIS_SENTINELS = "redis.sentinels";

	private static String redisCfg;
	@Value("${redisCfg}")
	public void setRedisCfg(String redisCfg) {
		RedisUtils.redisCfg = redisCfg;
	}
	private static Pool<Jedis> pool = null;

	private RedisUtils() {}
	private static Pool<Jedis> getPool() {
		if (pool == null) {
			Configuration config = ConfigUtils.addConfig(redisCfg);
			JedisPoolConfig poolCfg = RedisCfg.getPoolCfg(config);
			String mode = config.getString(REDIS_MODE);
			Integer timeout = config.getInteger(REDIS_TIMEOUT, null);
			String password = config.getString(REDIS_PWD, null);
			if(mode.equals(MODE_SHARDED)){
				logger.error(MODE_SHARDED + " Not implemented");
			}else if(mode.equals(MODE_SENTINEL)){
				String master = config.getString(REDIS_MASTER);
				List<Object> stnList = config.getList(REDIS_SENTINELS);
				Set<String> stns = new HashSet<>();
				stnList.forEach(s->stns.add(s.toString()));
				pool = new JedisSentinelPool(master, stns, poolCfg, timeout, password);
			}else{
				String ip = config.getString(REDIS_IP);
				Integer port = config.getInteger(REDIS_PORT, null);
				pool = new JedisPool(poolCfg, ip, port, timeout, password);
			}
		}
		return pool;
	}
	
	public static Jedis getJedis() {
		return getPool().getResource();
	}

	public static Jedis getJedis(int index) {
		Jedis jedis = getJedis();
		jedis.select(index);
		return jedis;
	}

	public static void execute(Consumer<Jedis> consumer) {
		execute(consumer, 0);
	}
	public static void execute(Consumer<Jedis> consumer, int index) {
		Jedis jedis = getJedis(index);
		try {
			consumer.accept(jedis);
		} catch (Exception e) {
		    logger.error("Class [{}]call[{}]param[{}]error", "RedisUtils", "execute",
	                    String.format("consumer:%s,  index:%s", consumer.toString(), Integer.toString(index)), e);
		} finally {
			closeJedis(jedis);
		}
	}
	public static <R> R executeResult(Function<Jedis, R> function) {
		return executeResult(function, 0);
	}
	public static <R> R executeResult(Function<Jedis, R> function, int index) {
		R apply = null;
		Jedis jedis = getJedis(index);
		try {
			apply = function.apply(jedis);
		} catch (Exception e) {
		    logger.error("Class [{}]call[{}]param[{}]error", "RedisUtils", "executeResult",
                            String.format("function:%s,  index:%s", function.toString(), Integer.toString(index)), e);
		} finally {
			closeJedis(jedis);
		}
		return apply;
	}

	private static void closeJedis(Jedis jedis){
		if(jedis != null)
			jedis.close();
	}

}
~~~

> ## Finally,the above codes are neither complete or executable,some sensitive codes aren't included in.It's just a reference for such a function and mostly for my learning and working record :). 

