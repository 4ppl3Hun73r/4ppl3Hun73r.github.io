+++
title = "Nginx 302 Redirect 설정시 Upstream Name으로 이동할때 처리"
description = "Nginx 302 Redirect 설정시 Upstream Name으로 이동할때 처리"
toc = true
authors = ["jiho"]
tags = ["nginx"]
categories = ["개발"]
series = ["2019"]
date =  "2019-12-09T00:00:00+09:00"
lastmod = "2019-12-09T00:00:00+09:00"
featuredVideo = ""
featuredImage = ""
draft = false
+++

```nginx
upstream target {
    server 127.0.0.1;
    ...
}

...

location / {
    proxy_pass http://target;
}
```
위와 같이 설정했을때 기본적은 proxy 는 문제 없이 동작하는데 302 처리할때는 url이 http://target 으로 사용자에게 처리되는 경우가 발생했었다.

아래와 같이 header를 설정해서 문제를 해결하였다.
```nginx
location / {
    proxy_set_header Host $http_host;
}
```