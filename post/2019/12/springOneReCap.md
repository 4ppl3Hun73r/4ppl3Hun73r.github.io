# Spring One re:Cap Seoul 2019 참석

12월 18일에 2019년도 Spring One re:Cap 행사에 참석을 했습니다.

전체적인 순서는
Keynote 정리 -> 기술 세션 정리 -> Live Coding with Mark Heckler -> 
Apache Geode Summit re:Cap -> Concourse / Spinnaker
로 진행 되었습니다.

Live Coding 순서부터 회사에서 급한 일이 생겨서 원격에서 대응하느냐고 뒤에는 놓쳤는데요.
일단 앞부분만이라도 정리를 했습니다.

## Intro - Keynote Review

Home Depot
좋은 개발자 확보
개발자들의 업무 환경 개선
개발자들의 산출물을 프로덕트 환경에 빠르게 배포하도록 환경 구성
 - Top Down / 관료주의를 없애고 기술스택을 초기화 하고 평등하게 다시 구성

America Trade
 - Seed Team을 만들어서 에자일을 적용하여 성과가 내는걸 보이고 다른 팀에 자연스럽게 전파

Netflix
 - 자체 in house에서 spring으로 넘어가려 한다.
 - 자체 영상들을 찍다 보니 기존 플랫폼 영상 제공을 넘어서 신규 기능들에 대한 요구사항이 생김
 - 자체적으로 플랫폼을 유지하고 개선하는 부분에 한계가 있어서 spring 생태계에 적극적으로 참여

GM
 - 개발자를 중요하게 생각한다
 - Project에서 Product 로 마인드를 변경함
   - 본인의 서비스를 하나의 Product로 바라보고 직접 관리하고 업그레이드 하는 관점의 전환
 - 기술은 도구일 뿐 문화와 관점의 전환이 필요하다

JP Mogun
 - 1일 6000조 정도 거래가 발생하는데 이를 유지하는 이력이 25K 정도 있고 어쩌구 저쩌구
 - 특이하게 3개의 cloud(google, aws, azure) + 1개의 자체 cloud를 모두 사용
 - 일 배포를 되게 많이함 6000번 정도 , 마이크로 서비스화가 되어 있음
   - 배포가 많지만 99% 이상의 가용성을 가짐
 - 1만2천명이 참여하는 Inner 커뮤니티가 구성되어 있음

DICK'S 
 - 모노로 되어 있는걸 마으크로 클라우드로 전환
 - 물건의 검색에 중점을 두고 있음
 - 키노트 중에 리얼 서비스를 죽여 버림. 하지만 서비스는 문제가 없었고 바로 복구됨

내년은 시애틀에서 Spring One 행사 진행

## Technical Highlights
Cloud-native 를 하기 위해서
 - Spring boot, kubernetes, kafka

CI / CD 의 중요성
 - 은근 안정적이다
 - 배포 범위가 작아져서 장애가 날 확률이 줄어든다

Pivotal 과 VMware merge 함
 - VMware는 기존 가상 머신의 관리 기술에 컴포넌트 관리의 영역까지 지원
 - Pivotal은 기존 플랫폼에 kubernetes 관련 지원을 좀더 강화함

그리고 Pivotal 의 제품 설명
 - kubernetes 로 컨테이너 관리
 - 관리 운영툴 제공
 - jdk, os 업그레이드를 쉽게 할수 있게 지원
 - 로드 밸런스 및 dns update 자동화
 - Event Streams 방식으로 동작하게 만들어서 히스토리 보존, 리플레이 등등 처리
 - Reactive 사상 반영
 - Reconciliation (화해? 틀린 부분을 맞춰 나가기) => 상태머신으로 동작을 관리함
 - Immutable > reactive > Reconciliation > Composable > Immutable
   - 4개의 원칙으로 구성
 - 자세한건 youtube로

## Live Coding with Mark Heckler
 - Github 코드 : https://github.com/mkheck/building-reactive-pipelines

## 정리하면서...
개발자들에게 기회가 더 많아지는 세상이 되가고 있다
Cloud Platform에 대한 공부가 필요 하다
CI / CD 에 대한 고민
Kubernetes 를 공부하자



