# Github Page 를 만들어 보자


## 신규 레파지토리 생성

[신규 레포 생성](https://github.com/new)
![CreateNewRepo](https://4ppl3hun73r.github.io/post/2019/11/createNewRepo.png)
레파지토리 이름은 반드시 {profile id}.github.io 로 생성해야 한다.

## GitHub Pages 활성화

각 레파지토리의 세팅 페이지에서 ( ex: https://github.com/4ppl3Hun73r/4ppl3Hun73r.github.io/settings ) 에서 GitHub Pages Theme 를 선택해주게 되면 바로 기본 페이지가 생성된다.

![enableGitHubPages](https://4ppl3hun73r.github.io/post/2019/11/enableGitHubPages.png)

## 이렇게 생성하게 되면

1. [Jekyll](https://jekyllrb.com/) 기반으로 생성 된다.
2. https://github.com/pages-themes/ 하위에서 선택한 테마의 상세 구성을 확인 할 수 있다.
3. 마음에 드는 테마가 없다면 1번의 Jekyll 에서 원하는 테마를 install 함으로써 직접 구성할 수도 있다.
4. 기본적으로 _config.yml 을 수정해서 title / description / ga 등을 세팅 할 수 있다.

## 다른 테마를 install 할 정도는 아닌데 소소하게 수정하고 싶은데...

내가 설치한 cayman 기준으로 원본 레파지토리의 https://github.com/pages-themes/cayman 의 _layout/default.html 을 내 레파지토리로 가져 온다.
해당 파일을 수정하면 간단하게 커스터마이징이 가능하다.

예를 들면 난 skip to the content 가 최상단에 있는 마크업이 마음에 들지 않아서 
![skipToTheContent](https://4ppl3hun73r.github.io/post/2019/11/skipToTheContent.png)
default.html 에서 해당 마크업을 제거 하였다.