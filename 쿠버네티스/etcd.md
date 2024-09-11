1. etcd란
```
etcd는 오픈소스 “분산 키/밸류 스토어”로 분산시스템의 동작을
지속하는데 필요한 중요한(critical) 정보들을 관리하는데 사용됩니다.

주목할만한 점은 etcd가 가장 인기있는 컨테이너 오케스트레이션 플랫폼인
쿠버네티스의 설정데이터, 상태데이터, 메타데이터들을 관리합니다.
```

2. etcd의 주요 특징
```
1. Fully replicated
Every node in an etcd cluster has access the full data store
Stores multiple copies of each database fragment at multiple sites
```
```
2.Highly available
etcd is designed to have no single point of failure and
gracefully tolerate hardware failures and network partitions
```
```
3. Reliably consistent
Every data ‘read’ returns the latest data ‘write’ across all clusters.
```
```
4. Fast
etcd has been benchmarked at 10,000 writes per second
```
+ etcd의 성능은 디스크 성능과 아주 밀접하기 때문에 ssd 사용을 권장

3. redis vs etcd
- 성능 : redis
- 하지만 etcd는 fault tolerance와 failover, 그리고 지속적인 데이터 가용성 측면에서 더 뛰어나고, 가장 중요하게는, etcd는 본질적으로 속도를 희생하는 대신에 더 높은 신뢰성과 일관성을 보장하기 위해 모든 데이터를 디스크에 저장합니다.

### etcd가 사용하는 Raft 알고리즘
- Raft Algorithm은 voting과 log entry 복제라는 방식으로 동작
- 목표 : etcd 클러스터내 어느 한곳에서 발생하는 이벤트는 클러스터내 구성원 모두에게 전파돼야한다.
  - Replicated state machine : 상태 머신 복제 방법은 동일 데이터를 여러 서버에 계속해서 복사하는 방식
  - Consensus Algorithm : 어떤 데이터를 복사할지 결정하는 알고리즘
### Replicated state machine 동작 방식

#### 기본 개념
- consensus 확보 전제 사항
```
항상 올바른 결과를 리턴해야 함
서버가 몇 대 다운되더라도 항상 응답해야 함
네트워크 지연이 발생하더라도 로그의 일관성이 깨져서는 안됨
모든 서버에 복제되지 않았더라도 조건을 만족하면 빠르게 요청에 응답해야 함
```
- quorum
```
정족수
의사 결정을 하기 위한 최소한의 서버 수
etcd 서버들 중 quorum만큼 서버에 데이터 복제가 일어나면 완료로 판단
```
- state
```
etcd cluster를 구성하는 서버들의 role
- leader
- follower
- candidate

모든 서버들은 마지막으로 log가 저장된 위치 lastindex를 갖고 있다
leader만 추가적으로 현재 etcd cluster의 서버들이 log를 저장할 다음 위치 nextIndex를 갖는다
etcd cluster내 변경 내역은 leader를 통해서만 일어나고 follower에게 전파된다
```
- timer
```
leader는 다른 서버들에게 자신이 살아있음을 알리는 heartbeat를 주기적으로 전송
다른 서버들은 leader에게 log를 저장할 다음 위치를 전달받으며 동작한다
leader가 죽어서 heartbeat를 보내지못하면 새로운 leader 선출 작업을 진행한다.
```

#### 리더 선출 / leader election
1. 클러스터 구성시 모든 서버는 follower로 시작. 이때 Term = 0
2. 리더가 없으니 heartbeat를 보내지않고 서버 중 어느곳에서 election timeout발생. 리더 선출이 필요함을 인지
3. election timeout이 발생한 서버는 state를 candidate로 변경하고 Term += 1 & 다른 서버들에게 RequestVote RPC call
4. Request Vote를 받은 서버는 자신이 가진 term정보와 log를 비교해서 candidate보다 자신의 것이 크다면 거절, 크지 않다면 OK 응답
5. Candidate는 자기 자신을 포함하여 다른 Server로 부터 받은 OK응답의 숫자가 Quorum과 같으면 Leader가 됨
6. 이후 리더는 주기적으로 서버들에게 heartbeat를 보낸다. 이때  Append RPC call에는 Leader의 term과 log index정보가 들어있음
7. Leader가 아닌 서버는 Append RPC call을 받았을 때, 자신의 term보다 높은 값인지 확인 후 자신의 term값을 해당 값으로 업데이트

#### 로그 복제 / log replication
- etcd는 로그복제를 통해 모든 etcd가 동일한 값을 갖도록 유지
1. 각 서버는 가장 최근 로그를 저장한 마지막 위치에 대한 lastIndex값을 갖고있다. 이때 leader만 다음 로그를 저장할 nextIndex값을 가진다
2. log append 요청을 받은 리더는 lastIndex에 로그를 저장하고 lastIndex += 1
3. heartbeat를 통해 모든 서버들에게 AppendEntry RPC call을 보내어 follower들에게 nextIndex에 해당하는 log를 함께 보냄
4. follower들은 log를 저장하고 이에 대한 응답을 전송
5. leader는 follower들에게 받은 응답의 수가 quorum만큼 도달하면 정상 전파되었다고 판단하고 commit 수행
6. follower들도 leader가 해당 log를 commit했다는 사실을 전파받은뒤 자신들도 모두 commit

#### 리더 다운
1. 앞서 리더 선출과정처럼 heartbeat가 없으면 election timeout이 발생하고 타임아웃이 발생한 서버는 term값을 1증가시킨뒤 candidate로 변경
2. 해당 서버가 다른 서버들에게 request vote rpc call
3. follower들은 자신의 term값과 전달받은 term값 비교하여 응답
4. 만약 candidate가 다른 follower로부터 정상 응답받은 개수가 quorum이상이면 리더로 선출
5. 그렇지 못하면 기각되고 다른 서버에서 election timeout 발생후 동일 과정 반복

