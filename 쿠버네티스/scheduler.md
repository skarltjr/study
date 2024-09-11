### scheduler
- 새로운 파드를 어디에 배포해야할까?
- 파드를 배포할 노드를 선택하는 과정과 동작방식을 이해하자
- ![image](https://github.com/user-attachments/assets/8547563d-6815-49b7-a0d3-cb948cb2ca98)

### 대략적인 순서
- ![image](https://github.com/user-attachments/assets/a4c414c4-3b58-4b90-a9cf-873e859de069)
- ![image](https://github.com/user-attachments/assets/2a4df244-c2a8-40e1-863c-836a2611d382)
- ![image](https://github.com/user-attachments/assets/4b215f86-c8c6-4c27-af54-22ea349f36b0)
- ![image](https://github.com/user-attachments/assets/ff2840df-a465-4432-8436-ef43d9f9dd93)

### 동작 원리
- 그럼 적절한 노드를 어떻게 찾는가?
1. filter & score
```
filter
pod 요구사항과 node 상태를 취합하여 배포가능한 노드 목록을 추출한다.
ex. resource(cpu,mem,..etc)가 충분한가?
ex. pod가 지정하는 nodeSelector에 충족하는가
ex. node가 schedulable한 상태인가?
```
```
score
조건에 부합하는 노드가 여러개일때 가장 최적의 노드를 선정하기 위해 점수를 계산한다
ex. 기준은 충족하는 node가 여러개이지만 가장 resource가 널널한 노드에 배점
ex. pod가 사용하는 container image 존재여부에 따른 배점
```
✅ 스케줄링은 blocking 작업으로 한 번에 한 작업이 완료될때까지 기다린다.
✅ 바인딩은 non-blocking

2. binding cycle : 적절한 노드를 선택하여 binding한다.
- pod가 특정 node에 할당되었음을 kube-apiserver에게 알린다.
- 이때 이 binding cycle이 병목이되지 않도록 비동기로 동작하여 binding cycle 완료를 기다리지 않고 다음 pod의 binding cycle을 시작한다.
- ![image](https://github.com/user-attachments/assets/a6d6851b-66ed-4142-87a4-3f6cfe1f0ee8)

### 문제 : Race condition
- ![image](https://github.com/user-attachments/assets/ab20537a-2fc0-4711-96ab-244f0ea722f5)
- ![image](https://github.com/user-attachments/assets/3eed3d07-7fee-4991-bda6-629eab46d9de)
- ![image](https://github.com/user-attachments/assets/faa3e46b-13a8-4f38-88d4-0f232f94719b)
- ![image](https://github.com/user-attachments/assets/5973ff77-7292-4c03-87da-5463418aca6b)

### Optimistic binding
- pod를 노드에 배치하는것이 실패할 수 있음을 전제하에 진행한다.
- scheduling(filter & score) 후 배치할 노드가 정해지면 스케줄러는 이를 cache에 써놓는다(ex.pod1 - node1)
- 만약 race condition 문제로 배치에 실패했을때 노드는 위 cache를 revert한다
- cache가 revert 되었으니 스케줄러는 다시 파드를 배치하도록 스케줄링을 진행한다.

### 스케줄러의 동작 흐름
- ![image](https://github.com/user-attachments/assets/bcd6f2cd-4ecc-4f4f-a7c8-a2ae40b458f6)
1. event handler는 kube-api를 통해 새로운 pod 생성 이벤트 발생을 감지한다.
2. event handler는 pod에 노드가 할당되지 않았으니 scheduler queue에 추가한다.
  - 이때 cache에 pods, nodes 정보를 캐싱한다
3. scheduler는 queue의 작업을 scheduling cycle 시작
  - 이때 scheduling cycle은 동시에 여러 pod를 처리하지 않는다 (blocking)
4. 만약 scheduling cycle간 에러가 발생하면 다시 queue insert
5. scheduling cycle 후 pod를 배치할 노드가 정해지면 cache에 pod가 특정 node에 배치된것처럼 캐싱해둔다.
6. binding cycle을 통해 kube-apiserver에게 해당 pod가 특정 node에 배치되었음을 알린다.
  - binding cycle은 non blocking으로 이때 동시에 다른 pod의 scheduling cycle은 동작한다.
7. 만약 binding cycle간 에러가 발생하면 파드가 노드에 배치되었다고 캐시에 적어둔것을 revert하고 다시 큐에 작업을 집어넣는다.

