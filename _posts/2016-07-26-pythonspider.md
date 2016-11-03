---
layout:     post
title:      "Python3 Urllib"
subtitle:   "Crawling with urllib"
date:       2016-07-25 17:19:00
author:     "ZanXus"
header-img: "img/blog/header/post-bg-01.jpg"
thumbnail: /img/blog/thumbs/thumb01.png
tags: [urllib]
category: [python3]
comments: true
share: true
---

> ### Simple trying with urllib.

~~~ python
import urllib.request

handler = urllib.request.HTTPBasicAuthHandler()
handler.add_password(realm='', uri='https://unsplash.com/', user='***', passwd='***') # The user and passwd need to be from a real account but this line is not necessary.
opener = urllib.request.build_opener(handler)
urllib.request.install_opener(opener)
req = urllib.request.urlopen('https://unsplash.com/')
print(req.read().decode('utf-8'))
~~~
