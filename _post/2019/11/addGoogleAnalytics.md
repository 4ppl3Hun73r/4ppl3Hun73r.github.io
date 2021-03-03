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

## Google Search Console 연결

### Search Console 연결을 하는 이유

1. 구글에서 내 사이트가 잘 노출되는지 확인 가능
2. 어떤 검색어로 유입되는지 확인 가능
2. 검색이 잘 되는 가이드 라인을 지속적으로 피드백 해줌
3. 검색 제외 설정 가능

### Search Console 가입하기

1. [Google Search Console](https://search.google.com/search-console/about)에 들어가서 가입을 한다.
2. 내 사이트 등록 작업을 진행 해 준다.
3. URL 인증 방식으로 진행 한다.
   ![sc2](https://4ppl3hun73r.github.io/post/2019/11/sc2.png)
4. GA 와 연동을 해놨기 때문에 자동으로 인증이 된다.
   - GA 연동이 안되어 있다면 Search Console에서 제공하는 head 마크업을 _layouts/default.html 에 삽입해 주면 된다.

### GA - Search Console 연동 하기

1. GA 관리 페이지에서 제품 연결하기를 선택한다.
   ![sc1](https://4ppl3hun73r.github.io/post/2019/11/sc1.png)
2. Search Console 연동을 선택 한다.
3. 추가 링크를 눌러서 위에서 가입한 Search Console을 선택한다.
   ![sc4](https://4ppl3hun73r.github.io/post/2019/11/sc4.png)

