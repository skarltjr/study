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
- 헷갈릴 수 있는데 비대칭키 암호화는 결국 공개키 암호화, 개인키 복호화 or 개인키 암호화, 공개키로 복호화
- 그러나 이 비대칭 암호화는 cpu등 리소스 사용량이 많아 tls의 경우 이후부턴 대칭키를 통한 암복호화를 진행
```
TLS와 디지털 서명은 다른데
쿠버네티스는 pki(x.509)를 TLS / 신원 인증(디지털 서명) 모두를 위해 이 방법을 활용한다.

TLS의 경우 SSL3.0 계승
- 서버가 private key를 들고있고
- public key를 CA에 전달하여 인증서를 발급받고
- 클라이언트가 CA로부터 public key를 전달받고
- 클라이언트가 서버 접속시 서버가 발급받은 인증서를 전달하고
- client는 인증서를 확인하여 서버가 믿을 수 있는 대상임을 인지
- 이제 클라이언트는 앞으로 이 key(대칭키)로 암호화하여 서로 소통하자고 알리기 위한 pre-master secret를 공개키로 암호화하여 전달
- 클라이언트와 서버는 ChangeCipherSpec을 서로 전달하여 이제부터 각자 동일한 알고리즘으로 pre-master secret+ClientHello.random + ServerHello.random를 암호화한 세션키를 각자 준비하고
- 이제부터 이들은 이 동일한 대칭키(세션키)로 암호화 통신

디지털 서명
서명 생성 과정:
- 발신자는 메시지의 해시값을 생성
- 이 해시값을 발신자의 private key로 암호화
- 암호화된 해시값이 디지털 서명이 됨
- 원본 메시지와 디지털 서명을 함께 전송


서명 검증 과정:
- 수신자는 받은 메시지의 해시값을 계산
- 디지털 서명을 발신자의 공개키로 복호화
- 두 해시값을 비교하여 일치하면 서명이 유효함을 확인
```

2. 쿠버네티스에서 신원 인증
```
1. 인증서 발급 과정
- 클라이언트가 private key 생성
- private key로 CSR(Certificate Signing Request) 생성 (이때 신원정보 포함)
- CSR을 ROOT CA에 전달
- ROOT CA가 자신의 private key로 CSR에 서명하여 인증서 발급

2. API 서버 인증 과정
- tls를 위한 핸드쉐이크 시작
- 클라이언트가 요청에 private key로 디지털 서명
- 인증서와 함께 API 서버로 전송
- API 서버는 ROOT CA의 public key로:
  1) 인증서의 서명 검증 (위조 여부)
  2) 인증서 유효기간 등 검증
- 검증 성공 시 인증서 내 신원정보(CN/O)로 권한 확인
- 이후 대칭키로 암호화된 통신

kubeconfig도 결국 위와 동일한데
client(local)에서 개인키를 생성하고 쿠버네티스에 csr 전달
crt(cert) 인증서 발급
kubeconfig에 해당 인증서 설정함으로써 쿠버네티스 접근시 해당 인증서를 사용하고
이를 통해 인증을 거친다.
```

```
참고로 그럼 우리는 어떻게 kubeconfig를 활용하여 kubectl로 eks에 접근하는가?

1. aws credential 세팅 => ~/.aws/credential
2. 적절한 권한이 존재하니 eks로부터 kubeconfig 설정 정보를 받을 수 있다
- aws eks --region <region-code> update-kubeconfig --name <cluster_name>
3. 그럼 먼저, 내가 kubeconfig + kubectl(client)로써 접근하고자하는 대상 서버(eks api server)가 믿을 수 있는 대상인지 어떻게 판단하는가

> cat ~/.kube/config를 확인해보면 certificate-authority-data를 확인할 수 있다.
$ cat ~/.kube/config           
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJTm5E~~~~xxxxx~~~~lDQVRFLS0tLS0K
    server: https://7398A2FE7F18173AC9B75F09C9288F0B.yl4.ap-northeast-2.eks.amazonaws.com
  name: ~
contexts:
- context:
    cluster: ~
    user: ~

> certificate-authority-data를 실제로 echo "certificate-authority-data" | base64 -d로 디코딩해보면 인증서 정보가 나온다
-----BEGIN CERTIFICATE-----
MIIDBTCCA~~~xxxx~~~alxb+7fvwwTE
-----END CERTIFICATE-----


> 즉 kubeconfig 설정시 인증서 정보를 인코딩한 데이터가 certificate-authority-data에 설정되는데
이 certificate-authority-data 정보는 x.509 형식의 우리 eks Root CA 인증서로 클러스터의 다른 모든 인증서들을 검증하는 기준
이제 클라이언트(kubectl) <-> API(eks api server) 서버 통신 과정에서
- kubectl이 API 서버에 연결 시도
- API 서버가 자신의 인증서 제시
- kubectl이 CA 인증서로 API 서버 인증서 검증
- 검증 성공시 안전한 통신 시작

4. 그렇다면 kubeconfig를 통해 쿠버네티스는 어떻게 인증을 거치는가
- aws eks는 좀 다른 방식(iam)
- eks는 kube config에 있는 commnad를 통해 aws의 토큰을 생성하여 쿠버네티스에 bearer <token>으로 http 인증 방식으로.
```

