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

### 3. jenkins 설치
```
목표는 
1. 젠킨에서 코드 레포지토리 변경사항이 있을 경우 이를 바탕으로 docker image build 및 push
2. 이후 배포 전용!!!레포지토리에 새로운 이미지 태그 반영(deployment.yaml의 이미지 태그 변경)
3. argo CD는 배포 전용 레포지토리로부터 auto sync

- 현재 레포지토리를 배포 전용 레포로 사용할것이다.
- https://github.com/skarltjr/ci_cd_test 는 코드 레포지토리
```
- 우선 젠킨스를 설치하자
- 젠킨스는 별도의 인스턴스를 생성해서 따로 관리해보고자한다.
- https://github.com/skarltjr/Kubernetes-with-Docker/blob/main/쿠버네티스_dev(6)/4.%20jenkins를%20통한%20CI.md
- <img width="1817" alt="스크린샷 2022-03-09 오후 6 01 26" src="https://user-images.githubusercontent.com/62214428/157408221-ec95394f-1854-4836-b8aa-227f614eeaf1.png">

### 4. jenkins ci 파이프라인 구성
- https://hwannny.tistory.com/113 참고
- 
