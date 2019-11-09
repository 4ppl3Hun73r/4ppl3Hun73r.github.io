# Google Analytics 붙이기

## Google Analytics 계정 만들기

[무료 계정 생성](https://analytics.google.com/analytics/web/provision/#/provision/create) 을 통해서 계정을 만든다

생성 자체는 별거 없긴 한데 혹시 모르니 스크린샷을 남겨 놓는다.
![ga1](https://4ppl3hun73r.github.io/post/2019/11/ga1.png)
![ga2](https://4ppl3hun73r.github.io/post/2019/11/ga2.png)
![ga3](https://4ppl3hun73r.github.io/post/2019/11/ga3.png)

## 생성 이후 GA ID 확인

신규 계정이 생성되고 계정이 생성되자 마자 발급된 ID를 확인 할수 있다.
![ga3](https://4ppl3hun73r.github.io/post/2019/11/gaID.png)

## ID 추가

이 ID 를 가지고 GA 스크립트를 로딩하는 로직을 Page의 Head 부분에 넣어줘야 하는데 

Jekyll 기준으로 _config.yml에 ID 값을 추가해 줌으로써 바로 적용 할수 있다.

_config.yml
```yml
google_analytics: UA-1******
```
