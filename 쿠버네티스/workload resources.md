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

3. replicaSet 주의
- 앞서 `selector`에 일치하는 파드를 식별한다고 했다.
- 이때 단일 pod의 label이 replicaSet의 selector에서 명시한 label과 같다면, 이 또한 replicaSet에 포함된 파드로 인식해버린다.
- 참고로 replicaSet은 목적부터 파드의 복제본 수를 유지하는것
  - deployment는 replicaSet의 상위 개념으로 replicaSet을 바탕으로 롤링 업데이트 및 롤백 등 배포를 다루는 개념

4. statefulSet
- stateful한 애플리케이션 관리를 위해 사용되는 workload object
- 각 파드는 독자성을 유지한다.
  - 이 파드들은 동일한 스펙으로 생성되지만 서료 교체는 불가능.
  - 즉 재스케줄링이 되더라도 지속적으로 유지되는 식별자를 갖는다.
- 참고로 statefulSet은 statefulSet 삭제시 연관 파드 종료에 대한 보증을하지 않는다
  - 즉 제거전 replicaSet을 0으로 축소해야할 필요가 있다.
- statefulSet은 network identity를 책임질 headless service가 필요하다.
  - 로드밸런싱과 single service ip가 필요하지 않을 수 있다.
  - 이때 Service의 .spec.clusterIp를 None으로 설정하여 headless service를 활용할 수 있다.
  - headless service에는 clusterIp가 할당되지 않으므로 kube-proxy는 이 service를 다루지 않는다.
    - 안정된, 고유한 네트워크 식별자 = 고유한 DNS 이름을 개별적으로 제공하여 접근할 수 있도록하기 위함
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # .spec.template.metadata.labels 와 일치해야 한다
  serviceName: "nginx"
  replicas: 3 # 기본값은 1
  minReadySeconds: 10 # 기본값은 0
  template:
    metadata:
      labels:
        app: nginx # .spec.selector.matchLabels 와 일치해야 한다
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi

```

5. statefulSet을 그럼 언제 사용하나요?
```
안정된, 고유한 네트워크 식별자
- statefulSet으로 생성된 파드는 고유한 dns host name을 갖는다.
- 파드가 재스케줄링되거나 재배치되어도 이 고유한 네트워크 식별자를 유지한다
  - web-0
  - web-1
  - web-2 ..

안정된, 지속성을 갖는 스토리지.
- statefulSet은 각 파드에 대한 고유한 persistent volume을 제공하여, 재시작되더라도 데이터 유지를 보장
- 즉 파드의 라이프사이클과 독립적으로 스토리지 유지

순차적인, 정상 배포(graceful deployment)와 스케일링.
순차적인, 자동 롤링 업데이트.
- 파드 n번부터 순차적으로 생성
- 업데이트도 n번부터 순차적으로
- scale-in시 삭제는 역순으로 순차적으로


```

6. daemonSet
- 모든 혹은 일부 노드 전체에 특정 파드를 실행하기 위함
- daemonSet 삭제 = 생성된 데몬셋 파드도 같이 삭제
- 우리같은 경우 각 노드마다 n개의 dd agent가 필요하니 사용할 수 있겠다.
- daemonSet의 파드가 스케줄 되는 방법은 조금 특별한데
  - daemonSet controller가 노드 정보를 파악하고 파드가 떠있지 않은 노드에 파드 생성을 요청한다. 이것이 기본 옵션이다
  - 그런데 문제점은 이렇게 생성되는 daemonSet pod는 일관성이 없다
  - pending 상태가 존재하지 않으며
  - 기본 scheduler의 파드 선점이나 우선순위를 고려하지 않는다.
     - 특정 노드에 공간이 부족하여 스케줄러가 파드를 예약해둔 상태가 있을 수 있는데
     - 이때 daemonSet controller는 이를 무시한다. 일관성이 없다
  - 따라서 기본 scheduler를 사용할 수 있도록 NodeAffinity를 활용한다.


7. deployment
- pod와 replicaSet에 대한 선언적 업데이트를 제공
- deployment에서 의도한 상태를 명시, deployment controller는 현재 상태와 Desired state를 비교하여 desired state로 맞춘다.
- .spec.template이 변경된 경우에만 Deployment update가 발생한다.
- .spec.strategy 는 이전 파드를 새로운 파드로 대체하는 전략을 명시
  - .spec.strategy.type 은 "재생성" 또는 "롤링업데이트"가 될 수 있다. "롤링업데이트"가 기본값이다.
```
참고 ------
- .spec.strategy.rollingUpdate.maxUnavailable
  - 정의: 업데이트 중에 사용할 수 없는 최대 파드 수를 지정합니다.
  - 형태: 절대 숫자(예: 5) 또는 비율(예: 10%)로 설정할 수 있습니다.
  - 기본값: 25%
  - 예를 들어, 전체 파드 수가 10개이고 maxUnavailable을 30%로 설정하면, 최대 3개의 파드가 사용할 수 없는 상태로 만들 수 있다.
    즉, 업데이트 시작 시 기존 파드 중 7개는 항상 작동하고 있어야한다.
- .spec.strategy.rollingUpdate.maxSurge
  - 정의: 업데이트 중에 생성할 수 있는 최대 추가 파드 수를 지정합니다.
  - 형태: 절대 숫자(예: 5) 또는 비율(예: 10%)로 설정할 수 있습니다.
  - 기본값: 25%
  - 예를 들어, 전체 파드 수가 10개이고 maxSurge를 30%로 설정하면, 업데이트 시 최대 3개의 추가 파드를 생성할 수 있다.
    즉, 새로운 레플리카셋의 크기를 13개까지 늘릴 수 있습니다.


쿠버네티스가 availableReplicas 수를 계산할 때 종료 중인(terminating) 파드는 포함하지 않으며,
이 수는 replicas - maxUnavailable 와 replicas + maxSurge 사이에 존재한다.
그 결과, 롤아웃 중에는 파드의 수가 예상보다 많을 수 있으며,
종료 중인 파드의 terminationGracePeriodSeconds가 만료될 때까지는 디플로이먼트가
소비하는 총 리소스가 replicas + maxSurge 이상일 수 있다.

```

- .spec.template.spec.restartPolicy 에는 오직 Always 만 허용되고, 명시되지 않으면 기본값이 된다.
- rollout이란 쿠버네티스에서 애플리케이션의 새로운 버전을 배포하거나 기존 배포를 업데이트하는 과정을 의미
  - `kubectl rollout status deployment/nginx-deployment`
- roll back
  - `kubectl rollout history deployment example-deployment`로 rollout 히스토리를 파악할 수 있다.
  - `kubectl rollout history deploy example-deployment --revision=64`로 특정 리비전 히스토리를 파악하고
  - `kubectl rollout undo deploy example-deployment --to-revision=64`로 특정 리비전으로 롤백

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
