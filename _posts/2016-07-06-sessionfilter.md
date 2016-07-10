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
comments: false
share: false
---

# This filter is made to make sure a  request is legal. 

## It's a  manage system in the project,when a client send a request to server,the server need to detect if the request includes  legal session.

 
### Firstly,add properties in web.xml.


```xml
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
    filter mapping also can be multiple-->
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
<span></span>

> If the photos are **not** included in the `div` the text will float on the right.
