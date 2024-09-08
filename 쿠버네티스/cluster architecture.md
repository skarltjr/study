![image](https://github.com/user-attachments/assets/260f6ba0-b37b-414f-91bc-3f75eb9f8c61)

1. 쿠버네티스 클러스터
- control plane과 애플리케이션을 실행하는 워커 노드로 구성된다
- worker node는 application workload의 구성요소인 pod를 호스팅한다.
- control plane은 클러스터내 worker node와 pod들을 관리한다

2. control plane components
- control plane의 구성요소는 클러스터에 관한 global decision을 내린다 / ex. scheduling
- 또한 클러스터내 이벤트를 감지하고 대응한다. / ex. deployment의 replicas 갯수가 충족되지 않을때 새로운 추가 pod를 띄운다

2-1. control plane components / kube-apiserver
- control plane의 api server는 쿠버네티스 api를 노출하는 control plane의 front end다
- 수평 확장 가능하도록 설계되어있어서 여러 api server 인스턴스를 실행할 수 있다.

2-2. control plane components / etcd
- highly-available key value store로 모든 클러스터내 데이터를 저장한다

2-3. control plane components / kube-scheduler
- 새로 생성된 파드를 어떤 노드에 배치할지 결정한다.
- 노드를 어디에 배치할지 결정할땐 다음 요소들을 고려한다.
  - hardware/software/policy constraints
  - affinity and anti-affinity specifications
  - data locality
  - inter-workload interference
  - deadlines.

2-4. control plane components / kube-controller-manager
- controller = control loop(제어루프)로써 api server를 통해 클러스터내 shared state를 파악하고 현재 상태를 desired state로 변경한다.
- 논리적으로 control loop는 모두 개별 프로세스이지만, 복잡성을 줄이고자 하나의 바이너리로 컴파일되어 단일 프로세스에서 실행된다.
- controller의 종류는 다양하다
  - node controller : 노드의 down을 알아채고 대응
  - job controller : one-off task(일회성 작업)을 위한 job object들을 관찰하고 이를 실행하기 위한 파드를 생성한다.
  - endpointSlice controller : 쿠버네티스 클러스터에서 서비스와 파드(Pod) 간의 네트워크 연결을 효율적으로 관리하는 데 중요한 역할을 하는 endpointSlice 객체를 생성한다.
  - serviceAccount controller : 새로운 네임스페이스에 대한 기본 serviceAccounts를 생성한다
  - 그외에도 존재한다.

3. node components
- node components는 모든 노드에서 실행되며 실행중인 pod를 관리하고 쿠버네티스 런타임 환경을 제공한다

3-1. kubelet
- 각 노드에서 실행되는 node agent로써 hostname or hostname override flag를 활용하여 api server를 통해 노드를 등록할 수 있다.
- kubelet은 pod에 대해 기술하는 podSpec(yaml or json)을 통해 동작한다
- api server를 통해 전달받은 podspec을 가져와서 스펙에 정의된 컨테이너가 실행중인지, healthy한지 확인한다.
- 쿠버네티스를 통해 생성되지 않은 컨테이너는 관리하지 않는다.
- api server외에도 kubelet에게 컨테이너 매니페스트를 제공하는 방법은 크게 2가지가 있다
  - file or http endpoint / `kubelete [flags]`
 
3-2. kube-proxy
- 클러스터내 각 노드에서 실행되는 네트워크 프록시로 k8s service conecpt 구현의 일부다
- 노드의 네트워크 규칙을 관리하며, 클러스터 내-외부 네트워크 세션으로부터 pod로의 네트워크 통신을 허용한다.
- 운영체제 패킷 필터링 계층이 있으면 이를 활용하고 아니면 트래픽 자체를 전달한다.
- kubernetes api에 정의된 services들을 각 노드에 반영하며,
- 백엔드간 tcp udp, sctp stream을 포워딩 혹은 라운드로빈 포워딩한다.

3-3. container runtime
- 쿠버네티스가 컨테이너를 실행하기위한 컴포넌트
- 쿠버네티스 환경에서 컨테이너 라이프사이클을 매니징한다.
- containerd, cri-o등을 지원한다.

4. addons
- addon은 deployment등 쿠버네티스 리소스를 활용하여 클러스터의 기능을 구현한다.
- 클러스터 레벨의 기능을 구현하는 구현체로 kube-system namespace에 종속된다

5. DNS
- 다른 addon은 필수가 아니지만 dns는 필수 addon이다.
- 모든 클러스터는 cluster dns를 갖는다
- cluster dns는 dns server이자 쿠버네티스 서비스들에 대한 dns 레코드를 제공하는 dns server이다.

