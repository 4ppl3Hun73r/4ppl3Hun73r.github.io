# V Live Playlist Chrome Extension 요구사항 정의

## conetent 영역 정의
![Content 영역](https://4ppl3hun73r.github.io/post/2019/11/vliveExtensionContent.png)

### 공유 영역 옆에 버튼 추가
* Playlist 추가, PIP 재생 기능을 추가해주는 영역 추가

### Playlist 클릭시 동작 정의
* Playlist 클릭시 플레이리스트 추가 layer 가 오버 됨
* 구글의 플레이리스트 추가 처럼!!!

### 추가 layer 동작 정의
* 최상단 input box에 플레이리스트명 입력 가능
* 체크박스 클릭시 해당 플레이리스트 마지막에 추가됨

### PIP 재생 기능 버튼 클릭시
* 재생중인 영상이 PIP 모드로 재생됨

## popup 영역 정의
![popup 영역](https://4ppl3hun73r.github.io/post/2019/11/vliveExtensionPoup.png)

### 아이콘 상태
* 플레이리스트 관련 tab이 없을때 : 빨간색 버튼 아이콘
* 플레이리스트 재생중 : 재상 마크 아이콘
* 일시정지 상태 : 일지 정지 마크 아이콘

### 아이콘 클릭시
* 현재 재생중인 플레이 리스트 재상 팝업창 노출
* 앞 / 뒤 / 재생 / 일시정지
* SelectBox로 플레이리스트 선택 기능 추가
* 플레이리스트에 속한 VideoList 노출 최대 N 개 노출 후 스크롤 처리
* Manage Playlsit 클릭시 playlist 관리 신규 탭이 생성됨

## playlist 관리 탭
* SelectBox로 플레이리스트 선택 기능 추가
* 삭제 기능 추가
* 이름 변경 기능 추가
* video 목록 순서 변경 가능
* video 삭제 기능