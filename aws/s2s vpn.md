aws
1. s2s vpn
- IPSec 암호화 프로토콜을 사용해 AWS Cloud 환경과 On-Premise 환경을 연결해주는 서비스다
- AWS 클라우드와 On-Premise 데이터 센터가 있다. AWS에는 VGW(Virtual Private Gateway)가 있고 On-Premise에는 CGW(Customer Gateway)가 붙어있으며 이 둘 사이에 IPSec 프로토콜을 이용해 터널링을 만들어 줘 인터넷을 통해서 상호 간 통신이 가능하게끔 해주는 원리다.
- ![img1 daumcdn](https://github.com/skarltjr/study/assets/62214428/0d9c7e46-a381-4074-8d5f-e33019e676a6)

2. virtual gateway(vgw)
- VGW는 AWS Cloud VPC의 라우터
- 1 vgw : n on-prem
- 1 vgw : 1 vpc

3. customer gateway(cwg)
- CGW는 On-Premise의 라우터 값을 AWS에 제공해주는 서비스
- On-Premise 내부적으로 사용하는 Private IP주소 대역이 있을 텐데 VPN에 사용할 사설 IP대역을 CGW에 지정하고 VGW는 이 값을 참조해 연결하는 것이다.
- 헷갈릴 수 있는데 CGW는 AWS에서 구성되는 가상 게이트웨이이며 이를 위해 On-Premise 쪽에 별도의 VPN 장비를 준비해야 한다.
- 모든 준비가 완료되면, AWS 쪽에서 VPN 라우팅을 위한 구성 파일을 다운로드하여 On-Premise 쪽의 VPN 장비에 설치를 해줘야 비로소 통신이 연결된다.

4. autonomous system / asn
-  AWS VPC를 위한 ASN이 있고 On-Premise 장비 쪽에도 ASN이 있다
-  ![img1 daumcdn-1](https://github.com/skarltjr/study/assets/62214428/a666f312-c72e-4627-8e54-7c84ceca92c1)
- 각각의 네트워크 그룹은 Interior Gateway(OSPF etc... ) 프로토콜을 이용해 라우팅 테이블을 내부적으로 공유해 자기들끼리는 통신이 가능하다. 그리고 이 둘의 라우팅 테이블을 교환하기 위해서는 Interior Gateway가 아닌 Exterior Gateway(BGP) 프로토콜을 이용해 Peering을 해야 한다.
- 선 끝에 위치하고 있는 라우터 2개만 BGP Peering을 통해 서로의 라우팅 테이블을 교환하고 내부에 위치하고 있는 반투명 라우터들은 서로를 모르게 구성함으로써 데이터를 가볍게  교환할 수 있다. 이것을 Autonomous System이라고 부르며 각각을 식별하기 위해 ASN(Autonomous System Number)가 있다. 이것을 지정해 서로 통신을 하는 거다. 참고로 AWS의 디폴트 ASN은 64512다.




- https://bosungtea9416.tistory.com/entry/AWS-Site-to-Site-VPN-%EA%B5%AC%EC%84%B1%ED%95%98%EA%B8%B0

----
aws azure
vpc : virtual network

azure
1. virtual network gateway를 만들기 전 gateway subnet 구성 필요
- gatewaysubnet이라는 특정 서브넷이 필요하다고한다. -> 이거 어떻게 구성하면 좋을지 물어봐야한다.

cgw : local network gateway
vgw : virtual network gateway


- https://learn.microsoft.com/ko-kr/azure/vpn-gateway/tutorial-site-to-site-portal
- https://azurekor.com/m/221

