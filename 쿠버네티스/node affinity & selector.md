1. 노드에 파드 할당하기
- 파드가 특정 노드를 선호할 수 있도록 설정이 필요할 수 있다

2. node label & node selector
- 쿠버네티스의 다른 오브젝트와 마찬가지로 노드 또한 label을 가질 수 있다.
- nodeSelector는 파드 스펙에 `nodeSelector` 필드를 추가하여, 타겟으로 삼고 싶은 노드가 갖고 있는 node label을 명시한다.
- 이를 통해 파드가 특정 노드에 `반드시` 바인딩 될 수 있도록한다.
- 하지만 말 그대로 반드시 반인딩이 필요한 경우도 있겠지만, 이는 선호 사항이 아니기 때문에 파드가 뜰 수 없는 상황이 발생할 수 있다.

3. node affinity
- `.spec.affinity.nodeAffinity`
- 말 그대로 `선호`. 파드가 특정 노드를 선호한다.
- 즉 node selector에 비해 soft한 규칙이다.
- 조건에 만족하는 노드가 없더라도 다른 노드로 스케줄링 될 수 있다.
```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0

---
규칙을 살펴보면 파드는
- 노드 중 topology.kubernetes.io/zone이라는 키를 가진 label을 소유, 이때 값은 antarctica-east1 or antarctica-west1에 해당하는 노드를 선호한다.
- 
```
- 참고로 `nodeSelector`와 `nodeAffinity`를 모두 사용한다면, 두 조건에 모두 만족되는 노드에만 스케줄링이 가능하다.
- 참고로 `nodeSelectorTerms`에 n개의 조건을 명시하는 경우 이 중 하나만 만족하는 노드에도 스케줄링 될 수 있다.
- 참고로 `nodeSelectorTerms`의 조건으로 단일 `matchExpressions`에 여러 표현식이 존재하는 경우, 모든 표현식을 만족하는 노드에만 파드가 스케줄링

4. nodeAffinity 가중치
- `preferredDuringSchedulingIgnoredDuringExecution`는 가중치를 설정할 수 있다.
- 가중치의 경우 명시된 가중치 값의 합을 전체 값으로 다룬다
- 스케줄러는 각 노드의 명시된 가중치 + 다른 점수를 더하여 가장 점수가 높은 노드에 파드를 스케줄링한다.
```
apiVersion: v1
kind: Pod
metadata:
  name: with-affinity-anti-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```
