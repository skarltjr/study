1. service
- 클러스터안에서 실행되는 애플리케이션을 외부로 노출시키는 단일 엔드포인트
- 각 service는 개별적인 clusterIp를 보유하며 이를 통해 클러스터내 클라이언트간 통신
- 왜 필요한가? 
  - 쿠버네티스는 desired state를 유지하며, 이로인해 pod는 재생성, 삭제된다.
  - 따라서 언제든지 대체될 수 있는 동적인 pod에 대해 고정적인 접근 지점이 필요하다.
- 기본적으로 Service의 대상 pod는 selector를 통해 결정된다.
- l4, ip/port 계층 로드밸런싱

2. port
- port:
  - 정의: Service가 클러스터 내부에서 노출하는 포트
  - 역할: 클러스터 내의 다른 클라이언트(다른 Pod 또는 외부 애플리케이션)가 Service에 접근할 때 사용하는 포트
- targetPort:
  - 정의: Service가 트래픽을 전달할 대상 Pod의 포트
  - 역할: 실제 애플리케이션이 실행 중인 Pod 내의 특정 컨테이너 포트를 지정
- 참고로 targetPort는 숫자나 컨테이너 포트의 이름으로 지정할 수 있다. 지정하지 않으면 기본적으로 port 값과 동일하게 설정

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-and-logging-deployment
  labels:
    app: web-and-logging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-and-logging
  template:
    metadata:
      labels:
        app: web-and-logging
    spec:
      containers:
      - name: web-server
        image: nginx:1.21.6
        ports:
        - name: http
          containerPort: 80
      - name: logging-agent
        image: fluent/fluentd:v1.14.2
        ports:
        - name: metrics
          containerPort: 9090

```

```
apiVersion: v1
kind: Service
metadata:
  name: web-and-logging-service
spec:
  selector:
    app: web-and-logging
  ports:
    - name: web
      protocol: TCP
      port: 80          # 클러스터 내에서 노출되는 포트
      targetPort: http  # web-server 컨테이너의 포트 이름
    - name: metrics
      protocol: TCP
      port: 9090        # 클러스터 내에서 노출되는 포트
      targetPort: metrics # logging-agent 컨테이너의 포트 이름
```
- 서비스의 port는 다른 서비스 port와 중복될 수 있을까?
  - 각 서비스는 개별 clusterIp를 갖고 있기 때문에 port의 중복은 상관없다.

3. Service Types
- ClusterIp : 
  - `Exposes the Service on a cluster-internal IP`
  - service를 클러스터내부에만 노출
  - default type
  - ![image](https://github.com/user-attachments/assets/e9a657fa-e2eb-4084-a228-a163b4f5f3ba)

- NodePort :
  - 외부에서 node ip: node port로 들어오는 요청을 해당 포트와 연결된 pod로 트래픽 전달
  - 이때 pod가 서로 다른 node에 거쳐 존재하는 경우, 각 노드에 service가 자동으로 생성
    - 즉 NodePort Service는 모든 쿠버네티스 노드의 동일 포트를 개방한다.
    - 그러나 node ip에 해당하는 node에 떠 있는 pod로만 접근되는것은 아니다.
      - kube-proxy는 iptables를 통해 서비스와 연결된 pod의 ip 목록을 알고 있고, 여러개라면 라운드로빈으로 전달
  - ![image](https://github.com/user-attachments/assets/96656cc8-6ce4-41e3-9704-eb83697b1887)

- LoadBalancer
  - 별도 외부 로드밸런서를 k8s 클러스터의 서비스로 provisioning
  - 해당 타입 service는 NodePort 타입 Service를 한 번 더 감싼 Service다.
  - nodePort의 단점은 3개의 노드가 존재했을때 어떤 노드를 통해 nodeport로 접근하여 파드에 접근할 수 있지만,
  - 노드가 죽었을때 자동으로 제외되지 않는다는것. 즉 로드밸런서 타입은 nodePort타입에다가 노드들의 상태를 파악하여 살아있는 노드로만 nodePort를 통해 접근할 수 있도록 로드밸런서가 다뤄주는것
  - 그럼 각 nodePort타입 서비스가 lb의 타겟그룹이되는 것
  - ![image](https://github.com/user-attachments/assets/663ec14e-a50a-4e97-a990-449e50b98fc7)

4. 서비스 디스커버리
- source가 target에 접근하기 위해선 ip:port가 필요하고 이를 찾아내는 기능이 서비스 디스커버리
- 크게 DNS 기반, 환경변수 기반 서비스 디스커버리가 존재

5. DNS service discovery
- coreDNS와 같은 애드온과 함께 동작(기본은 kube-dns)
- service의 생성을 인지하고 레코드를 생성한다.
  - 예를 들면, 쿠버네티스 네임스페이스 my-ns에 my-service라는 서비스가 있는 경우, 컨트롤 플레인과 DNS 서비스가 함께 작동하여 my-service.my-ns에 대한 DNS 레코드를 만든다
- 참고로 srv(service) record를 지원하여, ip와 함께 port를 찾아낼 수 있다.



