---
layout:     post
title:      "Python3 Spider(Step one:Open Url)"
subtitle:   "Crawling WebPage Using Python3"
date:       2016-07-25 17:19:00
author:     "ZanXus"
header-img: "img/blog/header/post-bg-01.jpg"
thumbnail: /img/blog/thumbs/thumb01.png
tags: [urllib]
category: [python3]
comments: true
share: true
---

# It's my beginning of leraning python.

## My first executable python program.It's made to crawl great images from [unsplash](https://unsplash.com/).

### The code programed with urllib which is an extensible python3 library for opening urls.(PS: In python3,urllib and urllib2 was merge to urllib).

> ### This website need ssl connection,so use Basic Http Ahtuentication here.

~~~python
import urllib.request

handler = urllib.request.HTTPBasicAuthHandler()
handler.add_password(realm='', uri='https://unsplash.com/', user='***', passwd='***') # The user and passwd need to be from a real account,so it can not shown here.You can regist yourself,if needed.
opener = urllib.request.build_opener(handler)
urllib.request.install_opener(opener)
req = urllib.request.urlopen('https://unsplash.com/')
print(req.read().decode('utf-8'))
~~~
