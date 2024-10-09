파드 네트워킹에서 서로 다른 노드의 파드간 통신에서 nat없이 통신하게 해주는것이 cni라고 했다.

### CNI란
- container network interface로 cncf 프로젝트
- 리눅스 컨테이너의 네트워크 인터페이스를 설정할 수 있도록 도와주는 일련의 명세와 라이브러리로 구성
- `컨테이너의 네트워크 연결성`과 `컨테이너 삭제시 관련된 네트워크 리소스 해제`에 대한 책임
- CNI plugin은 IPAM(ip 할당관리) 책임을 가진다. ip주소 할당뿐만 아니라 적절한 라우팅 정보 입력까지 포함
- ![image](https://github.com/user-attachments/assets/6c949136-22e5-4f24-b40f-08b9b5fea70d)

### CNI 요구 사항
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


### CNI 플러그인의 네트워크 모델(feat. calico)
1. overlay network model
- ![image](https://github.com/user-attachments/assets/cb38f0b0-7185-4652-b400-85df82959275)
- 프로토콜 : VXLAN(Virtual Extensible LAN) or IP-in-IP
```
overlay network
오버레이 네트워크는 3계층을 넘어서 구축된 네트워크 간에 있는 엔드포인트의 노드간의 통신이 일어날때 패킷을 한겹 캡슐화 하여 통신시켜서,
2계층에서(같은 LAN에서) 통신이 일어나는 것처럼 통신할 수 있도록 하는 기술이다.
```
- ![image](https://github.com/user-attachments/assets/d5f9bcb3-72cf-4a3d-a9d4-81c0214f16c8)
  - Overlay Network는 기본적으로 패킷을 encapsulation 해서 통신 노드간에 가상 Tunnel을 생성
  - 각 노드에 위치한 가상의 터널의 Endpoint(Virtual Tunnel EndPoint : aka… VTEP)를 통해 캡슐화 된 패킷을 전달하는 방식을 활용하여 복잡한 네트워크 환경에서의 통신을 추상화 한다.
- 오버레이 네트워크를 사용하면 거의 대부분의 환경에서 기존 네트워크 환경에 영향 없이 CNI 환경을 구성할 수 있지만
  - BGP 프로토콜을 사용하는 방식에 비해 패킷을 캡슐화 할때 CPU 등의 자원도 소모하고 패킷의 전체 크기에서 캡슐화를 위해 사용하는 영역 만큼 패킷 당 송/수신 할 수 있는 데이터의 양이 줄어들어 상대적으로 비효율 적인 단점도 있다.


2. non overlay network model
- ![image](https://github.com/user-attachments/assets/57bb0d1c-dd65-44fa-aae3-acac58050417)
- 프로토콜 : BGP

