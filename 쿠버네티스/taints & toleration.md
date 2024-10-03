1. taint & toleration
- taint는 노드의 설정으로, 노드가 파드셋을 제외시킬 수 있다. 즉 설정한 노드에는 pod가 스케줄되지 않는다.
  - node affinity의 경우 파드의 속성으로 특정 노드를 끌어들이는 속성
- toleration은 파드에 적용되는것으로 위와 같이 노드에 설정된 taint를 무시할 수 있는 속성
  - 무시할 수 있다는것이지 반드시 특정 노드로 스케줄됨을 보장하는것이 아니다
- 목적은 파드가 부적절한 노드에 스케줄되지 않도록 하는 것
- 참고로 각 노드에는 하나 이상의 taint가 적용되는데, 이는 특정 수 이상의 파드를 수용하지 못한다는 taint

2. conecpt
- `kubectl taint nodes node1 key1=value1:NoSchedule`
  - node1에 taint를 묻힌다. 이제 이 노드는 key1의 값이 value1인 파드는 이 노드에 배치할 수 없다.
- 다시 taint를 제거하려면 `kubectl taint nodes node1 key1=value1:NoSchedule-`
- pod에 toleration 추가는
```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"

혹은

tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
-----
pod 구성 예시
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```
- 참고로 operator가 지정되지 않은 경우 기본값은 Equal이다
  - operator가 exists인 경우는 key 포함 여부만 확인하기 때문에 value는 지정하면 안된다
  - operator가 Equal인 경우 key, value가 모두 같아야 의도한 동작이 수행된다.

3. 예시
```
1. kubectl taint nodes node1 key1=value1:NoSchedule
2. kubectl taint nodes node1 key1=value1:NoExecute
3. kubectl taint nodes node1 key2=value2:NoSchedule
```

```
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```
- 파드는 node1에 스케줄링 될 수 있을까?
  - 없다
- 반드시 노드 taint에 대한 toleration이 있는 경우에만 파드는 노드에 스케줄링 될 수 있다.

4. UseCase
- taint and toleration은 파드를 노드에서 멀어지게 하거나, 실행되지 않아야하는 파드를 축출할 수 있는 유연한 방법
- 예를들어 특정 그룹에만 전용 노드를 할당하고 싶다거나, gpu 전용 노드에 특정 파드만 올리고 싶다던가

5. taint 기반 Eviction
- `NoSchedule`의 경우
  - taint가 나중에 추가되어 이미 실행중인 파드는 영향을 받지 않고 계속 동작
  - 물론 죽었다 다시 스케줄링 될 땐 불가능
- `NoExecute` 의 경우
  - 이미 실행중인 파드는 NoExecute에 대한 toleration이 없으면 즉시 축출
```
물론 이때 tolerationSeconds도 추가한다면 3600초 이후 축출된다는 것으로 그 전에 taint를 제거하면 파드는 축출된지 않는다.
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600

```


6. condition 기반 노드 taint
- 노드의 상태가 이상한 경우 컨트롤 플레인은 노드에 taint를 추가한다.
- 예를들어 node에 DiskPressure 컨디션이 활성화되는 경우 컨트롤 플레인은 노드에 `node.kubernetes.io/disk-pressure` 테인트를 추가한다
  - 그럼 스케줄러는 스케줄링시 노드의 상태를 직접 확인할 필요없이 taint를 통해 새로 생성된 파드가 해당 노드로 스케줄되지 않도록 할 수 있다.
- 참고로 데몬셋 컨트롤러의 경우 데몬에 `NoSchedule` toleration을 자동으로 추가한다.
  - 데몬셋은 기본적으로 모든 노드에 뿌려져야하는데 노드에 noschedule taint가 있으면 뿌려질 수 없으니
  - 물론 특정 노드는 추가 taint를 통해 daemonset도 막을 수 있겠다




