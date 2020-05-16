# 분산 시스템 추적 시스템

이전 포스팅에서 MDC 를 이용한 로그를 남기고 여러 시스템간의 로그를 추적하는 기능에 대해서 포스팅을 했었다. [Slf4j Mapped Diagnostic Context (MDC) 를 이용한 사용자 요청 추적](https://4ppl3hun73r.github.io/post/2020/02/slf4jMDCLogging) 나중에 확인해 보니 이미 MSA 환경에서의 요청 추적을 위한 기술들이 완성이 되어 있었다.

"내가 생각하는 것은 이미 구현되어 있고 만약 없다면 누군가 만들고 있을 것이다" 라는 오랜 격언을 다시 한번 새기면서 분산 시스템에서의 추적 시스템에 대해서 공부를 하였다.

## Google Dapper

[Dapper, a Large-Scale Distributed Systems Tracing Infrastructure](https://research.google/pubs/pub36356/)

역시 구글이다. 논문의 작성일은 2010년... 내가 막 취직한 꼬꼬마 시절에 이미 완성된 기술 디자인이 존재 했다.
후에 서술할 twitter 의 zipkin, uber의 jaeger 등 많은 추적 시스템은 이 Dapper 논문을 기반으로 구현되었다고 한다.


## 핵심 컨셉

### SpanId
하나의 작업의 단위로 새로운 호출시 신규 SpanId 생성한다. 첫번째로 생성되는 SpanId를 Root Span(TraceId)라고 부른다.
SpanId 발급은 최대한 가볍게 생성해야 하며, 중복이 발생할수 있음을 인정하고 발급한다.
zipkin과 jaeger는 모두 long 값을 기준으로 random으로 생성한다. 

## opentracing project
주소 : https://opentracing.io/
CNCF(Cloud Native Computing Foundation) 에서 지원하는 분산환경 추적을 위한 프로젝트이다.
언어 / 플랫폼에 구애받지 않고 분산 추적을 지원하는 API 를 제공하고 있다.
밑에서 이야기할 zipkin / jaeger 모두 이 opentracing api 에 기반을 두고 있다.

### zipkin, jaeger
가장 많이 사용되는 솔루션으로는 twitter의 zipkin과 uber의 jaeger 가 있다. 둘다 강력한데 kubernetes 환경에서의 잠재력에 있얻서 jaeger가 더 많은 인기를 얻고 있다. 
특히나 jaeger 는 zipkin API 와 호환이 되므로 jaeger로 먼저 구축하고 사용하다가 zipkin으로 넘어가는 방법도 하나의 방법이 될 수 있다.

# 투토리얼
java : https://github.com/yurishkuro/opentracing-tutorial/tree/master/java
others : https://opentracing.io/docs/getting-started/tutorials/