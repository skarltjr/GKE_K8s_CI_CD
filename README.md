# GKE_K8s_CI_CD
GKE를 활용한 쿠버네티스 클러스터 구축 및 젠킨스CI / argo CD 구성해보기
```
목표 및 진행 방향 정리
1. 소스코드 레포지토리에서 변동 사항이 발생
2. 웹훅 트리거 -> 젠킨스에서 소스코드 변동 사항을 반영
    -> 빌드 -> 새로운 이미지 생성 및 태깅 -> 도커허브 푸쉬 -> 별도의 manifest repo안 deployment.yaml을 새로운 이미지 태그로 업데이트
3. argo cd는 manifest repo 변동사항을 바탕으로 재배포    
  
- https://github.com/skarltjr/ci_cd_test 는 코드 레포지토리
- 별도의 k8s manifest repository 
```

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
- 우선 젠킨스를 설치하자
- 젠킨스는 별도의 인스턴스를 생성해서 따로 관리해보고자한다.
- https://github.com/skarltjr/Kubernetes-with-Docker/blob/main/쿠버네티스_dev(6)/4.%20jenkins를%20통한%20CI.md
- <img width="1817" alt="스크린샷 2022-03-09 오후 6 01 26" src="https://user-images.githubusercontent.com/62214428/157408221-ec95394f-1854-4836-b8aa-227f614eeaf1.png">

### 4. jenkins ci 파이프라인 구성
- https://hwannny.tistory.com/113 참고
- 소스코드 레포지토리에 웹훅 설정
- <img width="631" alt="스크린샷 2022-03-09 오후 8 54 30" src="https://user-images.githubusercontent.com/62214428/157437086-cc35d6fd-ed7f-4ade-bca1-a1909da4e0be.png">
```
도커허브에 이미지 푸쉬하기위해 젠킨스에서 도커허브에 접근할 수 있도록 credential발급
manage credentials -> jenkins -> Global credentials -> add 
유저네임 패스워드는 도커허브 계정
id는 pipeline구성시 활요할거니까 기억하기

추가로 deployment.yaml service.yaml은 gitops repo에 존재하고 나중에 이미지가 새로 빌드되었을 때 해당 레포에 변경이 반영되야한다
그래서 깃허브 credential도 동일하게 만들어준다. 이때 비밀번호는 토큰으로

도커허브 credential & github credential을 젠킨스에 생성해둔다.
```


```
이미지 생성을 위한 도커파일을 작성하자

FROM openjdk:11-jdk AS builder
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .
COPY src src
RUN chmod +x ./gradlew
RUN ./gradlew bootJar

FROM openjdk:11-slim
# 위에서 빌드한 jar 파일을 실행해 주기 위해 다시 JDK 11 버전을 베이스로 설정합니다.

COPY --from=builder build/libs/*.jar springboot-sample-app.jar
VOLUME /tmp
EXPOSE 8080
# builder를 통해 생성된 jar 파일을 이미지로 가져옵니다.
# 8080 포트를 공개한다고 명시합니다.

ENTRYPOINT ["java", "-jar", "/springboot-sample-app.jar"]
# 가져온 jar 파일을 실행시킵니다.
```

