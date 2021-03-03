# Slf4j Mapped Diagnostic Context (MDC) 를 이용한 사용자 요청 추적

## 문제점
특정 에러 로그를 봤을떄 에러의 시작점을 찾기까지 추적하는데 불편함을 느낌

## 개선 아이디어
사용자의 요청 마다 Unique ID 를 발급하여 로그를 남길때 마다 추가
로그에 찍힌 ID를 가지고 역추적을 하여 빠르게 에러의 시작점을 파악

## Slf4j Mapped Diagnostic Context (MDC)
Thread Local에 로그 관련 정보를 남길수 있는 방법으로
put, get, remove, clear 등등의 interface를 제공한다

%X{mdcKey} 를 통해서 Pattern 에 추가할 수 있다. ([logback mdc manual](http://logback.qos.ch/manual/mdc.html))
```
<encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
    <layout class="ch.qos.logback.classic.PatternLayout">
        <Pattern>%d{yyyy-MM-dd HH:mm:ss} [%-5level] \(%F:%L\) [%X{id}] %message%n</Pattern>
    </layout>
</encoder>
```

## Spring Boot에 설정
*Filter* or Interceptor 를 이용해서 처리

Unique Id 를 생성해서 세팅하는 Filter 생성
```
public class LogbackMDCFilter implements Filter {
    private static final String ID = "id";
    private static final String ID_HEADER = "x-u-id";

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        try {
            setUniqueId(servletRequest, servletResponse);
        } catch (Exception e) {
            // ignore exception
        }
        try {
            filterChain.doFilter(servletRequest, servletResponse);
        } finally {
            MDC.remove(ID);
        }
    }

    private void setUniqueId(ServletRequest servletRequest, ServletResponse servletResponse) {
        HttpServletRequest httpRequest = (HttpServletRequest) servletRequest;
        HttpServletResponse httpResponse = (HttpServletResponse) servletResponse;
        // request header에 id 값이 세팅되어 있으면 그 값으로 세팅
        String uniqueId = httpRequest.getHeader(ID_HEADER);

        if (StringUtils.isEmpty(uniqueId)) {
            uniqueId = UUID.randomUUID();
        } 

        // MDC 설정
        MDC.put(ID, uniqueId);
        // Webserver logging 을 위한 Response Header 세팅
        httpResponse.addHeader(ID_HEADER, uniqueId);
    }
}
```

Filter 등록
```
	@Bean
	public FilterRegistrationBean<LogbackMDCFilter> logbackFilter() {
		FilterRegistrationBean<LogbackMDCFilter> registrationBean = new FilterRegistrationBean<>();
		LogbackMDCFilter logbackMDCFilter = new LogbackMDCFilter();
		registrationBean.setFilter(logbackMDCFilter);
		registrationBean.addUrlPatterns("*");
		return registrationBean;
	}
```

코드를 보면 request header에 id 값이 있으면 해당 값을 계승하도록 구현을 하였다. 이는 MSA 환경에서 여러 서비스간 로깅 규약을 맞췄다고 했을때 여러 요청들을 하나의 컨텍스트로 묶기 위함이다.

## RestTemplate 사용시 자동으로 Header에 Unique ID 값을 세팅해서 넘기게 처리
위에서도 설명했지만 MSA 환경에서 다른 서비스를 호출할때 발급한 Unique Id 값을 유지하게 되면 요청 발생 위치 추적도 용이하고 여러 로그를 ES 등을 통해서 묶어서 볼수도 있다. 그렇기에 API Call에 사용되는 RestTemplate 의 interceptors 기능을 이용해서 Unique Id를 계승 시키도록 한다. 
```
    @Bean
	public RestTemplate apiRestTemplte() {
		var httpClientBuilder = HttpClientBuilder.create().useSystemProperties();
		var clientHttpRequestFactory = new HttpComponentsClientHttpRequestFactory(httpClientBuilder.build());
		var restTemplate = new RestTemplate(clientHttpRequestFactory);
		List<ClientHttpRequestInterceptor> interceptors = restTemplate.getInterceptors();
		if (CollectionUtils.isEmpty(interceptors)) {
			interceptors = new ArrayList<>();
		}
		// Unique Id를 Header 에 넣어서 relay 처리
		interceptors.add(
			(request, body, execution) -> {
				HttpHeaders headers = request.getHeaders();
				String uniqueId = MDC.get("id");
				if (StringUtils.isNotEmpty(uniqueId) &&
					!headers.containsKey("x-u-id")) {
					headers.set("x-u-id", uniqueId);
				}
				return execution.execute(request, body);
			}
		);
		restTemplate.setInterceptors(interceptors);
		return restTemplate;
	}
```

## rabbitMq 사용시 자동으로 Unique ID 값 세팅
MSA 환경에서 Http call 이외에도 Message queue 를 이용한도 할때 이또한 Unique Id가 자동으로 계승 될수 있도록 설정을 할수 있다.

rabbitTemplate을 이용한 전송 발생시 Unique Id 세팅
```
	@Bean
	public RabbitTemplate rabbitTemplate() {
		RabbitTemplate template = new RabbitTemplate(connectionFactory());
		template.setBeforePublishPostProcessors(message -> {
			// rabbitMQ로 요청할때 Unique Id 세팅해서 넘겨줌
			String uniqueId = MDC.get("id");
			if (StringUtils.isNotEmpty(uniqueId)) {
				message.getMessageProperties().setHeader("id", uniqueId);
			}
			return message;
		});
		return template;
	}
```

rabbit Listener에 Advice 설정할 클래스 생성
```
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.MDC;
import org.springframework.amqp.core.Message;

public class LogbackMDCRabbitAdvice implements MethodInterceptor {
    private static final String ID = "id";

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        try {
            setUniqueId(servletRequest, servletResponse);
        } catch (Exception e) {
            // ignore exception
        }

        try {
            return methodInvocation.proceed();
        } finally {
            MDC.remove(ID);
        }
    }
    private void setUniqueId(MethodInvocation methodInvocation) {
        Object[] arguments = methodInvocation.getArguments();

        Message message = (Message) arguments[1];
        String uniqueId =(String) message.getMessageProperties().getHeaders().get(ID);
        if (StringUtils.isEmpty(uniqueId)) {
            uniqueId = UUID.randomUUID();
            message.getMessageProperties().setHeader(ID, uniqueId);
        }
        MDC.put(ID, uniqueId);
    }
}
```
Rabbit Listener에 Advice 설정
```
	@Bean
	public SimpleRabbitListenerContainerFactory rabbitFactory(ConnectionFactory connectionFactory
		, LogbackMDCRabbitAdvice logbackMDCRabbitAdvice) {
		SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
		factory.setAdviceChain(logbackMDCRabbitAdvice);
		return factory;
	}

	@Bean
	public LogbackMDCRabbitAdvice logbackMDCRabbitAdvice() {
		return new LogbackMDCRabbitAdvice();
	}
```
