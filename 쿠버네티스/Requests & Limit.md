1. Request & Limit
- request : 이 만큼은 보장해줘라
- limit : hard limit
  - cpu의 경우 쓰로틀링
  - memory의 경우 OOM

2. 파드와 컨테이너의 리소스 request & limit
- 기본적으로 각 컨테이너에 대한 req / limit을 설정할 수 있다.
```
spec.containers[].resources.limits.cpu
spec.containers[].resources.limits.memory
spec.containers[].resources.limits.hugepages-<size>
spec.containers[].resources.requests.cpu
spec.containers[].resources.requests.memory
spec.containers[].resources.requests.hugepages-<size>
```
- 파드의 Req/Limit은 결국 각 컨테이너에 대한 해당 타입(cpu, memory..) 리소스 req / limit의 총 합
  - 즉 개별 컨테이너에 대해 req/limit을 지정할 수 있지만, 이들의 총합이 파드의 req / limit

3. 쿠버네티스의 리소스 단위
- cpu 리소스는 상대적이 아닌 절대적량으로 표시한다.
- 2, 4, 6 core에서 동작하더라도 500m은 같은 양의 컴퓨팅 파워를 의미한다.
```
- 1 CPU: 1 vCPU, 1 코어 (가상 CPU 또는 물리 CPU의 하나의 코어)
- 1000m: 1 CPU에 해당 (1 CPU = 1000 밀리코어)
- 0.5 CPU: 500m (0.5 코어 또는 500 밀리코어)
  - 이 경우 1코어에 비해 절반의 cpu 타임을 사용할 수 있다는 것
```

4. 쿠버네티스의 메모리 단위
- memory는 바이트 단위로 측정
- 수량 접미사를 사용한다.
```
512Mi: 512 메비바이트 (2의 거듭제곱)
1Gi: 1 기비바이트 (1024Mi = 약 1.07GB)
100M: 100 메가바이트 (십진 단위)
2G: 2 기가바이트 (2000MB)

-------
이진 접미사 (2의 거듭제곱, 1024 단위):
Ki: 키비바이트 (1024 바이트)
Mi: 메비바이트 (1024 KiB = 1,048,576 바이트)
Gi: 기비바이트 (1024 MiB = 1,073,741,824 바이트)
Ti: 테비바이트 (1024 GiB)

----------
십진 접미사 (10의 거듭제곱, 1000 단위):
K: 킬로바이트 (1000 바이트)
M: 메가바이트 (1000 KB = 1,000,000 바이트)
G: 기가바이트 (1000 MB = 1,000,000,000 바이트)
T: 테라바이트 (1000 GB)
```

- 예시
```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:  => 0.25 코어 & 64MB 메모리
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: log-aggregator
    image: images.my-company.example/log-aggregator:v6
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

5. 쿠버네티스가 request가 포함된 파드를 스케줄링 하는 방법
- 각 노드는 파드에 제공할 수 있는 CPU와 메모리양과 같은 각 리소스 타입에 대한 최대 용량이 존재
- 스케줄러는 각 리소스 타입마다 파드(컨테이너 req의 총합) 리소스가 노드 용량보다 작아야 스케줄링 가능

6. 그래서 쿠버네티스가 req / limit을 어떻게 적용하는가? 동작 방식
- 기본적으로 req의 경우 파드가 특정 노드에 스케줄링 되기 전 req을 만족시킬 수 있는지 먼저 확인한 뒤 스케줄한다.
- 그렇다면 파드가 노드에 스케줄된 이후 req/limit은 어떻게 동작하는가?
- linux cgroup
  - kubelet이 파드의 컨테이너를 시작할때 kubelete이 각 컨테이너 메모리/cpu req/limit을 컨테이너 런타임에 전달한다.
- 먼저 request
  - 기본적으로 요청만큼 자원이 존재하는 노드에 파드가 배치되지만, cpu request의 경우 더 큰 값을 가진 컨테이너에 더 큰 가중치가 설정된다.
  - 즉 더 큰 cpu값을 가진 워크로드가 더 많은 cpu 시간을 할당받는다.
- Limit
  - cgroup을 통해 제어한다.
  - cpu의 경우 cgroup을 통해 제어된 수치를 넘는 순간 쓰로틀링이 발생한다. 이로인해 cpu를 사용하지 못하고 cpu 사용량이 낮아지면 다시 cpu를 사용하게된다.
  - 메모리의 경우 limit을 넘으면 OOM 발생
