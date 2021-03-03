+++
title = "rSocket"
description = "rSocket"
toc = true
authors = ["jiho"]
tags = ["spring", "rSocket"]
categories = ["개발"]
series = ["2020"]
date =  "2020-01-18T00:00:00+09:00"
lastmod = "2020-01-18T00:00:00+09:00"
featuredVideo = ""
featuredImage = ""
draft = false
+++

Netflix 에서 개발한 Reactive Stream을 기반으로 한 어플리케이션 프로토콜.
마이크로 서비스상에서 Http 통신이 비효율적인 부분이 있기 때문에 오버헤드가 적은 프로토콜로 대체하려는 용도로 개발됨

# 사용해 보기
https://start.spring.io/ 에서 rsocket Dependencies 를 추가하여 프로젝트 생성 후 작업 진행

## 서버
### 설정
application.properties 에 rsocket port 설정
```
spring.rsocket.server.port=7000
spring.main.lazy-initialization=true
```

### EndPoint 생성

@MessageMapping 사용
```
@Controller
public class RSocketController {


    @MessageMapping("hello")
    public Mono<String> helloRSocket(String name) {
        return Mono.just("Hello " + name + ", time : " + Instant.now());
    }
}
```

## 클라이언트
### 설정
RSocketRequester Bean 생성
```
	@Bean
	public RSocketRequester rSocketRequester(RSocketRequester.Builder builder) {
		return builder
				.connectTcp("localhost", 7000)
				.block();
	}
```

### EndPoint 호출
```
    @GetMapping("hello/{name}")
    public Publisher<String> hello(@PathVariable String name) {
        return rSocketRequester
            .route("hello")
            .data(name)
            .retrieveMono(String.class);
    }
```

# 사용해 보니...
Springboot 로 적용했을때 WebFlux 가 강제됨
성능상으로 이점은 확실해 보임
간단한 테스트만 한거여서 fail 관련된 작업에 대한 고민이 필요해 보임

[테스트 코드](https://github.com/4ppl3Hun73r/rsocket-test)