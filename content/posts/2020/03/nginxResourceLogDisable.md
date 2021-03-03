+++
title = "nginx resource logging disable 처리"
description = "nginx resource logging disable 처리"
toc = true
authors = ["jiho"]
tags = ["nginx"]
categories = ["개발"]
series = ["2020"]
date =  "2020-03-14T00:00:00+09:00"
lastmod = "2020-03-14T00:00:00+09:00"
featuredVideo = ""
featuredImage = ""
draft = false
+++

## 문제점
css, js, png 같은 정적 자원들이 log 에 표시되는걸 없애고 싶다

## Nginx Log 제어 방법

[공식문서, logging conditional](https://docs.nginx.com/nginx/admin-guide/monitoring/logging/#conditional)

```
map $uri $name {
    .(css|js|png)$   0;
    default       1;
}
access_log logs/access.log main if=$loggable;

or

set $loggable 1;
access_log logs/access.log main if=$loggable;

if ($uri ~* ".(css|js|png)$") {
    set $loggable 0;
}
```