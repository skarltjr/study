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
container status

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
- pod는 생애 한 번만 스케줄링된다.
  - 특정 노드에 할당되어 실행
```
pod의 status 다음과 같다.

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
- pod는 specification(명세)와 위처럼 actual status가 존재한다.
  - 당연히 pod의 상태를 지속적으로 살펴보며 명세에 맞게 조절하도록한다.
  - 이때 pod가 어떤 조건에 해당하는지에 따라 자세히 분류된다.
```
pod condition

PodScheduled: the Pod has been scheduled to a node.
PodReadyToStartContainers: (beta feature; enabled by default) the Pod sandbox has been successfully created and networking configured.
ContainersReady: all containers in the Pod are ready.
Initialized: all init containers have completed successfully.
Ready: the Pod is able to serve requests and should be added to the load balancing pools of all matching Services.
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

6. pod readiness
- 컨테이너의 준비 상태를 파악할 수 있는 추가 요소
- 각 목적에 맞는 probe(조사 방식)가 존재한다
- livenessProbe
  - 컨테이너 실행 여부를 나타낸다.
  - liveness probe가 실패하면 kubelet은 해당 컨테이너를 종료하고 재시작 정책에 따라 처리된다.
  - 컨테이너가 liveness probe를 제공하지 않으면 기본적으로 Success다
- readinessProbe
  - 컨테이너가 request에 응답할 준비가 되었는지 판단한다.
  - readiness probe가 실패하면 엔드포인트 컨트롤러는 Pod의 IP 주소를 해당 Pod와 일치하는 모든 서비스의 엔드포인트에서 제거
  - 마찬가지로 컨테이너가 readiness probe를 제공하지 않으면 기본적으로 Success다
```
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example-container
    image: your-image
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```
- startupProbe
  - 컨테이너 내 애플리케이션 시작되었는지 판단할 수 있는 요소
  - startupProbe가 존재하면 다른 모든 probe는 이 probe이 성공할때까지 비활성화
  - 만약 startupProbe가 실패하면 컨테이너 종료 후 재시작 정책에 따라 처리
  - 마찬가지로 컨테이너가 startupProbe를 제공하지 않으면 기본적으로 Success

7. container probe
- 그렇다면 readiness를 위해 컨테이너 상태를 파악할 container probe는 어떻게 구성되는가
- check mechanism은 다음과 같다
```
exec: 지정된 명령어를 컨테이너 내부에서 실행합니다. 명령어가 0 상태 코드로 종료되면 진단은 성공한 것으로 간주됩니다.

grpc: gRPC를 사용하여 원격 프로시저 호출을 수행합니다. 대상은 gRPC 상태 체크를 구현해야 합니다. 응답의 상태가 SERVING이면 진단은 성공한 것으로 간주됩니다.

httpGet: 지정된 포트와 경로에서 Pod의 IP 주소로 HTTP GET 요청을 수행합니다. 응답의 상태 코드가 200 이상 400 미만이면 진단은 성공한 것으로 간주됩니다.

tcpSocket: 지정된 포트에 대해 Pod의 IP 주소로 TCP 체크를 수행합니다. 포트가 열려 있으면 진단은 성공한 것으로 간주됩니다. 원격 시스템(컨테이너)이 연결이 열리자마자 바로 닫아도 이는 정상으로 간주됩니다.
```
- outcome은 다음과 같다
```
Success
The container passed the diagnostic.

Failure
The container failed the diagnostic.

Unknown
The diagnostic failed (no action should be taken, and the kubelet will make further checks).
```

8. pod의 종료
- kill signal로 갑작스런 중단이 아닌 gracefully terminate를 목표한다.
- 목표는 종료 요청을 할 수 있으며, 프로세스 종료 시점을 알 수 있도록하며, 최종적으로 삭제가 완료됨을 보장하는것이다.
- kubelet은 컨테이너 런타임에 종료를 요청한다
  - 이때 각 컨테이너 주요 프로세스에 sigterm을 보낸다.
    - sigkill은 프로세스의 자식 프로세스가 존재하는 경우에도 바로 죽여서 고아 프로세스가 남아있을 수 있다.
  - https://github.com/skarltjr/study/blob/main/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4/%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88.md#:~:text=container%20runtime%20interface
- 종료의 흐름은 다음과 같다
```
- pod 삭제 요청 : kubectl을 통해 특정 pod 제거 요청
- api server 업데이트 : API 서버의 Pod는 Pod가 "죽은" 것으로 간주되는 시간과 유예 기간으로 업데이트 / `Terminating`으로 변경
- kubelete의 종료 프로세스 시작 : pod가 Terminating으로 변경되면 kubelet은 로컬 pod 종료 프로세스를 시작
  - 컨테이너 런타임을 통해 각 컨테이너 1번 프로세스에 sigterm 전달
- service endpoint 업데이트 : kubelet이 pod의 graceful shutdown을 시작하는 동시에 control plane은 pod를 endpoint object에서 제거할지 결정
  - Pods that shut down slowly들은 일반 트래픽이 전달되어선 안된다.
  - 이러한 pod들은 connection 정리, session drainign이 먼저 수행된다.
- 하지만 우아한 종료를 위한 grace period 이후에도 컨테이너가 남아있다면, sigkill로 강제 제거  
```
---
init container
- 잠깐! Initialized에서 말하는 init containers란 무엇일까?
  - Init container는 파드의 컨테이너 내부에서 애플리케이션 container가 실행되기 전에 초기화를 수행하는 컨테이너이다.
  - 먼저 필요한 스크립트등을 수행한다던가의 목적으로 사용될 수 있다.
```
apiVersion: v1
kind: Pod
metadata:
 name: init-container-example
spec:
 initContainers: # 초기화 컨테이너
 - name: my-init-container
   image: busybox
   command: ["sh", "-c", "echo Hello world!"]
 containers: # 에플리케이션 컨테이너
 - name: nginx
   image: nginx
```
  - n개의 init container가 정상 실행 후 container 실행이 시작된다.
  - init container는 순차적으로 실행되며, 만약 init container 실행이 실패하면 kubelet은 재시작하고자한다.
  - 그러나 restart policy가 never인 경우 k8s는 파드가 실행에 실패했다고 판단한다.
- 그럼 일반 container와의 차이가 무엇인가?
  - 
