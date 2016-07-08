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
    <filter-name>merSessionFilter</filter-name>
    <filter-class>com.ucf.staging.filter.MerchantPcSessionFilter
    </filter-class>
    <init-param>
        <description>
        If this param not configed,it will redirect to the root path(/) of the 
        web application when user not loged in. 
        </description>
        <param-name>redirectUrl</param-name>
        <param-value>/ebusiness/merchant/pc/account/login</param-value>
    </init-param>
    <init-param>
        <description>
				另外，参数 redirectUrl 的值不用包含在该正则表达式中，因为 redirectUrl 对应的 url
				会被自动放行。
				还有一点需要说明的是，该参数的值不包含web应用的 ContextPath。
        </description>
        <param-name>excepUrlRegex</param-name>
        <param-value>/ebusiness/merchant/pc/login|(merchant/merLoginPost)</param-value>
        <param-value>/ebusiness/merchant/pc/merchant/merLoginPost</param-value>
    </init-param>
</filter>

<filter-mapping>
	<filter-name>merSessionFilter</filter-name>
	<url-pattern>/ebusiness/merchant/pc/*</url-pattern>
</filter-mapping>
```

<div class="row" style="margin-left: 10pt;">
<p style="float: left; font-size: 9pt; margin-right:1em;"> 
   <a href="{{ site.baseurl }}/img/blog/lb-lrg/img1.jpg" data-lightbox="gallery1" data-title="The first image" style="float: left; margin-right: -10%; margin-bottom: 1em;">
     <img src="{{ site.baseurl }}/img/blog/lb-sm/lbs01.png">Image#01</a></p>
        
<p style="float: left; font-size: 9pt; margin-right:1em;"> 
   <a href="{{ site.baseurl }}/img/blog/lb-lrg/img2.jpg" data-lightbox="gallery1" data-title="The second image" style="float: left; margin-right: -10%; margin-bottom: 1em;">
     <img src="{{ site.baseurl }}/img/blog/lb-sm/lbs02.png">Image#02</a></p>
</div>   

> If the photos are **not** included in the `div` the text will float on the right.

###### Image Source: [UNSPLASH](https://unsplash.com/photos/j0g8taxHZa0)