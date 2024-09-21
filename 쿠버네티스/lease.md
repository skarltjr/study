1. lease란
- 기존! 분산 시스템에는 공유 리소스를 잠그고 노드 간의 활동을 조정하는 메커니즘을 제공하는`리스(Lease)`가 필요하다.
- 쿠버네티스에서! `리스` 개념은 coordination.k8s.io API 그룹에 있는 Lease 오브젝트로 표현되며, 노드 하트비트 및 컴포넌트 수준의 리더 선출과 같은 시스템 핵심 기능에서 사용된다

2. node heart beat
- 쿠버네티스는 리스 API를 사용하여 kubelet 노드의 하트비트를 쿠버네티스 API 서버에 전달한다.
- 모든 노드에는 같은 이름을 가진 Lease 오브젝트가 kube-node-lease 네임스페이스에 존재한다.
- 내부적으로, 모든 kubelet 하트비트는 이 Lease 오브젝트에 대한 업데이트 요청이며, 이 업데이트 요청은 spec.renewTime 필드를 업데이트한다.
- 쿠버네티스 컨트롤 플레인은 이 필드의 타임스탬프를 사용하여 해당 노드의 가용성을 확인한다.

3. 그럼 왜 직접 api server로 요청을 보내는게 아니라 lease api를 활용하는가?
- 노드의 상태 정보는 kubelet에 의해 만들어지는 .status에 기록된다.
- node heartbeat은 이 .status를 업데이트하는것인데, .status는 노드의 모든 정보를 담으니 그만큼 매우 크다.
  - cpu, memory 사용량, 디스크 공간, 네트워크 상태, 이미지, 볼륨 등
- node heartbeat는 kubelet이 10초마다 반복하는데 이걸 그대로 api server에 요청하면 payload가 매우 크고 이를 다루는데 있어 api server 부하가 크며, 이를 etcd에 기록하게 된다면 etcd 용량도 영향을 줄 수 있다.
- 따라서 `잘 살아있다`를 빠르게 확인하는데 목적을 둔 것이 lease

4. 그래서 어떻게?
- 모든 노드에는 아래와 같이 같은 이름을 가진 Lease 오브젝트가 kube-node-lease namespace에 존재
```
> kubectl get lease -n kube-node-lease
NAME                                               HOLDER                                             AGE
ip-10-29-104-121.ap-northeast-2.compute.internal   ip-10-29-104-121.ap-northeast-2.compute.internal   27h
ip-10-29-69-101.ap-northeast-2.compute.internal    ip-10-29-69-101.ap-northeast-2.compute.internal    27h
ip-10-29-79-205.ap-northeast-2.compute.internal    ip-10-29-79-205.ap-northeast-2.compute.internal    88m
```
- 목적은 노드가 살아있음을 업데이트하는것으로 각 kubelet이 각 노드에 존재하는 lease object를 업데이트함으로써 기존 node heartbeat를 대체한다.
- 이를통해 기존처럼 status 자체를 업데이트 하지않으며 node status를 etcd에 기록하는 방식에서 벗어난다.
  - etcd에도 lease object가 기록됨으로.
  - 물론 아예 업데이트하지 않는것은 아니다. 중요 정보는 nodeStatus에 기록되며, 덜 빈번하게 업데이트한다.
  - 대신 controller manager는 nodeStatus가 아닌 lease object를 모니터링하여 노드 상태를 파악한다.

```
동작 방식은
Lease 생성: 노드가 클러스터에 가입되면, kubelet은 그 노드에 대한 Lease 객체를 생성합니다.
Lease 갱신: kubelet은 노드 상태에 변화가 없는 경우에도 Lease 객체를 주기적으로 갱신합니다. / timestamp update
    LeaseDurationSeconds: Lease가 만료되기까지의 기간을 정의합니다. 이 시간이 지나면 노드는 더 이상 정상적으로 작동하는 것으로 간주되지 않습니다.
    RenewTime: Lease가 마지막으로 갱신된 시간을 나타냅니다.
    HolderIdentity: Lease를 소유하고 있는 노드를 식별하는 정보입니다.  
노드 장애 감지: 컨트롤러 매니저는 Lease 객체의 타임스탬프를 모니터링하여 노드의 상태를 판단합니다. Lease가 일정 시간 내에 갱신되지 않으면, 해당 노드는 비정상적이라고 판단됩니다.
```

5. 컴포넌트 수준의 리더 선출과 같은 시스템 핵심 기능에서 사용된다? 는 어떤 경우를 말하는걸까
- ![image](https://github.com/user-attachments/assets/3a982d38-96ce-4c18-baca-e53ea18d4899)
- 기존 분산 시스템에서의 lease 개념과 마찬가지로 lock을 활용하여 공유 리소스 업데이트에도 사용된다.

- https://github.com/kubernetes/enhancements/blob/0460fcf4637d440ce7d539a931e44dae931f9849/keps/sig-node/0009-node-heartbeat.md
  
