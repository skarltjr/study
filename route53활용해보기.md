```
간단한 클라우드 환경을 구축해보자

- route53 : 도메인을 통해 내 서비스에 접근, 네임서버를 통해 도메인과 ip 연결
- vpc : 가상의 네트워크 구축
- igw : 외부 인터넷으로부터 진,출입
- lb : (당장 필요하지 않지만) 오토 스케일링 등으로 유동적으로 변하는 인스턴스에 고정적인 접근을 제공 및 부하 분산
- nacl : 서브넷 방화벽
- sc : 인스턴스 방화벽
```
- ![image](https://user-images.githubusercontent.com/62214428/236638041-5155d04c-46a3-4e94-85ea-3e1a0b02b6f8.png)
```
참고로 서브넷 1개만 할 예정.. 돈...나감
```


### 1. 도메인 구입
- 현재 도메인을 구입했으며 route53을 통해 네임서버 자동 등록

### 2. vpc 생성하기
- <img width="477" alt="스크린샷 2023-05-07 오전 1 21 47" src="https://user-images.githubusercontent.com/62214428/236635608-8de89600-6a93-417a-b120-e7844aad0d03.png">
- <img width="477" alt="스크린샷 2023-05-07 오전 1 21 52" src="https://user-images.githubusercontent.com/62214428/236635611-f7322aee-2047-4076-a2ec-9eee0b50c755.png">


```
사진에는 서브넷 1개지만 2개로 가야한다.
현재 vpc와 함께 생성되는것들
- 1개의 인터넷 게이트 웨이
- 2개의 퍼블릭 서브넷
- 2개의 네트워크 acl(nacl)
  - 서브넷과 연결된 nacl
- 2개의 라우팅 테이블
  - 해당 라우팅 테이블은 위 퍼블릭 서브넷과 연결되어있다.
  - igw - routing table - public subnet
  - 해당 라우팅 테이블은 igw와 연결되어 있다. 따라서 인터넷 게이트웨이를 통해 들어온 트래픽을 퍼블릭 서브넷으로 보내준다.
- 1개의 보안그룹
```
```
참고!!!! : 서브넷 2개 이상해야 lb 사용이 가능하다. 나는 나중에 추가로 구축했다 ;; 아래처럼
```
- <img width="1545" alt="스크린샷 2023-05-07 오전 2 44 50" src="https://user-images.githubusercontent.com/62214428/236639161-a155d666-758b-4a15-b396-3dc00bdd01d1.png">

### 3. ec2 생성하기
```
위 vpc에서 서브넷 2개를 만들어야한다. 
```
- <img width="777" alt="스크린샷 2023-05-07 오전 1 57 57" src="https://user-images.githubusercontent.com/62214428/236637292-19b35dd4-6d1c-4aa9-9ba2-f9b17264cbf0.png">
```
vpc를 생성하며 만든 vpc, 서브넷, 보안그룹을 적용한 ec2를 생성한다
- 참고로 보안그룹 인바운드 규칙에 source가 적용되어있을 수 있다. 
- 기존 규칙 제거 후 새로운 규칙 추가하여 모든 트래픽 유형에 0.0.0.0/0으로 설정하자
```

### 4. lb 생성하기
- <img width="847" alt="스크린샷 2023-05-07 오전 2 33 02" src="https://user-images.githubusercontent.com/62214428/236638747-77d51477-d95c-4331-b29b-e45fadcd1219.png">
```
internet-facing : private, public ip 
internal : private ip를 통해 내부에서만 로드밸런싱
```
- <img width="943" alt="스크린샷 2023-05-07 오전 2 33 11" src="https://user-images.githubusercontent.com/62214428/236638771-118e2229-7888-4aa2-95b0-f093f3614a34.png">
```
만들어둔 vpc와 서브넷 선택
```
```
보안그룹 또한 생성되었던 보안그룹 선택
```
- <img width="1081" alt="스크린샷 2023-05-07 오전 2 37 17" src="https://user-images.githubusercontent.com/62214428/236638893-91844c1e-d38a-4021-8c35-15c229eba868.png">
```
타겟 그룹 생성 후 지정
```
- <img width="1216" alt="스크린샷 2023-05-07 오전 3 20 47" src="https://user-images.githubusercontent.com/62214428/236640621-7af7d9c0-264d-4da4-860c-d20e2b67bfbc.png">

### 확인해보기
- lb의 dns로 접근해보면
- <img width="748" alt="스크린샷 2023-05-07 오전 3 20 07" src="https://user-images.githubusercontent.com/62214428/236640585-1bbc383b-6a2c-48de-803e-6e589c258627.png">
- <img width="656" alt="스크린샷 2023-05-07 오전 3 20 10" src="https://user-images.githubusercontent.com/62214428/236640591-b8f8ccc1-59d9-4aeb-9593-a38a354ca216.png">

### route53과 alb 연동
- 호스팅 영역 추가
- <img width="1110" alt="스크린샷 2023-05-07 오전 3 26 00" src="https://user-images.githubusercontent.com/62214428/236640846-dca07cf9-f7a9-440a-883d-693ef12f7149.png">
- <img width="531" alt="스크린샷 2023-05-07 오전 3 26 31" src="https://user-images.githubusercontent.com/62214428/236640890-c0392095-ffbb-451f-947d-316e367d9bd7.png">
- <img width="292" alt="스크린샷 2023-05-07 오전 3 26 34" src="https://user-images.githubusercontent.com/62214428/236640891-4f804336-1d43-4215-950b-c7b96e0b9201.png">

---- 

- https://velog.io/@madmingi/AWSELB-%EC%8B%A4%EC%8A%B5