3. 인가
```
# 예시: Pod 생성 요청이 들어왔을 때
요청 컨텍스트:
- 사용자: kidol
- 그룹: dev-team
- 네임스페이스: development
- 리소스: pods
- 동작: create

인가 체크 순서:
1) RoleBinding/ClusterRoleBinding 검색
   - 사용자 'kidol' 또는 그룹 'dev-team'에 연결된 바인딩 확인

2) Role/ClusterRole 권한 확인
   - 바인딩된 Role의 rules 섹션 검사
   - apiGroups, resources, verbs 매칭 확인

3) 결정
   - 모든 조건 만족 → 허용
   - 하나라도 불만족 → 거부 (403 Forbidden)
```

### HTTP Authentication
- http header를 통해 인증정보 전달(auth token)
- Authorization: <type> <credentials>
  - type: basic | bearer (쿠버네티스는 두 타입을 사용)
  - credential : id & password | secret(jwt)
- 보통 사람이나 클라이언트말고 pod가 api server 요청을 보낼떄 활용되는데
- serviceaccount 활용시 다음처럼 pod에 마운트하여 pod가 http authentication을 진행할 수 있도록한다.
- jwt는 X.509 Certificate와 마찬가지로 private key를 이용하여 토큰을 서명하고 public key를 이용하여 서명된 메세지를 검증한다.
- ![image](https://github.com/user-attachments/assets/23d54875-19f8-4060-8f0f-01f52f35d9d5)
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa


apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: my-sa-token
  annotations:
    kubernetes.io/service-account.name: my-sa

spec:
  serviceAccountName: my-sa
  # /var/run/secrets/kubernetes.io/serviceaccount/token에 마운트
```

### OIDC
- OpenID Connect(OIDC)는 권한허가 프로토콜인 OAuth 2.0 기술을 이용하여 만들어진 인증 레이어
  - OpenID는 인증 시스템으로써 사용자 정보를 관리하고 인증하는 것에 초점이 맞춰져 있다.
  - 그래서 보통 응답은 사용자 인증 및 유저 정보 제공(id token)
- OAuth는 권한허가를 처리하기 위해 만들어진 표준 프로토콜
  - OAuth는 사용자 인증 보다는 제한된 사람에게 (혹은 시스템) 제한된 권한을 어떻게 잘 부여할 것인가에 대해서 중점적으로 다룬다
  - 그래서 보통 응답은 권한
- ![image](https://github.com/user-attachments/assets/519f10aa-2bd1-4db5-89ac-cf0d30ae0a8c)

### Identity Provider / IdP
- 실제 사용자 정보를 제공해주는 신원 제공자
- Oidc는 인증 표준 규격을 정의하는 프로토콜 / idp는 이러한 실제 인증 및 사용자 정보 제공 수행
- ![image](https://github.com/user-attachments/assets/d8dfd07c-4dea-4746-8658-638cb5b11d0c)
```
oauth에서 ouath는 그 자체로 authentication(인증) 기술이 아니라고 명시한다.
oauth에서 제공하는 access token은 특정 액션을 위한 일시적으로 권한을 허가해준 토큰일뿐 사용자 정보가 담긴게 아니다.
물론 이를 위해 사용자 인증을 거치긴하지만 access token 자체가 사용자의 정보를 대표하지 않는다.

즉 oidc는 OAuth 2.0 기술을 이용하여 만들어진 인증 레이어이지만 oauth 자체는 인증이 목적이 아니다.
그럼 oidc는 이런 문제를 어떻게 해결하는가
```
### open id scope & id token
- openId connect는 사용자 인증과 신원 정보 제공을 하고 싶다
- oauth는 권한을 허가해준다
- 그렇다면 oidc가 oauth에게 사용자 신원 정보를 제공해달라는 권한 요청을 한다면?
  - 권한 요청이지만 신원 정보도 제공해달라
```
1. OIDC는 OAuth 2.0의 scope에 'openid'를 추가하여 인증 요청임을 표시
2. 인증 후 OAuth가 발급하는 access token과 함께 ID Token을 추가로 발급
   - ID Token은 JWT 형식의 토큰
   - 사용자 식별자(sub), 이름, 이메일 등 신원 정보 포함
   - IdP(Identity Provider)가 서명하여 신뢰성 보장
3. 필요시 UserInfo Endpoint를 통해 추가 사용자 정보를 요청할 수 있다
   - Access Token으로 인증
   - 표준화된 사용자 정보 claim 사용
```
- ![image](https://github.com/user-attachments/assets/bbc00876-9b8f-4cc5-bb1f-89ef86cad37d)