```
배포 스크립트. 
파이프라인을 구성해보자
pipeline{
    agent any

    environment {
        dockerHubRegistry = 'skarltjr/k8s'
        dockerHubRegistryCredential = 'docker-hub'
        githubCredential = 'github'
    }

    stages {
        stage('check out application git branch'){
            steps {
                checkout scm
            }
            post {
                failure {
                    echo 'repository checkout failure'
                }
                success {
                    echo 'repository checkout success'
                }
            }
        }
        stage('build gradle') {
            steps {
                sh  './gradlew build'
                sh 'ls -al ./build'
            }
            post {
                success {
                    echo 'gradle build success'
                }
                failure {
                    echo 'gradle build failed'
                }
            }
        }
        stage('docker image build'){
            steps{
                sh "docker build . -t ${dockerHubRegistry}:${currentBuild.number}"
                sh "docker build . -t ${dockerHubRegistry}:latest"
            }
            post {
                    failure {
                      echo 'Docker image build failure !'
                    }
                    success {
                      echo 'Docker image build success !'
                    }
            }
        }
        stage('Docker Image Push') {
            steps {
                withDockerRegistry([ credentialsId: dockerHubRegistryCredential, url: "" ]) {
                    sh "docker push ${dockerHubRegistry}:${currentBuild.number}"
                    sh "docker push ${dockerHubRegistry}:latest"

                    sleep 10 /* Wait uploading */
                }
            }
            post {
                    failure {
                      echo 'Docker Image Push failure !'
                      sh "docker rmi ${dockerHubRegistry}:${currentBuild.number}"
                      sh "docker rmi ${dockerHubRegistry}:latest"
                    }
                    success {
                      echo 'Docker image push success !'
                      sh "docker rmi ${dockerHubRegistry}:${currentBuild.number}"
                      sh "docker rmi ${dockerHubRegistry}:latest"
                    }
            }
        }
        stage('K8S Manifest Update') {
            steps {
                sh "ls"
                sh 'mkdir -p gitOpsRepo'
                dir("gitOpsRepo")
                {
                    git branch: "main",
                    credentialsId: githubCredential,
                    url: 'https://github.com/skarltjr/kube-manifests.git'
                    sh "sed -i 's/k8s:.*\$/k8s:${currentBuild.number}/' deployment.yaml"
                    sh "git add deployment.yaml"
                    sh "git commit -m '[UPDATE] k8s ${currentBuild.number} image versioning'"
//                     sshagent(credentials: ['19bdc43b-f3be-4cb9-aa1d-9896f503e3e8']) {
//                         sh "git remote set-url origin git@github.com:skarltjr/kube-manifests.git"
//                         sh "git push -u origin main"
//                     }
                    withCredentials([gitUsernamePassword(credentialsId: githubCredential,
                                     gitToolName: 'git-tool')]) {
                        sh "git remote set-url origin https://github.com/skarltjr/kube-manifests"
                        sh "git push -u origin main"
                    }
                }
            }
            post {
                    failure {
                      echo 'K8S Manifest Update failure !'
                    }
                    success {
                      echo 'K8S Manifest Update success !'
                    }
            }
        }

    }
}


참고로 여기서 헤맸다.
깜빡하고 도커 파이프라인 플러그인설치를 안함 젠킨스에.

sh "sed -i 's/k8s:.*\$/k8s:${currentBuild.number}/g' deployment.yaml" 를 통해 
deployment.yaml에 있는 image: skarltjr/k8s:{jenkins build number}중 k8s부터 뒤 모든 부분을  k8s:${currentBuild.number} 
새로 푸쉬된 이미지 태그로 갈아끼운다-> yaml수정이 된다.

sed 커맨드는 참고: https://m.blog.naver.com/hanajava/220595096628
```
```
배포 manifest를 작성하자
참고로 해당 manifest는 별도의 레포지토리 https://github.com/skarltjr/kube-manifests / 소스코드 레포랑 별개다

apiVersion: v1
kind: Service
metadata:
  name: k8s
spec:
  type: NodePort
  selector:
    app: k8s
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30007
      
apiVersion:apps/v1
kind: Deployment
metadata:
  name: k8s
spec:
  selector:
    matchLabels:
      app: k8s
    replicas:3
    template:
      metadata:
        labels:
          app: k8s
      spec:
        containers:
          - name: k8s
            image: {dockerhub}/k8s:{jenkins build number}
            ports:
              - containerPort:8080 
```
- 드디어 완성!
- 이제 소스코드 레포지토리 변경사항 -> 웹훅 -> 젠킨스를 통해 빌드&이미지 새로 구성&이미지 푸쉬 -> k8s manifest repository update가 완료!
- <img width="1622" alt="스크린샷 2022-03-09 오후 11 40 58" src="https://user-images.githubusercontent.com/62214428/157464095-1ef10c23-d70f-4f13-a639-7e936bb829ec.png">
- <img width="1076" alt="스크린샷 2022-03-10 오후 7 27 07" src="https://user-images.githubusercontent.com/62214428/157642811-c4e657e7-9c77-4c8a-b261-42c109bdf83c.png">
- <img width="1319" alt="스크린샷 2022-03-10 오후 7 25 40" src="https://user-images.githubusercontent.com/62214428/157642563-cc685b1a-b736-42bc-b14c-8007e5cb3889.png">


