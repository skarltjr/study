```
간단한 클라우드 환경을 구축해보자

- route53 : 도메인을 통해 내 서비스에 접근, 네임서버를 통해 도메인과 ip 연결
- vpc : 가상의 네트워크 구축
- igw : 외부 인터넷으로부터 진,출입
- lb : (당장 필요하지 않지만) 오토 스케일링 등으로 유동적으로 변하는 인스턴스에 고정적인 접근을 제공 및 부하 분산
- nacl : 서브넷 방화벽
- sc : 인스턴스 방화벽
```
- ![image](https://user-images.githubusercontent.com/62214428/236634179-be844fef-44f4-4cb7-a3f9-1d5e8b2fa66e.png)

### 1. 도메인 구입
- 현재 도메인을 구입했으며 route53을 통해 네임서버 자동 등록

### 2. vpc 생성하기
- <img width="477" alt="스크린샷 2023-05-07 오전 1 21 47" src="https://user-images.githubusercontent.com/62214428/236635608-8de89600-6a93-417a-b120-e7844aad0d03.png">
- <img width="477" alt="스크린샷 2023-05-07 오전 1 21 52" src="https://user-images.githubusercontent.com/62214428/236635611-f7322aee-2047-4076-a2ec-9eee0b50c755.png">
```
현재 vpc와 함께 생성되는것들
- 1개의 인터넷 게이트 웨이
- 1개의 퍼블릭 서브넷
- 1개의 네트워크 acl(nacl)
  - 서브넷과 연결된 nacl
- 1개의 라우팅 테이블
  - 해당 라우팅 테이블은 위 퍼블릭 서브넷과 연결되어있다.
  - igw - routing table - public subnet
  - 해당 라우팅 테이블은 igw와 연결되어 있다. 따라서 인터넷 게이트웨이를 통해 들어온 트래픽을 퍼블릭 서브넷으로 보내준다.
- 1개의 보안그룹
```
### 3. ec2 생성하기
- <img width="777" alt="스크린샷 2023-05-07 오전 1 57 57" src="https://user-images.githubusercontent.com/62214428/236637292-19b35dd4-6d1c-4aa9-9ba2-f9b17264cbf0.png">
```
vpc를 생성하며 만든 vpc, 서브넷, 보안그룹을 적용한 ec2를 생성한다
- 참고로 보안그룹 인바운드 규칙에 source가 적용되어있을 수 있다. 
- 기존 규칙 제거 후 새로운 규칙 추가하여 모든 트래픽 유형에 0.0.0.0/0으로 설정하자
```
