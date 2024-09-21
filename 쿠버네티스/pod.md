1. pod란
- Pods are the smallest deployable units of computing that you can create and manage in Kubernetes
- 스토리지와 네트워크 리소스, 명세를 공유하는 한 개 이상의 컨테이너의 집합체
- 파드에 속하는 요소들은 co-located & co-scheduled
- 파드에 속하는 구성요소들은 relatively tightly coupled
- 클라우드에선 파드 구성요소들은 동일한 논리적 호스트에서, non-cloud에선 동일한 물리적 호스트에서 동작한다

2. 조금 더 구체적으로
- namespace, cgroup, filesystem volume을 포함한 격리 요소를 공유하는 단위
- 각 파드는 동일한 network namespace 속한다.
  - 그렇기 때문에 파드내 컨테이너는 localhost를 통해 통신한다.
  - 파드에는 private ip가 할당되고 파드내 컨테이너는 이를 함께 활용한다
  - 파드내 컨테이너는 ip space, port space를 공유하기 때문에 동일 port 사용 불가
  - 반대로 말하면 서로 다른 파드의 컨테이너간 통신은 ip address 기반 통신

3. static pod
- 파드 중 api server를 통해 관찰되는것이 아닌 각 노드의 kubelet이 직접 관리
- 각 kubelet은 api server에 mirror pod(A pod object that a kubelet uses to represent a static pod)를 생성한다.
  - 이를 통해 api server에 static pod가 표시되지만 api server를 통해 제어할 순 없다.
  - 즉 스케일링, 업데이트등을 api server를 통해 수행될 수 없다는 것
- 그럼 왜 쓰냐?
  - 노드 단위의 control plane 같은 구성요소를 구성하고 싶을때
  - 혹은 api server 장애 상황에서도 반드시 동작해야하는 중요 서비스
 
4. pod의 전반적인 흐름을 살펴보자
- kubelet은 지속적으로 파드의 state를 api server에게 전달
  - controller manager가 명시된 state와 맞지 않으면 state를 맞추기위해 시도한다.
- 이렇게 파드내 컨테이너의 상태(container state)를 지속적으로 파악하면서 이상이 있는 경우 pod를 healthy하게 만들기 위한 조치를 취한다.
```
container state

- waiting
컨테이너가 running / terminated가 아닌 경우 해당 컨테이너는 waiting 상태
이 상태는 컨테이너 이미지를 가져오거나 secret 데이터를 적용하는 등의 컨테이너 시작을 위한 준비 단계
이 상태에서 kubectl을 통해 waiting 상태인 컨테이너가 포함된 pod를 조회하면 해당 컨테이너가 왜 이 상태에 머무르는지 reason 필드도 나온다.

- running
파드내 컨테이너가 문제 없이 실행되고 있는 상태

- terminated
컨테이너에 문제가 생겨 컨테이너가 종료된 상태
마찬가지로 reason, exit code등을 확인할 수 있다.
```
- pod는 specification(명세)와 actual status가 존재한다.
  - 당연히 pod의 상태를 지속적으로 살펴보며 명세에 맞게 조절하도록한다.
```
pod status

PodScheduled: the Pod has been scheduled to a node.
PodReadyToStartContainers: (beta feature; enabled by default) the Pod sandbox has been successfully created and networking configured.
ContainersReady: all containers in the Pod are ready.
Initialized: all init containers have completed successfully.
Ready: the Pod is able to serve requests and should be added to the load balancing pools of all matching Services.
```
- pod는 생애 한 번만 스케줄링된다.
  - 특정 노드에 할당되어 실행
```
pod의 state는 다음과 같다.

pending :
pod가 스케줄링을 대기하거나 하나 이상의 컨테이너가 아직 실행준비가 완료되지 않은 상태

running :
특정 노드에 바인딩되었으며, 모든 컨테이너가 생성
이 중 최소한 하나 이상의 컨테이너가 실행중

succeeded :
pod내 모든 컨테이너가 성공적 동작

failed :
pod내 모든 컨테이너가 종료
하나 이상의 컨테이너가 fail되어 0이 아닌 exit code로 종료된 상태

unknown :
pod의 상태를 알 수 없는 상태
pod가 실행중이어야할 노드와의 통신 오류가 존재
```

- 만약 파드내 컨테이너 중 하나라도 fail되는 경우 특정 컨테이너를 재시작하려고한다.
  - 파드내 문제가 있는 컨테이너는 다음처럼 다룬다
```
Kubernetes manages container failures within Pods using a restartPolicy defined in the Pod spec
각 상황에 따라 동작하는 방식은 다음과 같다

Initial crash:
Kubernetes는 Pod의 restartPolicy에 따라 즉시 재시작을 시도합니다.

Repeated crashes:
Initial crash 이후 Kubernetes는 이후 재시작 시점에 대해 지수 백오프 지연(exponential backoff delay)을 적용합니다. 이는 시스템이 과부하되는 것을 방지하기 위해 빠르고 반복적인 재시작 시도를 막는 역할을 합니다.

CrashLoopBackOff 상태:
이는 특정 컨테이너가 충돌 루프(crash loop)에 있으며, 계속해서 실패하고 재시작을 반복하고 있을 때 백오프 지연 메커니즘이 현재 적용 중임을 나타냅니다.

Backoff reset:
컨테이너가 일정 기간(예: 10분) 동안 성공적으로 실행되면 Kubernetes는 백오프 지연을 재설정하고, 새로운 충돌을 첫 번째 충돌로 간주합니다.

```
- 파드는 클러스터가 복구할 수 없는 방식으로 fail될 수 있다. 이 경우 쿠버네티스는 더 이상 해당 파드를 heal하려고 시도하지 않고, pod를 삭제하여 automatic healing 될 수 있도록한다.
- pod가 노드에 스케줄링된 후에 노드가 fail되면 파드는 unhealthy로 판단되고 파드를 삭제한다. 즉 리소스 부족이나 유지보수로 인해 발생하는 eviction(강제 종료)에서는 파드가 생존할 수 없다.
- 파드는 기본적으로 다른 노드로 재스케줄링되지 않는다. 다만 리소스 부족, 노드 이상시 near-identical Pod를 재생성하여 다른 노드등으로 배치될 수 있게 한다.
  - 재생성이란 결국 기존과 다른 파드다.
  - 동일 파드가 아니니 다른 노드로 재스케줄링되지 않는다에 해당하지 않는다.

5. Associated lifetimes
- 파드와 연관된 볼륨등 파드와 함께 라이프 사이클을 가져야하는 것들이 존재한다.
- 다시말해 파드가 종료되면 같이 제거되어야하고, 파드가 재생성되면 다시 생성된다.

