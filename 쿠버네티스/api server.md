![image](https://github.com/user-attachments/assets/f8f070a6-4244-4e71-83c4-912fdef3bec8)

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
