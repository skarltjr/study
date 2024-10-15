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

### StorageClass
- storageClass란 쿠버네티스에서 스토리지의 class를 정의할 수 있는 객체
- storage의 class란 성능 요구사항, 백업 정책 혹은 정책을 의미하는데, 즉 이를통해 파드가 스토리지를 요청할 때 다양한 요구사항을 만족할 수 있도록한다.
- StorageClass의 주요 기능
  - 동적 스토리지 프로비저닝:
    - StorageClass는 쿠버네티스 클러스터 내에서 PVC(Persistent Volume Claim)가 요청되었을 때, 스토리지를 동적으로 생성하는 방법을 정의
    - 사용자가 명시적으로 Persistent Volume (PV)을 먼저 생성하지 않아도, PVC를 통해 StorageClass에 정의된 프로비저너가 자동으로 PV를 생성
  - 스토리지 설정 정의:
    - StorageClass는 스토리지의 종류(예: AWS EBS, Google Persistent Disk, NFS 등)와 각 스토리지에 대한 설정 옵션(예: 성능, 용량, 복제본 수 등)을 정의
    - 이를 통해 사용자나 개발자는 스토리지의 세부 설정을 관리할 필요 없이, StorageClass만으로 스토리지를 자동으로 프로비저닝
  - 스토리지 프로비저너 지정:
    - StorageClass에서 중요한 속성 중 하나는 프로비저너(provisioner). 프로비저너는 특정 스토리지 시스템(AWS, GCE, Azure 등)에 맞춰 PV를 동적으로 생성하고 관리하는 CSI 드라이버나 내부 프로비저너를 의미합니다.
  - Reclaim Policy 설정:
    - StorageClass는 PV가 사용 중지되었을 때 해당 스토리지를 어떻게 처리할지 결정하는 Reclaim Policy(Retain, Delete)를 정의할 수 있습니다. 이는 PVC가 삭제된 후 PV와 그 스토리지를 어떻게 처리할지를 제어하는 역할을 합니다.
  - 파라미터 설정:
    - StorageClass는 스토리지 프로비저너에게 전달할 **파라미터(parameters)**를 설정할 수 있습니다. 이 파라미터들은 스토리지 유형별로 다를 수 있으며, 예를 들어 클라우드 블록 스토리지의 경우 성능 유형(IOPS), 스토리지 크기, 복제본 수 등 세부 설정을 포함할 수 있다.
