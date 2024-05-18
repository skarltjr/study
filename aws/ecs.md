1. ECS란
- elastic container service로 "컨테이너 오케스트레이션 툴" like k8s

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


------------
### ecs의 구조적 특징
1. single control plane
- ![ecs-layers](https://github.com/skarltjr/study/assets/62214428/04f19856-90fa-461f-ac2b-a09704a2629e)
- ecs의 경우 단일 control plane(스케줄러)이 인스턴스들의 스케줄을 관리하는 중앙 집중식이다 / 쿠버네티스의 경우 여러 컨트롤 플레인이 존재하는 아키텍처를 따른다

2. docker host
- ![image](https://github.com/skarltjr/study/assets/62214428/c5dcf71e-2ccf-4161-9543-1624872a1ff5)
- 일반적으로 컨테이너는 vm에 비해 더 높은 성능과 효율성을 장점으로 말한다.
- 그런데 ecs는 컨테이너 전용 서버들을 구성한게 아니라 vm과 같은 ec2 instance 위에서 컨테이너를 동작시킨다 / vm위에서 컨테이너를 동작시킨다?? => 컨테이너의 장점이 사라지는거아니야?
- 기존 ec2 기반 생태계(Network 서비스인 VPC와 ELB, EIP 에서 부터 Auto scaling)와의 연계의 장점을 얻을 수 있었다
- 반대로 얘기하면 완전한 docker 기반 ochestration은 아니며 그에 따른 단점은 존재한다

2-1. Overlay network 가 아닌 ELB 를 이용한 Network 구조
- 먼저 underlay network란 실제 물리 장비를 통해 연결된 네트워크다. 여기서 이웃 노드는 실제로 물리 장비로 연결된 노드들이다.
- 반면 overlay network이란 이런 기존 네트워크위에 논리적으로 링크들로 구성된 가상네트워크다. 여기서 이웃 노드란 실제 물리적으로 연결된 노드가 아니라 논리적으로 연결된 노드들이다.
- 서비스 규모가 커지면서 오케스트레이션에서 생기는 고민은 서로 다른 호스트에 위치한 컨테이너간 통신이었다. 그리고 이걸 overlay network로 극복했다.
- 하지만 ecs는 다르다. ecs는 overlay network이 아닌 elb를 활용한다.
- 즉 ecs는 서로 다른 host에 위치한 컨테이너간 통신에서 (컨테이너!에 할당된 private ip를 통한 통신이 아니라) host에 할당된 private ip를 통해 호스트에 도달하고 port를 통해 접근한다.
- ![image](https://github.com/skarltjr/study/assets/62214428/de26508e-7049-40d1-ab56-a001c714cdca)
- ecs 한계 : 같은 서비스에 속하지만 서로 다른 host에 떠있는 task간 통신은 어려울 것.



-------
### ecs 네트워크
1. awsvpc 네트워크 모드
- ecs 태스크에 ec2와 동일한! 네트워킹 속성을 제공
- 각 태스크는 고유한 ENI(ip,mac)를 받으므로 VPC Flow Log와 같은 기타 EC2 네트워킹 기능을 이용하여 태스크에서 주고받는 트래픽을 모니터링 가능
- 각 ENI는 분리되어 있으므로 각 컨테이너는 포트에 바인딩 가능 :80 → 포트 번호 추적할 필요 없음
- aws 관리형
- ![image](https://github.com/skarltjr/study/assets/62214428/d959f97b-a114-44fa-9d30-65a963e340a5)
