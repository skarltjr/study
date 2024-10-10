파드 네트워킹에서 서로 다른 노드의 파드간 통신에서 nat없이 통신하게 해주는것이 cni라고 했다.

### CNI란
- container network interface로 cncf 프로젝트
- 리눅스 컨테이너의 네트워크 인터페이스를 설정할 수 있도록 도와주는 일련의 명세와 라이브러리로 구성
- `컨테이너의 네트워크 연결성`과 `컨테이너 삭제시 관련된 네트워크 리소스 해제`에 대한 책임
- CNI plugin은 IPAM(ip 할당관리) 책임을 가진다. ip주소 할당뿐만 아니라 적절한 라우팅 정보 입력까지 포함
- ![image](https://github.com/user-attachments/assets/6c949136-22e5-4f24-b40f-08b9b5fea70d)

### CNI 요구 사항
쿠버네티스 자체에는 네트워크 구성 기능이 존재하지 않는다. 
따라서 CNI를 통해 `컨테이너의 네트워크 연결성`과 `컨테이너 삭제시 관련된 네트워크 리소스 해제` 및 IPAM에 대한 책임을 수행한다.
- veth 페어 생성 및 컨테이너 네트워크 인터페이스와 연결
- Pod 네트워크 대역 확인 후 IP 설정
- CNI 설정 파일 작성
- IP 설정 및 관리
- 컨테이너 내 기본 라우팅 정보 삽입 (default route rule)
- 동료 노드들에게 IP 라우팅 정보 전달(advertising the routes)
- 호스트 서버에 라우팅 정보 삽입
- 네트워크 정책에 따라 트래픽 처리

### CNI component
- ![image](https://github.com/user-attachments/assets/313f5950-e02e-40b4-9ddf-cc6b18794ff6)
- main  - 특정 네트워크 기능을 구현하는 메인 플러그인
- meta  - 자체적으로 특정 네트워크 기능을 제공하지 않으며, 다른 플러그인을 호출하거나 단순히 테스트 용도로 사용됩니다.
- ipam - IP 주소 할당용 플러그인

### Calico는 어떻게 CNI 요구 사항을 만족할까
- ![image](https://github.com/user-attachments/assets/f603358b-c08f-40bf-9a74-cd05ff759daa)
1. BIRD(BGP)
- BIRD는 각 노드마다 존재하는 BGP 데몬
  - 서로 다른 노드에 존재하는 BGP 데몬들과 라우팅 정보를 교환
  - 라우팅 정보 교환은 네트워크 변경이 생기는 경우 노드에 떠 있는 에이전트인 BIRD 데몬이 이를 감지하여 `라우팅 정보를 다른 peer로 announce`
  - 네트워크 구성(topology)는 full-mesh와 route reflector mode가 존재
    - full-mesh의 경우 모든 노드가 전부 서로 피어링
    - route reflector mode의 경우 대규모 클러스터에서 각 노드가 1 or n개의 중앙 Route Reflector node와만 피어링
2. ConfD
- confd는 calico-node 컨테이너 안에서 동작하는 간단한 설정관리 툴
- 데이터 저장소(ex. etcd)로부터 BIRD 설정값을 읽어들이고 디스크 파일로 쓰기 작업도 수행
- 네트워크와 서브네트워크에 설정값을 반영하고(CIDR 값) BIRD 데몬이 이해할 수 있도록 설정값들을 변환
- 그래서 네트워크에 어떤 변화가 생겼을때 BIRD 데몬이 이를 감지하여 라우팅 정보를 다른 peer로 전파 가능
3. Felix
- felix 데몬도 calico-node 컨테이너 안에서 동작
  - 쿠버네티스 etcd로부터 정보를 읽는다
  - 라우팅 테이블을 생성한다
  - iptable을 조작한다(kube-proxy가 iptables 모드인 경우)
  - ipvs 조작(kube-proxy ipvs mode)
- kube-proxy와는 다른것인가?
  - 파드가 생성, 재생성되는 경우 ipam을 통해 ip를 얻고 confd가 설정을 업데이트하며, bird가 각 노드에 있는 데몬에 전파하고, felix가 각 노드(호스트) os 라우팅 테이블에 업데이트하여 일관된 네트워크 상태 유지
  - kube-proxy는 특정 service가 생성되었을때 이에 맞는 정보를 각 노드 라우팅 테이블에 업데이트
### proxy-arp
- ![image](https://github.com/user-attachments/assets/bb5d422a-73dc-44c3-9d0e-c1f18e0dc5a5)
- 사진을 보면 두 노드의 서로 다른 네트워크 인터페이스는 veth를 통해 연결되어 있지 않다.
  - 즉 패킷을 전달할 네트워크 인터페이스의 mac 주소를 알 수 없는 상태이다.
  - 그런데 어떻게 다른 노드로 패킷이 라우팅 되는가?
- proxy-arp
- 마스터 노드의 파드에서 워커 노드의 파드인 10.0.2.11로 가고 싶다. 어떻게?
- 마스터 노드를 살펴보면 컨테이너의 네트워크 인터페이스의 169.254.1.1은 호스트 네트워크 인터페이스와 연결되어 있다.
  - 컨테이너의 자신이 연결된 네트워크 인터페이스(파드 네트워크 인터페이스) -> 게이트웨이(호스트 네트워크 인터페이스)를 통해 외부로 나갈 수 있다고 생각
  - 그리고 외부로 나가려할때 해당 주소(169.254.1.1 게이트웨이인 호스트 네트워크 인터페이스)에게 대한 arp 요청을 보낼 것
  - 그럼 arp 응답은 누가 주는가? proxy-arp
```
“Proxy ARP란, 해당 네트워크에 존재하지 않는 proxy device가 ARP 요청을 대리하여 응답하는 기술을 말합니다.
Proxy는 목적지의 실제 위치를 알고 있습니다(예시에서는 proxy가 10.0.2.11의 위치를 이미 알고 있다는 뜻입니다.)
그렇기에 ARP 요청이 왔을 때, 본인의 MAC주소를 대신 전달해주어 ARP 요청자로 하여금 자신에게 패킷이 전달되도록 응답합니다.
Proxy로 전달된 트래픽은 Proxy에 의해 실제 목적지로 전달됩니다.
이때 사용되는 인터페이스가 tunnel입니다.
이렇게 프록싱을 목적으로 ARP 요청에 대해 자신의 MAC 주소를 대신 응답하는 프로세스를 퍼블리싱(publishing)이라고도 부릅니다.”
```
- 이후 워커노트 커널로 패킷이 도착
- 커널이 칼리코 인터페이스로 패킷을 전달하고 이 인터페이스가 `네트워크 정책에 따라 트래픽 처리`

### CNI 플러그인의 라우팅 모드(feat. calico)
1. IP in IP(default)
- ![image](https://github.com/user-attachments/assets/cb38f0b0-7185-4652-b400-85df82959275)
```
먼저! overlay network
오버레이 네트워크는 3계층을 넘어서 구축된 네트워크 간에 있는 엔드포인트의 노드간의 통신이 일어날때 패킷을 한겹 캡슐화 하여 통신시켜서,
2계층에서(같은 LAN에서) 통신이 일어나는 것처럼 통신할 수 있도록 하는 기술이다.
```
- ![image](https://github.com/user-attachments/assets/d5f9bcb3-72cf-4a3d-a9d4-81c0214f16c8)
  - Overlay Network는 기본적으로 패킷을 encapsulation 해서 통신 노드간에 가상 Tunnel을 생성
  - 각 노드에 위치한 가상의 터널의 Endpoint(Virtual Tunnel EndPoint : aka… VTEP)를 통해 캡슐화 된 패킷을 전달하는 방식을 활용하여 복잡한 네트워크 환경에서의 통신을 추상화 한다.
- 오버레이 네트워크를 사용하면 거의 대부분의 환경에서 기존 네트워크 환경에 영향 없이 CNI 환경을 구성할 수 있지만
  - BGP 프로토콜을 사용하는 방식에 비해 패킷을 캡슐화 할때 CPU 등의 자원도 소모하고 패킷의 전체 크기에서 캡슐화를 위해 사용하는 영역 만큼 패킷 당 송/수신 할 수 있는 데이터의 양이 줄어들어 상대적으로 비효율 적인 단점도 있다.
- ![image](https://github.com/user-attachments/assets/09886bf8-b8a6-4c7c-abda-8244b0c32030)
- 결론적으로 위 그림이 된다.

2. VXLAN
- ![image](https://github.com/user-attachments/assets/42985a7c-540b-4efd-966d-170d268bf54a)
 

3. non overlay network model(DIRECT)
- ![image](https://github.com/user-attachments/assets/a7fe9b35-ed10-4b16-a0c9-77cd9c8ee9a4)
```
이 모드에서는 마치 Pod로부터 직접 전송된 것처럼 패킷을 전달합니다.
캡슐화와 디캡슐화가 수행되지 않기 때문에 성능면에서 우수한 장점을 가집니다.
AWS에서는 Source IP check 기능을 반드시 비활성화 해야 합니다.
AWS EC2에는 패킷의 source IP가 패킷을 전송하는 호스트 IP와 동일한지 확인하는 기능이 있습니다.
NoEncapMode에서는 source IP가 호스트 IP가 아닌 전송하는 Pod IP로 찍히기 때문에 해당 기능을 비활성화 하지 않으면 패킷이 전달되지 않습니다.
```

--------
### eks ipam
- ![image](https://github.com/user-attachments/assets/e853a0b3-f92a-4064-8484-8b277e019d10)
- 기본적으로 eks는 vpc cni를 활용한다.
  - aws vpc를 활용함으로써 각 노드는 노드가 위치한 서브넷의 하위 대역 ip를 할당 받는다.
    - 그렇기 때문에 대역 충돌이 발생할 가능성이 매우 낮다.
    - vpc cni의 장점은 파드가 같은 vpc내 ip를 할당받기 때문에 overlay network가 필요없다
    - calico 같은 경우 Ip in Ip가 디폴트로 패킷 en-de capsulation이 기본
    - 반대로 말하면 vpc native한 구성은 이기종 시스템에서 nat없이 직접 통신등의 overlay network를 통해 얻을 수 이점을 가질 수 없다고 생각.
  - vpc cni에서는 eni를 기반으로 ip를 할당한다.
    - 각 인스턴스(노드)는 타입에 따라 n개의 eni를 가질 수 있으며 각 eni는 n개의 private ip를 가질 수 있다.
    - 인스턴스가 가질 수 있는 private ip 수 `(ENI 개수 × (ENI 당 IP Address 개수 - 1)) + 2`
    - 예를들어 `a1.large`의 경우 최대 29개의 private ip를 가질 수 있겠다.
      - EC2 인스턴스가 가질 수 있는 최대 ENI 개수 = 3
      - EC2 인스턴스의 ENI가 가질 수 있는 최대 Private IP Address 개수 = 10
  - 조금 더 구체적으로 `L-IPAM(Local IP Address Manager)` 방식을 사용하는데
    - l-ipam은 노드(인스턴스)의 메타데이터를 확인하여 private ip를 미리 준비하는데 이를 warm pool이라고 한다.
    - 그리고 파드 생성 시점에 warm pool에서 ip를 가져와 실제 파드에 할당하고 기록하는것이 slot이다.
    - primary eni의 primary ip가 노드의 private ip가 되며
    - 각 eni의 secondary ip가 파드에 할당될 ip들이 된다.
    - ![image](https://github.com/user-attachments/assets/c42ce192-41dc-4b12-ba69-89ac31a54556)

