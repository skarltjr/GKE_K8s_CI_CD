# GKE_K8s_CI_CD
GKE를 활용한 쿠버네티스 클러스터 구축 및 젠킨스CI / argo CD 구성해보기


### 1. GKE 클러스터 구축
- <img width="1001" alt="스크린샷 2022-03-09 오후 1 14 58" src="https://user-images.githubusercontent.com/62214428/157371618-e7d768dd-5a06-426a-aae4-4b06c5a85183.png">
- 돌발상황을 고려하여 정적 버전을 선택
- default-pool에서 노드의 성능을 선택할 수 있지만 여기서는 기본 세팅을 따르기로한다.
- 만들기 선택!

### 2. cloud shell setting
- <img width="1487" alt="스크린샷 2022-03-09 오후 1 25 42" src="https://user-images.githubusercontent.com/62214428/157372600-11c9f7b8-2782-4698-a43d-71f6cf376f1e.png">
- 1번을 클릭하여 cloud shell을 열고
- 2번 - 클러스터 연결을 눌렀을 때 나오는 명령어를 복사하여
- 3번 - 쉘에 붙여넣으면 kubectl이 자동으로 활성화!!
- 이용준비는 완료!
