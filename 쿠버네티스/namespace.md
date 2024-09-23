1. namespace란
- 쿠버네티스에서 네임스페이스란 단일 클러스터내 리소스를 격리하는 매커니즘
- 동일 네임스페이스내에서 동일 리소스는의 리소스명은 중복될 수 없다.
- 중요한것은 네임스페이스는 네임스페이스 지정이 가능한 object에 대해서만 적용이 가능하다.
  - deployment, services. 등
  - storageClass, Nodes, PV등 cluster-wide object는 대상이아니다.
```
linux의 namespace 기술이 쿠버네티스까지 도달했다.
리눅스는 모든걸 파일로 관리하기에 네임스페이스도 파일로 관리
쿠버네티스는 etcd에 
```

2. multiple namespaces
- 쿠버네티스에서 네임스페이스의 기본 의도는 규모가 커진 팀, 프로젝트에서 다양한 환경을 위한 scope을 제공하기위함.
- 그 예시로 네임스페이스에 리소스 쿼터를 지정하여 네임스페이스별 격리가 가능하다.
- 기본 네임스페이스는 다음과 같다.
  - default
  - kube-node-lease : 각 노드 lease object와 관련된 네임스페이스
  - kube-public: 인증되지 않은 클라이언트를 포함하여 내-외부 모든 클라이언트가 읽을 수 있는 네임스페이스
  - kube-system : 쿠버네티스 시스템에 의해 생성된 object들을 위한 네임스페이스


3. 네임스페이스와 DNS
- Service를 생성할때 DNS entry도 생성을 한다.
- 기본적으로 `<service-name>.<namespace-name>.svc.cluster.local`이다.
- 그런데 만약 컨테이너에서 `service-name`으로만으로 참조하는 경우가 존재할 수 있는데, 이는 자신 네임스페이스에서만 찾게된다.
```
문제가 될 수 있는 상황은

- dev, stg, prd 환경별 네임스페이스를 나눴다.
- 각 환경별 서비스의 FQDN은 다음과 같다
<service-name>.dev.svc.cluster.local
<service-name>.stg.svc.cluster.local
<service-name>.prd.svc.cluster.local

- dev에서 stg 호출간 service-name만 사용하게되면 dev는 dev namespace에서의 서비스를 참조하게된다는것
```
- TLD를 네임스페이스 사용하지 말자.
- public DNS 질의보다 <service-name>.prd.svc.cluster.local로 검색이 먼저다.
```
- 그럼 서비스명은 hello / namespace가 com이라고 해보자
- 서비스의 FQDN은 hello.com.svc.cluster.local인데
- 애플리케이션에서 도메인 기반 호출을 진행하니 
- a -> hello.com(DNS query) -> b 애플리케이션 호출을 의도했는데, 
- com 네임스페이스가 존재해서 com 네임스페이스의 service-name 서비스를 먼저 찾는다.
```
