1. replicaSet
- 레플리카셋의 목적은 레플리카 파드 집합의 실행을 항상 안정적으로 유지하는것
- 명시된 파드 개수에 대한 가용성을 보장
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```
2. replicaSet의 동작 방식
- `selector`에 일치하는 파드를 식별
- desired state에 정의된 상태 파악
  - `spec.replicas` 필드를 통해 유지되어야하는 파드의 수 파악
- control plane의 controller manager 중 replicaSet Controller는 주기적으로 상태를 파악하고 정의된 상태를 유지하기 위해 동작
  - 각 노드의 kubelet이 노드,파드의 상태를 주기적으로 api server에게 전달 및 etcd에 저장
  - controller manager는 api server watch를 통해 주기적으로 이러한 파드 상태 정보를 알려주세요~ 요청 전달

3. 주의
- 앞서 `selector`에 일치하는 파드를 식별한다고 했다.
- 이때 단일 pod의 label이 replicaSet의 selector에서 명시한 label과 같다면, 이 또한 replicaSet에 포함된 파드로 인식해버린다.

