쿠버네티스에서의 인증

1. 쿠버네티스의 접근제어 체계
- authentication(인증) -> authorization(인가) -> admission control
```
접근자의 신원 정보를 확인하고
어떤 권한을 갖고 있으며, 어떤 행동을 할 수 있는지 확인한다

admission control은 opa등 admission controller가 필요에 따라 etcd에 정보를 저장하기 전
추가적인 validation등을 진행할때 사용된다
```

2. 쿠버네티스와 user database의 부재
- 쿠버네티스는 유저 데이터를 별도 db에 저장하지 않는다. 대부분의 웹서비스와 다른점이다.
- 쿠버네티스는 내부 인증체계에 종속되는 부분이 거의 없이 외부 시스템(X.509, HTTP AUTH, proxy authentication 등)에 의존하고 이를 통해 확장성을 갖는다.
- 그러니까 이러한 유저, 인증 정보를 별도로 저장 및 구현한게아니라 외부 시스템이나 매커니즘을 활용한다.

3. 쿠버네티스의 group
- group자체가 존재하는 리소스는 아니다.
- 쿠버네티스는 roleBinding | clusterRoleBinding 같이 role을 바인딩시켜 특정 권한을 갖는 그룹의 개념
```
system:authenticated: 사용자 인증을 통과한 그룹
system:anonymous: 사용자 인증을 하지 않은 익명 그룹
system:masters: 쿠버네티스의 full access 권한을 가진 그룹
```

### X.509 Certificate
먼저 x.509에 대해 살펴보면 
```
public key infrastructure / PKI

pki는 비대칭 암호화 기술을 이용한 공개키 기반 인증 체계
x.509는 이러한 pki 기술 중 표준 포맷
쿠버네티스는 x.509 certificate를 이용하여 사용자의 신원을 인증하는 목적으로 사용
```

1. public private keys
- public key는 누구나 가질 수 있다.
- private key는 오직 public/private key pair 소유자만 갖고 있다.
- public key를 활용하여 암호환 메세지는 매핑되는 private key로 복호화 가능
- private key로 암호화된 메세지는 public key로 복호화 불가 / 그러나 private key 소유자가 작성한 메세인지는 검증 가능
  - 그리고 이것이 디지털 서명
  - x.509에서는 이러한 디지털 서명을 사용하여 사용자의 신원을 확인한다.
```
참고로 TLS와 디지털 서명은 다른데
쿠버네티스는 pki(x.509)를 TLS / 신원 인증(디지털 서명) 모두를 위해 이 방법을 활용한다.

TLS의 경우 SSL3.0 계승
- 서버가 private key를 들고있고
- public key를 CA에 전달하여 인증서를 발급받고
- 클라이언트가 CA로부터 public key를 전달받고
- 클라이언트가 서버 접속시 서버가 발급받은 인증서를 전달하고
- public key로 복호화하여 일부 서버 정보와 인증서 정보를 확인한 뒤 안전함을 확인하면
- 서버에게 대칭키를 전달하고 이후부터 대칭키로 메세지를 암호화하여 주고받는다.

디지털 서명의 경우:
- 문서 작성자가 private key를 들고 있음
- 문서의 해시값을 private key로 암호화하여 서명 생성
- 수신자는 public key로 서명을 복호화하여 해시값 획득
- 원본 문서의 해시값과 비교하여 검증 / 복호화 할 수 있다는거 자체가 private key 소유자가 서명했다는뜻
- 이를 통해 문서 작성자 확인 및 위변조 여부 검증 가능
```

2. 쿠버네티스에서 신원 인증
```
1. 인증서 발급 과정
- 클라이언트가 private key 생성
- private key로 CSR 생성 (이때 신원정보 포함)
- CSR을 ROOT CA에 전달
- ROOT CA가 자신의 private key로 CSR에 서명하여 인증서 발급

2. API 서버 인증 과정
- 클라이언트가 요청에 private key로 디지털 서명
- 인증서와 함께 API 서버로 전송
- API 서버는 ROOT CA의 public key로:
  1) 인증서의 서명 검증 (위조 여부)
  2) 인증서 유효기간 등 검증
- 검증 성공 시 인증서 내 신원정보(CN/O)로 권한 확인
```
