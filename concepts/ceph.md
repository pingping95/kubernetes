# Ceph Concept & Architecture

# 학습 배경

- Kubernetes를 사용하다 보면 rook-ceph, storageclass 에서 ceph Storage를 사용하게 된다. Hands-on을 통해 간단히 사용하여 보았지만 내부적으로 돌아가는 Logic, 흐름을 이해하지 못하여 따로 정리하게 되었습니다.
- 추후에 Ceph에 대한 이해도가 생기고 나면 Ceph Storage를 구축해보고자 합니다.

# Ceph란?

- Ceph는 Ceph Node로 Stoage를 클러스터링 해주는 서비스이다.
- Ceph Object Storage 및 Ceph Block Device에 Service를 Cloud Platform에 제공하든지 Ceph File System을 배포하든지 다른 목적으로 Ceph를 사용하든지 간에 모든 Ceph Storage Cluster 배포는 각각의 Ceph Node를 구축하는 것으로부터 시작한다.
- Ceph Storage Cluster 구성 요소 : C- Monitoring, C- Manager, C- Object Storage Daemon (OSD)가 하나 이상 있어야 하고 Ceph File System Client를 사용하려면 Ceph Metadata Server가 있어야 한다.

## 구성 요소

![Untitled](https://user-images.githubusercontent.com/67780144/94763053-3c0f6200-03e4-11eb-90e1-4c6dec5b51fb.png)


- Monitors : Ceph 모니터는 moniter map, manager map, OSD map, MDS map, CRUSH map 등 Cluster의 상태의 Map을 유지 및 관리한다. 이 Map들은 Ceph 데몬들이 서로 조정하는 데 필요한 중요한 Cluster의 상태이다. Daemon과 Client 간 인증 관리도 monitor가 담당하게 된다. 고 가용성(HA)를 위해 일반적으로 최소 3대의 모니터가 필요하다.

- Managers : Ceph Manager Daemon은 Storage 활용률, 현재 성능 Metric 및 Syetem load를 포함한 Ceph 클러스터의 현재 상태와 Runtime Metric을 추척하는 역할을 한다. 또한 Ceph Manager Daemon은 Web 기반의 Ceph Dashboard와 REST-API를 포함한 Ceph Cluster 정보를 관리하고 노출하기 위해 Pyth 기반 모듈을 호스트한다. 일반적으로 HA를 위해 최소 2개의 Manager가 필요하다.

- Ceph OSDs : Ceph OSD (Object Storage Daemon, Ceph-osd)는 데이터를 저장하고 데이터 복제, 복구, 재조정을 처리하며 다른 Ceph OSD Daemons를 확인하여 Ceph Monitors 및 Managers에 일부 모니터링을 제공한다. HA를 위해 일반적으로 최소 3개의 Ceph OSD가 필요하다.

- MDSs : Ceph`s Metadata Server는 Ceph의 File System (즉, Ceph  Block Devices나 Ceph Object Storage는 MDS를 사용하지 않는다 !!!)을 대신하여 Metadata를 저장한다. Ceph Metadata Servers는 POSIX File System 사용자가 기본 명령 (ls, fine, tec...)을 Ceph Storage Cluster에 엄청난 Burden(부담)을 주지 않고 실행할 수 있다...

## 최소 Hardware 권장 사항

![Untitled 1](https://user-images.githubusercontent.com/67780144/94763056-3d408f00-03e4-11eb-8769-04d8ba5f635d.png)


## Architecture

- RADOS를 기반으로 Date를 R/W
- librados란 Library를 제공하여 RADOS에 직접 접근할 수 있다.
- RadosGW, Ceph FS 와 같은 Ceph clients를 제공하여 Ceph Storage에 접근할 수 있는 Interface를 제공한다.

![Untitled 2](https://user-images.githubusercontent.com/67780144/94763057-3dd92580-03e4-11eb-8a8f-1bff36688f5e.png)


### RADOS (Cpeh Storage Cluster)

- Scalable, Reliable Storage Service
- Ceph의 Data 접근을 할 때 이 RADOS를 사용해야 한다.
- Self-healing, Self-managing, Intelligent Storage Nodes

### Ceph Clients

- 다양한 Service Interface를 제공하여 외부에서 Data를 관리할 수 있도록 한다.

### Object Storage (RadosGW)

- Amazon에서의 S3, Openstack에서 Swift와 Compatible하는 Interface가 있는 RESTful API 서비스를 제공한다.

### Block Device (RBD)

- Snapshot 및 Clone 기능을 가지고 있는 block device 서비스를 제공한다.
- 사이즈 조정이 가능하며 씬프로비저닝 또한 가능하다.

### File System (CephFS)

- Mount 또는 사용자 공간에서 File System으로 사용할 수 있는 POSIX 호환 파일 시스템을 제공한다.

### Pools

- Ceph Storage 시스템은 Object 저장을 위해 Logical한 Partition인 Pools란 개념을 사용한다.

![Untitled 3](https://user-images.githubusercontent.com/67780144/94763058-3e71bc00-03e4-11eb-8ef2-259ae3f89d93.png)


→ Ceph Client는 Ceph Monitor가 가지고 있는 Cluster map을 조회하고 저장할 Object를 Pool에 전달한다. 그럴 시 Pool Size, Replica 갯수, Crush Rule, Placement Group 수에 따라 Ceph가 Data를 저장하는 방법이 결정된다.

![Untitled 4](https://user-images.githubusercontent.com/67780144/94763059-3e71bc00-03e4-11eb-984c-b43e9f146806.png)


→ 각 Pool은 다수의 Placement Group을 가지고 있는데 Ceph Client가 Pool에 Object를 전달할 시 Crush는 Object를 Placement Group에 Mapping 해준다.

### Placement Group (PG)

- PG는 Ceph Client와 Ceph OSD Daemon간 Loosely-Coupling 하는 역할을 맡는다.

    ⇒ Ceph OSD Daemon이 Dynamically하게 추가/삭제 되더라도 Rebalance를 동적으로 할 수 있도록 해준다.

### Ceph Metadata Server (MDS)

- Ceph File system Service에는 Ceph Storage Cluster와 함께 배포 된 Ceph Metadata Server가 포함된다.

### Replication

- Normal Write 시나리오에서 Client는 CRUSH Algorithm을 사용하여 Object를 저장할 위치를 Calculating하고 Object를 Pool 및 PG에 Mapping한 다음 Crush map을 보고 PG의 Primary OSD를 식별한다.

![Untitled 5](https://user-images.githubusercontent.com/67780144/94763060-3f0a5280-03e4-11eb-95bb-6fc8d22e7c50.png)

- Client는 Primary OSD에서 식별 된 PG에 Object를 쓴다. 그 다음 자체 Crush Map 사본이 있는 Primary OSD는 Replication을 위해 2차 및 3차 OSD를 식별하고 2차 및 3차 OSD에서 해당 PG로 Object를 복제하고 Object를 확인하면 Client에 응답한다.
- 위의 사진을 보면 Write, ACK의 과정을 지닌다.

### Shading

![Untitled 6](https://user-images.githubusercontent.com/67780144/94763061-3fa2e900-03e4-11eb-9531-c8cdf52c63a1.png)


- ABCDEFGHI란 Content를 NYAN Object가 Pool에 기록을 할 때 삭제 Encoding 기능은 컨텐츠를 3으로 나누는 것만으로 컨텐츠를 3개의 Data로 Chunk로 분할한다.  Data는 각 ABC, DEF, GHI로 나눠지는데, 내용 길이의 K의 배수가 아닌 경우 내용이 채워진다. 이 함수는 2개의 Coding Chunk로 만든다. 네 번째는 YXY의 경우와 다섯 번째는 QGC의 경우이다. 각 Chunk는 작동 Set의 OSD에 저장된다, Chunk는 이름 (NYAN)은 동일하지만 다른 OSD에 있는 Object에 저장된다. Chunk가 작성된 순서는 보존되어야 하며 이름 외에 Object의 속성으로 정의된다.

### Adding Nodes

![Untitled 7](https://user-images.githubusercontent.com/67780144/94763065-403b7f80-03e4-11eb-9eb0-a177e8fe4b46.png)

- Ceph OSD Daemon을 Ceph Storage Cluster에 추가하면 Cluster map이 새로운 OSD로 Update되고 PG ID를 다시 계산하면서 Cluster Map이 변경된다. 결과적으로 계산에 대한 입력이 변경되므로 Object 배치가 변경된다.
- 위 그림은 OSD 3가 추가되면서 모든 PG가 아닌 일부 PG가 기존 OSD에서 새로운 OSD (OSD 3)로 Migration하는 Rebalancing Process를 볼 수 있다 Rebalancing에도 Crush는 안정적일 뿐 아니라 대부분의 PG는 원래 구성으로 유지되며 각 OSD는 약간의 용량을 추가하므로 재조정이 완료된 후 새 OSD에 Load Spike가 없다.

## Installing 방법

⇒ 요약 : k8s에는 Rook을 통한 ceph 설치를 추천...

- Cephadm : CLI & Dashboard GUI와 긴밀하게 통합되어 Container 및 systemd를 사용하여 Ceph 클러스터를 설치하고 관리한다.
    - cephadm은 Podman 혹은 docker와 같은 Container Support랑 Python 3가 필요하다.
- Rook : Kubernetes에서 실행되는 Ceph Cluster 배포 및 관리하는 동시에 Kubernetes API를 통해 Storage resource 관리 및 Provisioning을 가능하게 한다. Ceph를 Kubernetes에서 실행하거나 기존 Ceph Storage Cluster를 Kubernetes에 연결하는 방법으로 Rook를 추천한다.
    - Rook은 오직 Nautilus와 새로운 버젼의 Ceph를 지원한다.
    - Rook은 k8s에서 Ceph를 실행하거나 k8s Cluster를 기존 Ceph 클러스터에 연결하는 데 선호되는 방법이다.
    - Rook은 새로운 Orchestrator API를 지원한다.
- ceph-ansible : Ansible을 이용하여 Ceph를 배포하고 관리한다.
    - Ceph-ansible은 널리 deploy된다.
    - nautilus와 Octopus에 도입된 새로운 Orchestrator API에는 ceph-ansible이 통합되지 않으며 이는 새로운 관리 기능과 Dashboard 통합을 사용할 수 없다는 것을 의미한다

- Reference (참고 사이트)

    [https://docs.ceph.com/en/latest/start/intro/](https://docs.ceph.com/en/latest/start/intro/)

    [https://yeti.tistory.com/240](https://yeti.tistory.com/240)
