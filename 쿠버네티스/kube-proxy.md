1. kube-proxy란
- kube-proxy는 서비스 정의를 네트워킹 규칙으로 변환하는 쿠버네티스 에이전트
```
kube-proxy는 클러스터의 각 노드에서 실행되는 네트워크 프록시로, 쿠버네티스의 Service 개념의 구현부이다.

kube-proxy는 노드의 네트워크 규칙을 유지 관리한다. 이 네트워크 규칙이 내부 네트워크 세션이나 클러스터 바깥에서 파드로 네트워크 통신을 할 수 있도록 해준다.

kube-proxy는 운영 체제에 가용한 패킷 필터링 계층이 있는 경우, 이를 사용한다. 그렇지 않으면, kube-proxy는 트래픽 자체를 포워드(forward)한다.
```
- 어떻게 서비스-파드간 네트워킹 매핑이 일어나는가? 쿠베 프록시 어떻게?

2. 동작 방식
- 새로운 서비스나 엔드포인트가 추가 및 삭제되면 api server는 이를 kube-proxy에게 전달
- kube-proxy는 이러한 변화를 노드의 NAT 규칙에 반영
  - nat : 서비스의 ip와 파드의 ip를 매핑
- 예를들어
  - service 생성
  - service와 관련된 파드(selector와 일치하는)를 찾는다
  - Api 서버는 Endpoint라는 추상화를 생성한다
    - 엔드포인트는 서비스가 트래픽을 전달하고자 하는 파드의 집합
  - 여기까지는 컨트롤 플레인에서 추상적인 레벨의 매핑이고 실제 네트워크 구성이 필요할텐데, 이를 Kube-proxy가 담당한다.
  - kube-proxy는 각 노드에 위에 생성한 규칙을 네트워크 레벨에서 적용한다.
  - ![image](https://github.com/user-attachments/assets/ed62d6b8-438c-4b57-9d48-938f03ac86ac)
  - 이제 SVC01로 향하던 트래픽은 DNAT(destination nat) 규칙을 따라가게 되고 파드로 연결
  - ![image](https://github.com/user-attachments/assets/0a4732cc-890c-4984-b800-ba8c4e5a7ac3)


3. 추가
- 서비스와 엔드포인트는 ip뿐만 아니라 port도 매핑
- 위 예시에서 DNAT 변환이 source node에서 수행되었다.
  - 이는 cluster ip type service를 활용했기 때문
  - 다시말해 clusterIp 타입 서비스는 소스 노드에서 DNAT가 수행되고, 내부 NAT 규칙이기 때문에 클러스터 내부에서만 액세스
- NAT 규칙은 파드 중 하나를 랜덤하게 선택.

4. kube-proxy mode
- iptables mode
  - default
  - IPtables는 내부 패킷 처리와 필터링 컴포넌트로 동작
  - 들어오고 나가는 트래픽을 검사, 그런 다음 특정 기준과 일치하는 패킷에 대해 특정 규칙을 적용
  - 이 모드를 사용하면 kube-proxy는 서비스에서 파드로의 NAT 규칙을 IPtables에 주입하고, 이를 통해 서비스 IP로 들어온 트래픽이 파드의 IP로 변환(NAT)되어 리다이렉
  - ![image](https://github.com/user-attachments/assets/87b01e68-73ea-45d9-8daa-707b8c169f51)
  - 다만 이 방식의 단점은, 서비스와 엔드포인트 개수가 많아질수록 시간복잡도 상승.
    - 순차적으로 규칙을 확인하기 때문
  - 또한 로드밸런싱이 아닌 랜덤 분배


