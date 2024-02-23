# Nocode AI 플랫폼 구축

# Table of contents

- [Nocode AI 플랫폼 MSA 구현](#---)
  - [클라우드 아키텍처 설계](#클라우드-아키텍처-설계)
    - [클라우드 아키텍처 구성](#클라우드-아키텍처-구성)
    - [MSA 아키텍처 구성](#MSA-아키텍처-구성도)
  - [서비스 분리/설계 역량](#서비스-분리/설계-역량)
    - [도메인분석 이벤트스토밍](#도메인분석-이벤트스토밍)
  - [MSA 개발](#MSA-개발)
    - [분산트랜잭션 SAGA](#분산트랜잭션-SAGA)
    - [보상처리 Compensation](#보상처리-Compensation)
    - [단일진입점 Gateway](#단일진입점-Gateway)
    - [분산 데이터 프로젝션 CQRS](#분산-데이터-프로젝션-CQRS)
  - [클라우드 배포 역량](#클라우드-배포-역량)
    - [Container 운영](#Container-운영)
  - [컨테이너 인프라 설계 및 구성 역량](#컨테이너-인프라-설계-및-구성-역량)
    - [컨테이너 자동확장 HPA](#컨테이너-자동확장-HPA)
    - [컨테이너로부터 환경 분리 ConfigMap/Secret](#컨테이너로부터-환경-분리-ConfigMap/Secret)
    - [클라우드스토리지 활용 PVC](#클라우드스토리지-활용-PVC)
    - [셀프 힐링/무정지배포 Liveness/Rediness Probe](#셀프-힐링/무정지배포-Liveness/Rediness-Probe)
    - [서비스 매쉬 응용 Mesh](서비스-매쉬-응용-Mesh)
    - [통합 모니터링 Loggregation/Monitoring](통합-모니터링-Loggregation/Monitoring)

# 클라우드 아키텍처 설계

기능적 요구사항

1. 사용자 접속 권한에 따른 메뉴 노출을 제어해야함
1. 사용자 권한 관리 기능이 필요함
1. 모델생성 요청에대한 승인기능이 필요함
1. Vertica(외부) 배치 연동
1. 모델 작업 관리를 위한 Queue 제공
1. 타겟추출 및 모델생성 요청 상태 변경시스윙챗 알림 서비스 제공
1. 학습/생성을 요청한 모델의 요건정보 및 모델학습 수행 단계에 대한 상태정보 연동
1. 타겟추출 요청 접수
1. 모델생성 요청 접수
1. 타겟추출이 완료되면 MIDAS(외부)으로 LMS 대량전송 요청
1. 생성 요청은 관리자에 의해 반려 가능
1. 모델생성이 완료되면 타겟추출할 수 있음
1. 모델생성 진행중 작업 요청을 취소 할 수 있음

비기능적 요구사항

1. 모델생성이 장애가 나더리도 타겟추출 요청은 계속 받아야한다. Async (event-driven)
1. 요청한내역의 상태를 CQRS
1. 생성 요청 상태가 바뀔때마다 스윙챗(사내메신저) 등으로 알림을 줄 수 있어야 한다 Event driven

## 클라우드 아키텍처 구성

![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/f85ca529-dc90-48cb-b6d9-a0ebb0e29e7d)

## MSA 아키텍처 구성도

![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/3351aa9f-bf48-4cf8-957e-020614b15251)

# 서비스 분리/설계 역량

## 도메인분석 이벤트스토밍

### 이벤트 도출

![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/8587f5ba-9b0d-4a0b-b6c5-8f1fd69c3013)

    - 요구사항에 맞추어 필요한 이벤트들을 나열

### 부적격 이벤트 탈락

![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/4cb42283-dff3-4731-b24e-27f411b20eb8)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
      요구사항중 로그인했을때 뿌려주는 메뉴리스트는 Common 성격으로 제외함

### 액터, 커맨드 추가

![finalProject-2024-02-21 01 04 27 (3)](https://github.com/grapefr/CloudNativeReport/assets/68136339/0bdf91f4-aa94-4105-96cd-953953ed3dc8)

### 어그리게잇으로 묶기

![finalProject-2024-02-21 01 04 28](https://github.com/grapefr/CloudNativeReport/assets/68136339/9776acb3-09b2-42f2-82c8-1bb9bdd52164)

    - Model 생성 요청, Target 추출 요청 프로세스 분리하며, 공통영역을 후속 프로세스를 나눔.

### 폴리시 부착

![finalProject-2024-02-21 01 04 28 (2)](https://github.com/grapefr/CloudNativeReport/assets/68136339/bc444f22-d265-444a-9f9c-4dd72065293a)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![finalProject-2024-02-21 01 04 28 (3)](https://github.com/grapefr/CloudNativeReport/assets/68136339/0de7e20d-b6fe-4e56-9bc2-5ac621ff31cf)

### 완성된 모형

![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/f8d3f13d-6fa9-4e38-882a-c6e59c6198b9)

    - View Model 추가

# MSA 개발

## 기능 확인

```
# 모델 생성 요청
 http gateway:8080/models userId=test state=request modelName=TestModel10

# 모델 생성 요청 승인
 http PUT gateway:8080/models id=1

# 타겟추출 요청
 http gateway:8080/targets userId=test type=testtype state=request modelRequestId=1
# 타겟추출 승인
 http PUT gateway:8080/targets id=1

# 각 프로세스 승인 처리 진행 후 kafka topic
{"eventType":"Requested","timestamp":1708532929928,"id":15,"state":"requested","userId":"test","modelName":"TestModel5"}
{"eventType":"Requested","timestamp":1708532930286,"id":16,"state":"requested","userId":"test","modelName":"TestModel6"}
{"eventType":"Approved","timestamp":1708532934871,"id":1,"state":"approved","userId":"test","modelName":"TestModel1"}
{"eventType":"ModelCompleted","timestamp":1708532939665,"id":1,"type":"model","state":"completed","requestId":"1"}
{"eventType":"TargetCompleted","timestamp":1708502151904,"id":1,"type":"target","state":"targetCompleted","requestId":"11"}

#요청 취소에 대한 이벤트 pub 내역
{"eventType":"RequestCanceled","timestamp":1708532942744,"id":12,"state":"requestCanceled","userId":"test","modelName":"TestModel2"}
```

![finalproject2024-02-22 16 41 09](https://github.com/grapefr/CloudNativeReport/assets/68136339/92bd78e7-bd09-4b34-966e-dbef8184db0c)

    - 알림 서비스에서의 처리 내역

## 분산 트랜잭션 SAGA

- 각 서비스내에서 사용할 데이터를 개별적으로 관리하고 이벤트 발행을 통하여 관련 서비스들에게 전파함.

```
# 서비스별 테이블 분리

gitpod /workspace/CloudNativeProject (main) $ find ./ -name *Repo*.java
./core/src/main/java/testab/domain/CoreRepository.java
./model/src/main/java/testab/domain/ModelRepository.java
./model/src/main/java/testab/infra/MyRequestRepository.java
./notice/src/main/java/testab/domain/NoticeRepository.java
./target/src/main/java/testab/domain/TargetRepository.java
./target/src/main/java/testab/infra/MyRequestRepository.java

# DB create 및 update 시 이벤트 발행

    @PostPersist
    public void onPostPersist() {
        Requested requested = new Requested(this);
        requested.publishAfterCommit();
    }

    @PostUpdate
    public void onPostUpdate() {
        if (this.state.equals("approved")) {
            Approved approved = new Approved(this);
            approved.publishAfterCommit();
        }
        else if (this.state.equals("rejected")) {
            Rejected rejected = new Rejected(this);
            rejected.publishAfterCommit();
        }
        else if (this.state.equals("stateChanged")) {
            StateChanged stateChanged = new StateChanged(this);
            stateChanged.publishAfterCommit();
        }
        else if (this.state.equals("requested")) {
            Requested requested = new Requested(this);
            requested.publishAfterCommit();
        }
        else if (this.state.equals("requestCanceled")) {
            RequestCanceled requestCanceled = new RequestCanceled(this);
            requestCanceled.publishAfterCommit();
        }
    }

```

## 보상처리 Compensation

- 시나리오에서 모델생성요청을 취소하였을때, 생성 및 학습중인 요청건은 취소할 수 없는 상태가 되며, 완료될때까지 기다려야함. 완료된 후 정상완료로 이벤트를 전파하게될 경우 업무 프로세스 오류가 발생할 수 있음. 해당 이벤트 전파 못하도록 막아줘야함.

```
# 업데이트 전 현재 요청상태를 Feign 으로 확인한 후 업데이트 여부 결정

# Feign 설정
@FeignClient(name = "models", url = "http://localhost:8088")
public interface ModelClient {
  @GetMapping("/models/{id}")
  Approved callOtherService(@PathVariable("id") Long id);
}


# 동기화를 위한 확인

    Approved approved = modelClient.callOtherService(core.getId());
    System.out.println("############ FEIGN:  " + approved);
    if( approved.getState().equals("requestCanceled") ){
      core.setState("requestCanceled");
    }
    else {
      core.setState("completed");
    }

```

## 단일진입점 Gateway

- Gateway는 Spring Gateway를 적용하였으며 관련된 설정은 아래와 같음.

```
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: model
          uri: http://localhost:8082
          predicates:
            - Path=/models/**,
        - id: notice
          uri: http://localhost:8083
          predicates:
            - Path=/notices/**,
        - id: core
          uri: http://localhost:8084
          predicates:
            - Path=/cores/**,
        - id: target
          uri: http://localhost:8085
          predicates:
            - Path=/targets/**,
        - id: frontend
          uri: http://localhost:8080
          predicates:
            - Path=/**
```

## 분산 데이터 프로젝션 CQRS

- Read 전용 DB를 위한 설정은 아래와 같이 발생된 이벤트 Sub 하여 처리

```
    @StreamListener(
        value = KafkaProcessor.INPUT,
        condition = "headers['serviceType']=='model'"
    )
    public void whenRequested_then_CREATE_1(@Payload Requested requested) {
        try {
            if (!requested.validate()) return;

            // view 객체 생성
            MyRequest myRequest = new MyRequest();
            // view 객체에 이벤트의 Value 를 set 함
            myRequest.setId(requested.getId());
            myRequest.setState(requested.getState());
            myRequest.setUserId(requested.getUserId());
            myRequest.setModelName(requested.getModelName());
            // view 레파지 토리에 save
            myRequestRepository.save(myRequest);
            System.out.println("############ CQRS 에 저장:  " + requested);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

# 클라우드 배포 역량

## Container 운영

- 각 서비스들은 CI/CD 파이프라인에 의하여 CI의 특정 이벤트에따라 자동으로 Cloud 환경에 Deploy 하게된다. 관련된 설정을 아래 설명함.

### Github repository 확인

    Github와 Gitlab 에서 소스코드를 관리할 수 있으며 Final 프로젝트는 Github에 upload

![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/db23bbae-9385-4cd2-bc52-b66dbd0c5eeb)

### Docker Rising 적용

- openjdk 이미지를 베이스로 jar 파일을 실행시킴
- Log 파일 적재를 위하여 /data 폴더 생성

```
# DockerFile
FROM openjdk:15-jdk-alpine
RUN mkdir /data
COPY target/*SNAPSHOT.jar app.jar
EXPOSE 8080
ENV TZ=Asia/Seoul
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ENTRYPOINT ["java","-Xmx400M","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar","--spring.profiles.active=docker"]
```

```
#application.yaml
logging:
  file:
    name: /data/app.log
```

### CodeBuild 내 파이프라인

- 빌드용 서비스이나 post build 프로세스에 배포 스크립트를 적용함
  ![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/6765b8bf-c35a-4b64-9bc1-6ab243ad3396)
  ![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/71005c0c-dfa3-445e-b7d8-fe4c51eaa3ef)

### ECR 내 repo

- post build 스크립트내 만들어진 image를 ECR에 push
  ![image](https://github.com/grapefr/CloudNativeReport/assets/68136339/bb17b713-f10a-4e6b-8b95-c027c41e7769)

### EKS 배포

- 이미지 Push 후 쿠버네티스내 Service와 Deployment를 적용 ( 스크립트 )

```
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - |
        cat <<EOF | kubectl apply -f -
        apiVersion: v1
        kind: Service
        metadata:
          name: $_PROJECT_NAME
          labels:
            app: $_PROJECT_NAME
        spec:
          ports:
            - port: 8080
              targetPort: 8080
          selector:
            app: $_PROJECT_NAME
        EOF
      - |
        cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
```

- [컨테이너 인프라 설계 및 구성 역량](#컨테이너-인프라-설계-및-구성-역량)
  - [컨테이너 자동확장 HPA](#컨테이너-자동확장-HPA)
  - [컨테이너로부터 환경 분리 ConfigMap/Secret](#컨테이너로부터-환경-분리-ConfigMap/Secret)
  - [클라우드스토리지 활용 PVC](#클라우드스토리지-활용-PVC)
  - [셀프 힐링/무정지배포 Liveness/Rediness Probe](#셀프-힐링/무정지배포-Liveness/Rediness-Probe)
  - [서비스 매쉬 응용 Mesh](서비스-매쉬-응용-Mesh)
  - [통합 모니터링 Loggregation/Monitoring](통합-모니터링-Loggregation/Monitoring)

# 컨테이너 인프라 설계 및 구성 역량

체크포인트가 모두 적용된 쿠버네티스 내 default namespace

```
[cloudshell-user@ip-10-132-33-59 ~]$ kubectl get all -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE                                              NOMINATED NODE   READINESS GATES
pod/core-5fb7895678-s7hnr      2/2     Running   0          12h   192.168.82.244   ip-192-168-73-11.ca-central-1.compute.internal    <none>           <none>
pod/gateway-6f895bccf6-x2rkb   2/2     Running   0          14h   192.168.80.60    ip-192-168-73-11.ca-central-1.compute.internal    <none>           <none>
pod/kafka-client               2/2     Running   0          14h   192.168.26.4     ip-192-168-11-240.ca-central-1.compute.internal   <none>           <none>
pod/kafka-controller-0         1/1     Running   0          17h   192.168.50.116   ip-192-168-49-13.ca-central-1.compute.internal    <none>           <none>
pod/kafka-controller-1         1/1     Running   0          17h   192.168.27.200   ip-192-168-11-240.ca-central-1.compute.internal   <none>           <none>
pod/kafka-controller-2         1/1     Running   0          17h   192.168.93.52    ip-192-168-73-11.ca-central-1.compute.internal    <none>           <none>
pod/model-5767f6b645-knj6b     2/2     Running   0          12h   192.168.45.113   ip-192-168-49-13.ca-central-1.compute.internal    <none>           <none>
pod/notice-54c798cf97-9bzq6    2/2     Running   0          12h   192.168.20.98    ip-192-168-11-240.ca-central-1.compute.internal   <none>           <none>
pod/siege                      1/1     Running   0          17h   192.168.39.93    ip-192-168-49-13.ca-central-1.compute.internal    <none>           <none>
pod/target-7c88bd5df7-658jn    2/2     Running   0          10h   192.168.92.229   ip-192-168-73-11.ca-central-1.compute.internal    <none>           <none>

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
service/core                        ClusterIP   10.100.76.89     <none>        8080/TCP                     18h   app=core
service/gateway                     ClusterIP   10.100.161.253   <none>        8080/TCP                     18h   app=gateway
service/kafka                       ClusterIP   10.100.109.201   <none>        9092/TCP                     17h   app.kubernetes.io/instance=kafka,app.kubernetes.io/name=kafka,app.kubernetes.io/part-of=kafka
service/kafka-controller-headless   ClusterIP   None             <none>        9094/TCP,9092/TCP,9093/TCP   17h   app.kubernetes.io/component=controller-eligible,app.kubernetes.io/instance=kafka,app.kubernetes.io/name=kafka,app.kubernetes.io/part-of=kafka
service/kubernetes                  ClusterIP   10.100.0.1       <none>        443/TCP                      34h   <none>
service/model                       ClusterIP   10.100.143.252   <none>        8080/TCP                     18h   app=model
service/my-kafka                    ClusterIP   10.100.179.248   <none>        9092/TCP                     17h   app.kubernetes.io/instance=kafka,app.kubernetes.io/name=kafka,app.kubernetes.io/part-of=kafka
service/notice                      ClusterIP   10.100.234.122   <none>        8080/TCP                     18h   app=notice
service/target                      ClusterIP   10.100.62.89     <none>        8080/TCP                     18h   app=target

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                                                                             SELECTOR
deployment.apps/core      1/1     1            1           18h   core         879772956301.dkr.ecr.ca-central-1.amazonaws.com/core:2fe53bd87a8b275efe799066509c97932175a903      app=core
deployment.apps/gateway   1/1     1            1           18h   gateway      879772956301.dkr.ecr.ca-central-1.amazonaws.com/gateway:12a264aef5dc0cb77149c56e652ce6044b3302cb   app=gateway
deployment.apps/model     1/1     1            1           18h   model        879772956301.dkr.ecr.ca-central-1.amazonaws.com/model:446b7afb98d35eae610d6e0a6c2856bee08bbc9a     app=model
deployment.apps/notice    1/1     1            1           18h   notice       879772956301.dkr.ecr.ca-central-1.amazonaws.com/notice:ba61eb1e00bb557367ad9d3b532770e2fe3ee0af    app=notice
deployment.apps/target    1/1     1            1           18h   target       879772956301.dkr.ecr.ca-central-1.amazonaws.com/target:bfee56e94063c4517973b8e80c3478c088fe5568    app=target

NAME                                 DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                                                                                             SELECTOR
replicaset.apps/core-5fb7895678      1         1         1       12h   core         879772956301.dkr.ecr.ca-central-1.amazonaws.com/core:2fe53bd87a8b275efe799066509c97932175a903      app=core,pod-template-hash=5fb7895678
replicaset.apps/gateway-6f895bccf6   1         1         1       14h   gateway      879772956301.dkr.ecr.ca-central-1.amazonaws.com/gateway:12a264aef5dc0cb77149c56e652ce6044b3302cb   app=gateway,pod-template-hash=6f895bccf6
replicaset.apps/model-5767f6b645     1         1         1       12h   model        879772956301.dkr.ecr.ca-central-1.amazonaws.com/model:446b7afb98d35eae610d6e0a6c2856bee08bbc9a     app=model,pod-template-hash=5767f6b645
replicaset.apps/notice-54c798cf97    1         1         1       12h   notice       879772956301.dkr.ecr.ca-central-1.amazonaws.com/notice:ba61eb1e00bb557367ad9d3b532770e2fe3ee0af    app=notice,pod-template-hash=54c798cf97
replicaset.apps/target-7c88bd5df7    1         1         1       10h   target       879772956301.dkr.ecr.ca-central-1.amazonaws.com/target:bfee56e94063c4517973b8e80c3478c088fe5568    app=target,pod-template-hash=7c88bd5df7


NAME                                READY   AGE   CONTAINERS   IMAGES
statefulset.apps/kafka-controller   3/3     17h   kafka        docker.io/bitnami/kafka:3.6.1-debian-12-r11

NAME                                        REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/model   Deployment/model   3%/50%    1         3         1          13h
[cloudshell-user@ip-10-132-33-59 ~]$
```

### 컨테이너 자동확장 HPA

#### AutoScale을 적용하기 위해서는 쿠버네티스 내 metric서버가 설치되어있어야함

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### Deployment 내 resource 정책 적용 ( 시작시 필요한 최소 CPU 자원 )

```
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/
                  ... 중간 생략 ...
                  resources:
                    requests:
                      cpu: "200m"
```

#### 이후 서비스 Model 에 대하여 AutoScale 적용

```
kubectl autoscale deployment model --cpu-percent=50 --min=1 --max=3
```

#### 동작 확인

1. 부하발생기 설치

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: siege
  namespace: mall
spec:
  containers:
  - name: siege
    image: apexacme/siege-nginx
EOF
```

2. 부하발생기 접속 후 siege 명령어 실행

```
[cloudshell-user@ip-10-132-33-59 ~]$ kubectl exec -it siege -- /bin/bash
root@siege:/#
root@siege:/# siege -c20 -t2S -v "http://gateway:8080/models POST userId=test&state=request&modelName=TestModel1"
** SIEGE 4.0.4
** Preparing 20 concurrent users for battle.
The server is now under siege...
HTTP/1.1 415     0.24 secs:     121 bytes ==> POST http://gateway:8080/models
...중간생략 ...
HTTP/1.1 415     0.09 secs:     121 bytes ==> POST http://gateway:8080/models
HTTP/1.1 415     0.09 secs:     121 bytes ==> POST http://gateway:8080/models

Lifting the server siege...HTTP/1.1 415     0.04 secs:     121 bytes ==> POST http://gateway:8080/models
HTTP/1.1 415     0.05 secs:     121 bytes ==> POST http://gateway:8080/models

Transactions:                    320 hits
Availability:                 100.00 %
Elapsed time:                   1.87 secs
Data transferred:               0.04 MB
Response time:                  0.11 secs
Transaction rate:             171.12 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                   18.26
Successful transactions:           0
Failed transactions:               0
Longest transaction:            0.32
Shortest transaction:           0.00

root@siege:/#
```

3. model POD 의 상태를 모니터링
   ![finalproject2024-02-22 16 41 12 (2)](https://github.com/grapefr/CloudNativeReport/assets/68136339/c4d62b87-0932-4a71-a817-288a6d9376ac)
   ![finalproject2024-02-22 16 41 12 (3)](https://github.com/grapefr/CloudNativeReport/assets/68136339/6905c4bf-f91f-44fc-8552-e97ca77207f3)

## 컨테이너로부터 환경 분리 ConfigMap/Secret

가장 많이 사용하는 패턴인 LogLevel을 ConfigMap에서 가져와 적용

### ConfigMap 생성

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-dev
  namespace: default
data:
  HYBERNATE_TYPE: trace
  LOG_LEVEL: debug
EOF
```

### 배포 스크립트 내 Deployment 수정

```
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  env:
                  - name: HYBERNATE_TYPE
                    valueFrom:
                      configMapKeyRef:
                        name: config-dev
                        key: HYBERNATE_TYPE
                  - name: LOG_LEVEL
                    valueFrom:
                      configMapKeyRef:
                        name: config-dev
                        key: LOG_LEVEL
```

### application.yaml 내 환경변수 적용 ( default 값 적용 )

```
logging:
  level:
    org.hibernate.type: ${HYBERNATE_TYPE:trace}
    org.springframework.cloud: ${LOG_LEVEL:error}
```

### 출력되는 LogLevel 확인

![finalproject2024-02-22 16 41 12 (4)](https://github.com/grapefr/CloudNativeReport/assets/68136339/6fc07223-2bb4-421f-81ad-a89d7644ba94)

## 클라우드스토리지 활용 PVC

컨테이너환경에서 컨테이너의 Data 영역은 컨테이너 라이프사이클과 같이한다. 즉 컨테이너가 삭제되면 Data도 삭제됨. 때문에 컨테이너의 라이프사이클에서 벗어나 자체 라이프사이클을 가진 Persistant Volume 기능이 있음. CSP 에서 제공하는 쿠버네티스의 경우 이 PV를 자사 솔루션과 연동하여 사용할 수 있게끔 각종드라이버의 StorageClass를 제공한다. 해당 StorageClass를 이용하기 위하여 POD에서는 PVC로 볼륨을 Mount함.

### ebs csi driver 설정관련 가이드

- aws 공식 문서외 쿠버네티스 관련 설정도 많지만.. 이번과정에서는 스킵함

https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/csi-iam-role.html
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/managing-ebs-csi.html

### StorageClass 확인

```
[cloudshell-user@ip-10-132-38-3 ~]$ kubectl get storageclass
NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  34h
gp2                kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  35h
[cloudshell-user@ip-10-132-38-3 ~]$
```

### PVC 생성

```
[cloudshell-user@ip-10-132-38-3 ~]$ kubectl get pvc ebs-claim -o yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: ebs-sc
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 4Gi
```

### POD mount 및 Container mount

```
        cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/
                  .... 중간 생략 ...
                    failureThreshold: 5
                  volumeMounts:
                  - name: pvc-vol
                    mountPath: /data
              volumes:
              - name: pvc-vol
                persistentVolumeClaim:
                  claimName: ebs-claim
```

### 적용결과 확인

```
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                  80.0G      6.6G     73.4G   8% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     1.9G         0      1.9G   0% /sys/fs/cgroup
/dev/nvme3n1              3.9G    120.0K      3.8G   0% /data
/dev/nvme0n1p1           80.0G      6.6G     73.4G   8% /etc/hosts
/dev/nvme0n1p1           80.0G      6.6G     73.4G   8% /dev/termination-log
/dev/nvme0n1p1           80.0G      6.6G     73.4G   8% /etc/hostname
/dev/nvme0n1p1           80.0G      6.6G     73.4G   8% /etc/resolv.conf
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                     3.2G     12.0K      3.2G   0% /run/secrets/kubernetes.io/serviceaccount
tmpfs                     1.9G         0      1.9G   0% /proc/acpi
tmpfs                    64.0M         0     64.0M   0% /proc/kcore
tmpfs                    64.0M         0     64.0M   0% /proc/keys
tmpfs                    64.0M         0     64.0M   0% /proc/latency_stats
tmpfs                    64.0M         0     64.0M   0% /proc/timer_list
tmpfs                    64.0M         0     64.0M   0% /proc/sched_debug
tmpfs                     1.9G         0      1.9G   0% /sys/firmware
/ # cd /data
/data # ls -rlth
total 112K
drwx------    2 root     root       16.0K Feb 22 04:46 lost+found
-rw-r--r--    1 root     root       94.6K Feb 22 04:47 app.log
/data #

```

## 셀프 힐링/무정지배포 Liveness/Rediness Probe

쿠버네티스에 POD의 상태체크 방법을 제공함으로서 롤링업데이트시 서비스중단을 방지할 수 있음.

### 무정지배포를 위하여 Rediness Probe 적용

```
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  env:
                  - name: HYBERNATE_TYPE
                    valueFrom:
                      configMapKeyRef:
                        name: config-dev
                        key: HYBERNATE_TYPE
                  - name: LOG_LEVEL
                    valueFrom:
                      configMapKeyRef:
                        name: config-dev
                        key: LOG_LEVEL
                  ports:
                    - containerPort: 8080
                  resources:
                    requests:
                      cpu: "200m"
                  readinessProbe:
                    httpGet:
                      path: /actuator/health
                      port: 8080
                    initialDelaySeconds: 10
                    timeoutSeconds: 2
                    periodSeconds: 5
                    failureThreshold: 10
```

### 배포수행시 POD 상태 확인

![finalproject2024-02-22 16 41 13 (2)](https://github.com/grapefr/CloudNativeReport/assets/68136339/317e6772-9e86-4a65-8b47-7948725853d7)
![finalproject2024-02-22 16 41 13 (3)](https://github.com/grapefr/CloudNativeReport/assets/68136339/7fcd2d80-8dbd-4a1c-a7de-08b144e3ee46)

### API 호출 상태 확인

```
root@siege:/# siege -c10 -t300S -v "http://gateway:8080/models POST userId=test&state=request&modelName=TestModel1"

HTTP/1.1 415     0.07 secs:     121 bytes ==> POST http://gateway:8080/models

... 중간생략 ...

Lifting the server siege...
Transactions:                  55222 hits
Availability:                 100.00 %
Elapsed time:                 299.42 secs
Data transferred:               6.37 MB
Response time:                  0.05 secs
Transaction rate:             184.43 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                    9.45
Successful transactions:           0
Failed transactions:               0
Longest transaction:            2.38
Shortest transaction:           0.00

```

## 서비스 매쉬 응용 Mesh

Mesh 는 MSA환경에서 네트워크 추적이 어려워지고 있는 가운데 트러블슈팅을 쉽게 도와주는 도구임. Mesh를 이용하면 서비스 간 연결을 관리를 위한 모니터링, 로깅, 추적, 트래픽 제어를 수행할 수 있음.

### istio 쿠버네티스 적용

```
export ISTIO_VERSION=1.18.1
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=$ISTIO_VERSION TARGET_ARCH=x86_64 sh -
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo --set hub=gcr.io/istio-release
```

```
# default에 istio injection 적용!
kubectl label namespace tutorial istio-injection=enabled
```

### 이후 SideCar가 적용된 POD 현황 확인

![finalproject2024-02-22 16 41 12 (3)](https://github.com/grapefr/CloudNativeReport/assets/68136339/611a7650-a4cb-4053-be4e-b580de5df038)

### Kiali 확인

![finalproject2024-02-22 16 41 10 (4)](https://github.com/grapefr/CloudNativeReport/assets/68136339/e084c775-88da-4d96-ad8e-b8adcad4f0f0)
![finalproject2024-02-22 16 41 11](https://github.com/grapefr/CloudNativeReport/assets/68136339/b2025627-dcdd-4620-9e79-f1dcf65607ae)

### Jaeger 확인

![finalproject2024-02-22 16 41 10 (3)](https://github.com/grapefr/CloudNativeReport/assets/68136339/605feeb1-7cf6-4309-a1c6-32076e6a1fd3)

- [통합 모니터링 Loggregation/Monitoring](통합-모니터링-Loggregation/Monitoring)

## 통합 모니터링 Loggregation/Monitoring

prometheus와 grafana가 istio 설치시 같이 탑재되었음.

### 모니터링 서비스노출

```
kubectl patch service/prometheus -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'
```

### prometheus 확인

![finalproject2024-02-22 16 41 10 (2)](https://github.com/grapefr/CloudNativeReport/assets/68136339/df22d203-675f-427d-9606-1966734a992f)

### grafana 확인

![finalproject2024-02-22 16 41 10](https://github.com/grapefr/CloudNativeReport/assets/68136339/5b037525-dd33-4349-8fe6-c01aa5b04e5f)

### Loggregation 적용

```
# helm 으로 loke stack 설치
helm install loki-stack grafana/loki-stack --values ./loki-stack-values.yaml -n logging
```

### Logging 확인

![finalproject2024-02-22 16 41 11 (2)](https://github.com/grapefr/CloudNativeReport/assets/68136339/f189d970-06cb-415c-ad49-598485db4331)

# Final Project 소감

1. 소중한 설계, 개발, 배포 경험
1. 쿠버네티스는 필수 ( autoscale )
1. 이벤트스토밍 결과를 코드로 변환해주는 msaez, 생산성
1. 프로젝트에 해당 업무로 참여하고 싶은 열정
