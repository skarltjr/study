![image](https://github.com/user-attachments/assets/f8f070a6-4244-4e71-83c4-912fdef3bec8)
![image](https://github.com/user-attachments/assets/e1a81596-fe47-44d7-a05e-1e2a4c360728)

```
먼저 control plane의 api server란
- 중앙 관리 entity이자 etcd와 통신하는 유일한 컴포넌트
```

1. Key Responsibilities of kube-apiserver
- `API exposure`
  - 내외부에 노출되는 http 인터페이스로 쿠버네티스 api를 노출한다.
  - REST requests를 validation하고 클러스터내 etcd storage에 저장된 state object를 업데이트한다.
  - 업데이트된 내용이 모든 컴포넌트에 동기화될 수 있도록 보장한다.
- `Authentication`
  - 인증
  - 쿠버네티스 클러스터내 요청으로부터 해당 요청을 수행할 수 있는 권한이 존재하는지 확인합니다.
- `Authorization`
  - 인가
  - 요청이 인증되었다면, apiserver는 요청자의 요청 수행을 허용할지 판단합니다.
- `Admission Control`
  - 중요 보안 및 거버넌스 툴
  - 요청 수행을 허락하기 전, apiserver는 여러 admission control module을 거쳐 특정 조건에서 요청을 수락하거나 거절할지 결정
- `Regulatory Compliance`
  - apiserver는 모든 api 요청과 응답을 로깅하여 모든 클러스터와의 상호작용 및 변화를 감시


2. How kube-apiserver Works
- `Receiving API Requests` : kubectl 혹은 다른 클라이언트들이 kube-apiserver에게 http/https 요청을 전달한다
- `Authentication and Authorization` : 각 요청은 인증되고 권한이 부여된다.
  - 인증(authentication)은 static token, certificates 혹은 openID와 같은 third-party authention을 통해 처리된다
  - 인가(authorization)는 쿠버네티스내에 정의된 role과 policies로 관리된다
- `Admission Control` : 요청이 수행되기전, 다양한 admission controller들을 거친다.
  - 이 컨트롤러들은 security policies를 검토, 리소스 제한등을 수행한다
- `Watch for Updates` : apiserver는 특정 객체의 변경사항을 감시하고 updates stream을 반환한다
  - 이것은 desired state를 유지하는데 매우 중요하다.
  - control-loop는 api server를 계속 watch하여 리소스의 update stream을 전달받으며 이상있을 시 선언된 상태로 유지하도록 행동한다.
- `Data Storage` : 요청을 통해 클러스터의 상태가 변화되면 apiserver는 etcd에 내용을 수정 및 저장한다.

3. api server code layer에서 요청처리 흐름
- ![image](https://github.com/user-attachments/assets/2e11b2ff-a783-4a74-9fa1-37aa8a915374)
- DefaultBuildHandlerChain() :
  - DefaultBuildHandlerChaind에 등록된 필터 체인을 거쳐 ctx.RequestInfo에 사용자 인증 정보, 응답 코드등을 담아 다음 체인으로 넘긴다.
  - multiplexer : http path에 따라 해당 요청을 처리할 핸들러로 라우팅
  - 각 api group에 맞는 핸들러가 요청 처리

3-1. deep dive for DefaultBuildHandlerChain
- WithRequestInfo : http request에 담긴 정보를 context에 requestInfo 객체로 담아낸다.
- WithTimeoutForNonLongRunningRequests : watch처럼 long-running이 아닌 non-long-request에 대해 요청이 장기간 실행되지 않은 경우 timeout 수행
- WithPanicRecovery : panic으로 시스템에 영향을 주지 않도록 recovery
- WithCORS : Cross-Origin Resource Sharing 검사
- WithAuthentication : 사용자를 인증하고, 인증후에는 context에 유저 정보를 저장한다.
  - 이때 인증이 끝난 요청에 대해서는 요청 헤더에서 Authorization 헤더를 제거한다
  - 이는 인증이 끝났기에 재인증 제거, 불필요한 정보를 다음으로 넘길 필요도 없으며, 보안상 이런 인증정보를 유지할 필요가없다.
- WithAuthorization : 모두 인증된 요청이 넘어오며, 인가 단계를 진행
  - 권한 여부를 판단하여 멀티플렉서에게 전달하여 요청 라우팅

4. api server는 etcd와 어떻게 통신하는가
- 쿠버네티스 api server는 etcd 클라이언트를 직접 호출하는것이 아니라 이를 한단계 래핑하여 추상화된 레이어를 활용한다
  - 그리고 이 추상화된 레이어 안에서 etcd 클라이언트가 사용된다
  - 즉 etcd 클라이언트 동작방식을 이해해봐야한다.
```
- etcd 클라이언트는 etcd 클러스터 노드들에 대한 TCP 커넥션을 모두 들고 있다.
- write/read 상관없이 모든 요청은 라운드 로빈 형태로 각 etcd 멤버들에게 전달된다.
- 만약 읽기 요청이라면 요청을 전달받은 etcd 노드가 그대로 응답을 반환한다
- 만약 쓰기 요청이면서 전달받은 etcd 노드가 leader라면 리더가 이를 처리한다.
- 만약 쓰기 요청이면서 전달받은 etcd 노드가 follower라면 리더에게 전달한다.
```

-------
https://www.redhat.com/es/blog/kubernetes-deep-dive-api-server-part-1

