1. 컨테이너
- 올인원 패키징
- 자원 보장
- 격리

1-1. 올인원 패키징
- 일관성 보장 : 컨테이너 이미지를 통해 애플리케이션 실행을 위해 필요한 종속성을 정의하여, 실행에 있어서 항상 동일한 환경을 보장한다.
  - 운영 체제 (ex. FROM ubuntu)
  - 애플리케이션 종속성(ex. python's requirements.txt)
  - 애플리케이션

1-3. 자원 보장
- control group(cgroup)
  - 리눅스 커널 기능으로, 하나 이상의 컴퓨팅 리소스 및 장치를 묶어서 그룹화
    - cpu, mem, disk i/o, network
  - 리눅스는 결국 모두 파일시스템이며, 이러한 cgroup도 파일로 관리된다.
    - `/sys/fs/cgroup/cpu/xxx`
- 컨테이너에 자원 할당은 어떻게 수행되는가?
  - 호스트에서 수행된다.
  - 컨테이너 실행전 각 컨테이너마다 cgroup을 생성 및 수치 조정
  - unshare를 통해 네임스페이스 격리 (ex. unshare -m -u -i -fp)
  - pid namespace를 포함한 네임스페이스가 격리되었으니
  - 격리된 pid namespace에서 1번 프로세스에 cgroup 매핑 (ex. `echo "1" > /sys/fs/cgroup/cpu/red/cgroup.procs`)
  - 격리된 pid 네임스페이스에서 1번 프로세스는 추후 컨테이너의 1번 프로세스
- 컨테이너 내부에서 이 파일을 수정하면 자원 관리가 안되는거 아닌가?
  - 직접 해보면 readonly로 막혀있는걸 확인할 수 있다.
  - `echo "1" > /sys/fs/cgroup/cpu/cgroup.procs` readonly라 컨테이너 내부에서 실행할 수 없다.
- cgroup과 네트워크 자원 관리
  - cgroup의 `net_cls(network classifier)`는 네트워크 트래픽을 분류하고 제어할 수 있도록, 네트워크 패킷에 클래스 ID(Class ID)를 부여하는 기능을 제공
    - 호스트의 tc(traffic control)을 통해 특정 ID / 특정 cgroup에 속한 트래픽 필터링
  - cgroup의 `net_prio`는 네트워크 인터페이스별로 트래픽의 우선순위를 설정할 수 있는 기능을 제공
    - 호스트의 네트워크 인터페이스별 네트워크 자원 사용 우선순위 설정등이 가능하다. 
