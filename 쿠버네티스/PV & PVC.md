![image](https://github.com/user-attachments/assets/1215a84d-5a5d-4467-b532-f0cc94be6526)

### Persistent Volume, Persistent Volume Claim
- pv 자체는 쿠버네티스 클러스터의 리소스로 volume plugin으로 스토리지 시스템을 추상화하는 역할.
- 그 자체로써 호스트 머신에 영향을 주진 않고, 호스트 머신이나 스토리지 시스템에 연결되었을때 영향을 준다.
- 쿠버네티스는 볼륨을 바로 파드에 직접 할당하는것이 아닌 pvc를 통해 pv<->storage에 연결되어 있는 pv와 pod를 연결한다
  - `pod<->pvc <-> pv<->storage`

### pv와 pvc의 생명주기
1. provisioning
- static provisioning
  - 정적으로 pv를 미리 관리자가 만들어두고 사용자의 요청이 있으면 미리 만들어둔 pv를 할당
  - storage에 용량 제한이 있을때 유용
  - 미리 만들어둔 pv의 용량이 100GB라면 150GB 요청은 실패
- dynamic provisioning
  - pvc를 거쳐 pv를 요청할때 pv를 생성하여 제공
  - 스토리지가 충분하다면 원하는 용량만큼 생성해서 사용가능
  - pvc는 동적 프로비저닝시 여러 스토리지중 원하는 스토리지를 정의하는 storage class로 pv를 생성

2. binding
- provisioning 단계를 거쳐 생성된 pv와 pvc를 연결
- pvc에 명시된 스토리지 용량, 접근 방법에 맞춰 pv가 할당
- pvc에서 원하는 pv가 없다면 요청 자체는 실패하지만 pvc는 pv가 생성될때까지 대기하다 바인딩
- pv 1:1 pvc로 1대1 매핑관계이기 때문에 pvc하나가 여러 pv에 할당될 수 없다

3. using
- PVC는 파드에서 볼륨처럼 사용.
- 즉, 파드는 PVC를 통해 Persistent Volume(PV)에 연결된 스토리지를 마치 로컬 볼륨처럼 사용
- 파드 사양에서 PVC를 지정하면 쿠버네티스가 해당 PVC를 참조하여 필요한 스토리지를 파드에 연결
- PVC가 파드와 연결된 상태에서는 PVC를 사용하는 동안 쿠버네티스가 임의로 PVC를 삭제할 수 없다. 
  - 이는 스토리지 오브젝트가 실제로 사용 중일 때 실수로 삭제되는 것을 방지하기 위한 보호 기능
  - Storage Object in Use Protection은 PVC가 파드에서 사용되고 있을 때 PVC가 실수로 삭제되는 것을 방지하는 쿠버네티스의 기본 보호 메커니즘

4. reclaiming(반환)
- 사용이 끝난(파드 삭제, pvc 제거) pvc는 삭제되면서 pv를 초기화(reclaim)하는 과정을 거친다.
- 초기화 정책에는 retain, delete, recycle이 존재
- retain
  - pv는 삭제하지만 연결된 스토리지의 데이터(볼륨)은 유지
  - 이 정책에서 pvc가 삭제되면 pv는 release되어 다른 pvc가 재사용 할 수 없다.
  - 이걸 재사용하기 위해선
    - pv 삭제(연결된 스토리지에 데이터는 그대로 보존된다)
    - 해당 볼륨을 연결하는 pv 재생성
- delete
  - pv 삭제 및 연결된 외부 스토리지의 볼륨도 제거
  - default option
- recycle
  - pv의 데이터들을 삭제하고 새로운 pvc에서 이 pv를 사용할 수 있도록한다
  - 쿠버네티스에서 더 이상 사용되지 않는다
