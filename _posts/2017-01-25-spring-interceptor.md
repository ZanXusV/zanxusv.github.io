---
layout:     post
title:      "Spring interceptor implemention"
subtitle:   "Signature interceptor"
date:       2017-01-25 10:50:00+0800
author:     "ZanXus"
header-img: "img/blog/header/post-bg-08.jpg"
thumbnail: /img/blog/thumbs/thumb08.png
tags: [springmvc,interceptor]
category: [java]
comments: true
share: true
---

## Interceptor is used here to validate api signature when app request server.

## Spring HandlerInterceptor interfaces declares three methods based on where we want to intercept the HTTP request,

1. **boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)**: This method is used to intercept the request before it’s handed over to the handler method. This method should return ‘true’ to let Spring know to process the request through another spring interceptor or to send it to handler method if there are no further spring interceptors.
If this method returns ‘false’ Spring framework assumes that request has been handled by the spring interceptor itself and no further processing is needed. We should use response object to send response to the client request in this case.

2. **void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)**: This HandlerInterceptor interceptor method is called when HandlerAdapter has invoked the handler but DispatcherServlet is yet to render the view. This method can be used to add additional attribute to the ModelAndView object to be used in the view pages. We can use this spring interceptor method to determine the time taken by handler method to process the client request.

3. **void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)**: This is a HandlerInterceptor callback method that is called once the handler is executed and view is rendered.

## Here i extend `HandlerInterceptorAdapter` to implement the `preHandle()` method,besides above all three methods,there's another method **void afterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response, Object handler)** for implementing async request.But in my case there is only `preHandle()` method useful.

## In my case,this interceptor just need to intercept `POST` requests,all of the api are RESTful so there's only  path variables and  json body,no query string,so i need to get these params from `HttpServletRequest`.

## Get path variables:

```java
Map<String,Object> pathVariables = (HashMap) request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);
```

## Get json body:

```java
String body=request.getReader().lines().collect(Collectors.joining());
```

## However,i encountered an exception  **java.lang.IllegalStateException: getReader() has already been called for this request** when i get json body in this way,because the `request.getReader()` can only read data once,it can not be read when handler step into controller since it had already been read in the interceptor.

## So i need to wrap a `HttpServletRequestWrapper`:

```java
import javax.servlet.ReadListener;
import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.stream.Collectors;

public class MultiReadRequest extends HttpServletRequestWrapper{

    private String requestBody;

    public MultiReadRequest(HttpServletRequest request) {
        super(request);
        try {
            requestBody = request.getReader().lines().collect(Collectors.joining(System.lineSeparator()));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(requestBody.getBytes());
        return new ServletInputStream() {
            @Override
            public boolean isFinished() {
                return byteArrayInputStream.available()==0;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {

            }

            @Override
            public int read() throws IOException {
                return byteArrayInputStream.read();
            }
        };
    }

    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(this.getInputStream(), Charset.forName("UTF-8")));
    }
}
```

## Then config interceptor in spring configuration file.
```xml
<mvc:interceptors>
		 <mvc:interceptor>
			 <mvc:mapping path="/**"/>
			 <bean class="com.ucf.staging.interceptor.SignatureInterceptor"></bean>
		 </mvc:interceptor>
	</mvc:interceptors>
```

## In fact,spring interceptor is similar to filter,[Spring Doc](http://docs.spring.io/spring/docs/3.0.x/javadoc-api/org/springframework/web/servlet/HandlerInterceptor.html) described as follows:

>Typically an interceptor chain is defined per HandlerMapping bean, sharing its granularity. To be able to apply a certain interceptor chain to a group of handlers, one needs to map the desired handlers via one HandlerMapping bean. The interceptors themselves are defined as beans in the application context, referenced by the mapping bean definition via its "interceptors" property (in XML: a <list> of <ref>).

>HandlerInterceptor is basically similar to a Servlet 2.3 Filter, but in contrast to the latter it just allows custom pre-processing with the option of prohibiting the execution of the handler itself, and custom post-processing. Filters are more powerful, for example they allow for exchanging the request and response objects that are handed down the chain. Note that a filter gets configured in web.xml, a HandlerInterceptor in the application context.

>As a basic guideline, fine-grained handler-related preprocessing tasks are candidates for HandlerInterceptor implementations, especially factored-out common handler code and authorization checks. On the other hand, a Filter is well-suited for request content and view content handling, like multipart forms and GZIP compression. This typically shows when one needs to map the filter to certain content types (e.g. images), or to all requests.

## On the other hand,filter can be used only under servlet container. Whereas handlers can be invoked within Spring container,not necessarily in the web environment.Besides,interceptors can be invoked before or after the requests processing by controller, and also after rendering action's view to the user while filters can be applied only before returning the response to the final user.




















