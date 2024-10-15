### Container Storage Interface
```
csi란 컨테이너 스토리지 드라이버를 추상화하여 
컨테이너 오케스트레이션 시스템에서 Ceph, EBS 등 특정 스토리지 벤더나 시스템에 종속되지 않고
다양한 스토리지 시스템에 접근 및 관리할 수 있는 표준
```

### CSI의 핵심 목표
- 스토리지 시스템의 통합을 단순화:
  - CSI는 다양한 스토리지 시스템(예: AWS EBS, Google Cloud Persistent Disk, Ceph, NFS 등)을 컨테이너 오케스트레이션 플랫폼에 통합하는 방법을 통일된 방식으로 제공
  - 이를 통해 새로운 스토리지 솔루션을 지원할 때마다 별도로 구현할 필요 없이, 표준화된 방식으로 스토리지 드라이버를 개발 가능
- 오케스트레이션 플랫폼과 스토리지 시스템의 독립성:
  - CSI는 컨테이너 오케스트레이션 플랫폼(쿠버네티스, Docker Swarm 등)과 스토리지 시스템을 분리하여 관리
  - 이를 통해 오케스트레이션 플랫폼과 상관없이 스토리지 드라이버를 작성하고 배포 가능
- 확장 가능성:
  - CSI는 스토리지 시스템이 새로 등장하더라도 쉽게 통합될 수 있도록 확장성을 고려해 설계
```
개인적인 생각
- 결국 통일된 api interface를 제공하고 각 스토리지 시스템에 맞게 서로 다른 구현체(스토리지 드라이버)가 존재할 것
- 코드 관점에서 보자면 우리는 추상화에 의존하되, 연결하고자하는 스토리지 시스템에 해당하는 구현체를 주입받아 사용할듯
``` 

### CSI의 주요 기능
- 볼륨 생성(CreateVolume): 스토리지 시스템에서 새로운 스토리지 볼륨을 생성합니다.
- 볼륨 삭제(DeleteVolume): 기존의 스토리지 볼륨을 삭제합니다.
- 볼륨 마운트(ControllerPublishVolume, NodeStageVolume, NodePublishVolume): 특정 노드(서버)에 스토리지 볼륨을 연결하고 마운트하여 컨테이너가 사용할 수 있도록 합니다.
- 볼륨 언마운트(ControllerUnpublishVolume, NodeUnstageVolume, NodeUnpublishVolume): 노드에서 스토리지 볼륨을 연결 해제합니다.
- 볼륨 확장(ExpandVolume): 이미 생성된 스토리지 볼륨의 크기를 확장합니다.
- 볼륨 스냅샷(Volume Snapshot): 볼륨의 스냅샷을 생성하거나 스냅샷을 복구합니다.


### CSI 이전의 쿠버네티스에서의 동작
- ![image](https://github.com/user-attachments/assets/07def947-6fa4-4dad-930d-bdbb8e2c523e)
- 쿠버네티스 코어에 volume plugin이 존재
- 이로 인하여
```
- volume plugin이 쿠버네티스 코어 시스템에 강하게 결합
  - volume plugin과 다른 코어 시스템이 서로에게 영향
  - volume plugin 버그가 다른 코어 시스템에 영향
  - volume plugin이 kubelet, kube-controller-manager등과 같은 주요 쿠버네티스 컴포넌트와 같은 권한을 가짐
- 다양한 스토리지 시스템에 대한 모든 구현체를 유지해야함
```

### CSI 도입
- 코어 시스템에서 분리 / external component
- ![image](https://github.com/user-attachments/assets/2cfeb5c0-4bd9-46ef-9b9e-1ae561d6032d)
- 각 노드에 데몬셋으로 csi driver 컨테이너가 뜨는데 이때 이 컨테이너는 사이드카 컨테이너로 driver registrar와 함께 뜬다.
- external provisioner, external attacher는 컨트롤 플레인에서 동작하는 컨테이너
- `driver registrar`
  - 역할:
    - 쿠버네티스가 deamonset으로 각 노드에 떠 있는 CSI 드라이버를 인식하고, 해당 드라이버가 노드에서 제대로 작동하도록
    - Driver Registrar는 CSI 드라이버를 쿠버네티스의 kubelet에 등록하는 역할
    - 또한 CSI 드라이버가 사용하는 NodeId를 쿠버네티스 노드 API 객체에 라벨로 추가
  - 상세 동작:
    - Driver Registrar는 CSI 드라이버의 Identity 서비스와 통신하여 해당 노드의 고유한 식별자인 NodeId를 가져온다
    - 이 NodeId를 쿠버네티스의 노드 객체(Node API Object)에 라벨로 추가하여 쿠버네티스는 해당 노드가 어느 CSI 드라이버와 연결되어 있는지 알 수 있음
- `external provisioner`
  - 역할:
    - pvc 요청을 감지하고 이를 csi driver에게 전달하여 volume 생성(CreateVolume API) 및 삭제(deleteVolume API)
- `external attacher`
  - 역할:
    - VolumeAttachment를 감시하여 볼륨을 특정 노드에 연결 혹은 해제
```
그럼 파드가 재생성되어 서로 다른 노드에 뜰때는?
- 죽을때 external attacher가 csi driver에게 unpublishVolume요청 전달하여 csi driver가 volume을 삭제하는게 아니라 detach
- 다시 뜰 때 새로운 노드에 뜨는 경우
- external provisioner는 이미 볼륨이 존재하기 때문에 동작하지 않는다.
- external attacher가 기존 볼륨을 새로운 파드에 attach
- 파드 생성
```


