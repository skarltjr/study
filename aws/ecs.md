1. ECS란
- amazon container service로 "컨테이너 오케스트레이션 툴" like k8s

2. ECS의 구성요소
- <img width="733" alt="스크린샷 2023-04-12 오전 11 25 16" src="https://user-images.githubusercontent.com/62214428/231331688-4d3b3b15-135a-4371-8fed-d47790582b70.png">

- Task Definition
- Task
- Service
- Container Instance
- Cluster

#### Task Definition
- 컨테이너 생성시 설정을 정의
  - 쿠버네티스에서의 deployments의 yaml과 비슷
  - 이미지 종류, cpu/메모리와 같은 리소스, port, volume 매핑 설정 등


#### Task
- Task Definition을 통해 생성된 container set을 task라고 한다.
  - 쿠버네티스에서의 Pod 혹은 docker-compose와 비슷한 개념
  - task안에는 한 개 이상의 컨테이너들이 포함
  - ecs에서 컨테이너를 실행하는 최소 단위
  - Task를 실행하는 방법에는 task definition으로 직접 task를 실행하는 방법 & service를 정의하는 방법이 존재
    - 단, task definition으로 직접 실행하는경우 최초 실행 이후 더 이상 관리되지않는다.

#### Service
- Task들의 life cycle을 관리
  - 쿠버네티스의 서비스는 파드를 외부로 노출시키기 위한 수단으로 ecs의 service와 차이가 좀 있다.
```
Service는 Task Definitions를 이용하여 Task 실행 유형(EC2 or Fargate), 타입(복제 or 데몬), 
테스크가 실행 되어야할 작업 개수, 배포 방식(롤링 or Blue Green), 배치 전략, AutoScale 설정 및 컨테이너와 ELB(ALB or NLB)등을 
설정하여 Task를 실행 및 관리 하도록 도와준다.
```


- 서비스에서 실행 유형은 ec2, fargate 두 가지가 존재.
- <img width="948" alt="스크린샷 2023-04-12 오전 11 34 42" src="https://user-images.githubusercontent.com/62214428/231333218-fe180921-2780-4a16-a0e0-09b6f09c85c9.png">
```
- hosting : 컨테이너를 위한 컴퓨팅 리소스
EC2 : vm으로 독립된 환경이 있고 개별 운영체제를 갖는 컴퓨팅 리소스
Fargate : EC2보다 더 추상화된 컴퓨팅 환경. 기본 인프라 관리할 필요 없는 EC2의 서버리스 버전
```
```
서비스에서 타입은 replica와 daemon 두 가지가 존재.
- replica : 복제는 서비스에 정의된 작업 개수 및 AutoScale 설정에 따라 클러스터 인스턴스에 Task가 복제하여 실행하는 방식
- daemon : 클러스터내 모든 ecs 인스턴스에 task가 무조건 하나씩 실행
```

#### Cluster
- 클러스터는 컨테이너 인스턴스(EC2)의 논리적인 그룹화 이다.
- 논리적인 공간이므로, 컨테이너 인스턴스가 없는 빈 클러스터도 생성이 가능하다.
- ECS Agent를 통해 논리적인 클러스터에 연결된다.
