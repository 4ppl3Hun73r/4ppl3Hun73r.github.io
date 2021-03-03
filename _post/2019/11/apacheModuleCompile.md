# Apache Httpd Module Compile

Apache 가 설치되어 있을때 추가로 모듈을 설치해야 할때

## Apache Source Code Download

[Apache Http Server Download Page](https://httpd.apache.org/download.cgi) 에서 httpd-{version}.tar 다운로드

ex)
```bash
wget http://apache.mirror.cdnetworks.com//httpd/httpd-2.4.41.tar.gz
```

## Compile

mod_include.so 가 필요할때 컴파일로 모듈 만들기
```bash
$APACHE_HOME/bin/apxs -i -a -c $APACHE_SOURCE_HOME/modules/filters/mod_include.c 
```