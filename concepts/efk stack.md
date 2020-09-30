# EFK Stack

# 개요

- Logging : Debugging 외에 Transaction의 흐름, Error 모니터링, 경고 생성, 문제가 발생할 때 Root Cause Analysis을 수행할 수 있다.

- 중앙 집중식 Logging

응용 프로그램 서버의 Overhead를 줄이고 Log Data를 효과적으로 제어 및 이용할 수 있으며 Log 추적의 복잡성을 없앨 수 있다.

# Logging Architecture : EFK Stack

1. Elasticsearch : 확장 가능한 Realtime 분산 검색 엔진이다. 대량의 로그 데이터를 인덱싱하고 검색하는 데 사용할 수 있다.
2. Fluentd : Collector, Shipper 로써의 역할을 할 수 있다. Shipper로써 Log 데이터를 Elasticsearch로 보내고 Collector로써 로그를 수집하며 최종적으로 Shipper에게 보낸다.
3. Kibana : Elasticsearch에 대한 시각화와 Dashboard를제공하며 Elasticsearch를 쿼리할 때도 사용한다

# 사용 사례

## #1

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled.png)

단순한 접근이다. 대규모 Application Server가 있다고 가정해보자. 여기서 Collector & Shipper 역할을 하는 Fluentd 서버를 서버 내부에 설치할 수 있다. Log를 수집하고 Elasticsearch에게 푸시하며 병목 현상 (bottleneck down the line)이 발생할 수 있다.

## #2

위보다 더 좋은 방식이다. FluentBit을 Application Server 내부에 Log Collector로써 설치한다. FluentBit은 설치 후 몇 mb의 작은 설치 공간을 가지고 있으며 Log Data를 수집하는 FluentD 서버로 전송한다.

- 이 아키텍쳐에서 Fluentd의 역할
    - Elasticsearch의 Shipper 역할을 한다.
    - 병목 현상이 발생하면 임시로 Log Data를 저장하는 Buffer 역할을 할 수 있다.
    - FluentD 내부의 Tag를 기반으로 하는 Routing 규칙을 추가하여 특정 Application 서버에서 오는 Log를 Elasticsearch 내부의 해당 Index로 보낼 수 있다.

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%201.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%201.png)

## #3

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%202.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%202.png)

FluentD에는 복제 (Replication) 혹은 샤딩 (Sharding) 기능이 없다. Multiple Instance를 DNS 뒷 단에 실행할 수 있지만 만약 하나의 Instance가 죽으면 Buffer data를 지닐 것이다. 이러한 시나리오에서 FluentBit의 Log를 Kafka topic으로 Push하고 FluentD가 이 topic을 Subscription하도록 할 수 있다. 여기서 Kafka는 무거운 복제와 Shading를 처리해준다.

🤥Sharding 이란?

- DB 샤드는 데이터베이스나 웹 검색 엔진의 데이터의 수평 (Horizontal Partition)이다. 개개의 파티션은 샤드 혹은 데이터베이스 샤드라고 불리는데 각 샤드는 개개의 데이터베이스 서버 인스턴스에서 부하 분산을 위해 보유하고 있다.
- DB에서 자주 쓰는 용어이며, 하나의 DB에 데이터가 늘어나면 용량 이슈도 생기고, 느려지는 CRUD는 자연스레 서비스 성능에 영향을 줄 수 있다. 그래서 DB 트래픽을 분산할 목적으로 샤딩을 고려해볼 수 있다. DB를 분산하면 특정 DB의 장애가 전면장애로 이어지지 않는다는 장점이 있다.

## #4

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%203.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%203.png)

위 #1, #2, #3의 방법들이 대부분의 Scenarios에서 잘 작동하지만 2가지 Issue가 있다.

1. FluentBit Agent를 Orchestrating하고 있다. 각 Node에서 해당 Agent를 다시 구성하고 업데이터하는 것이 번거로울 수 있다.
2. Application Server의 CPU 주기가 해당 Agent가 Log를 처리하고 FluentD 서버로 전송하는 동안 갈취 당할 수 있다 (보안상의 약점)

- 이러한 Issue를 완화하기 위해 이 Server들이 Log를 직접 쓸 수 있는 별도의 Shared Storage를 장착할 수 있다. 쉽게 확장될 수 있고 agent가 필요하지 않으므로 유지,보수에도 편리하다. 같은 스토리지에 FluentD를 Mount하며 Elasticsearch에 Log를 보내는 메커니즘이다.

# Kubernetes에서의 Logging

## #1

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%204.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%204.png)

이에 관해 Kubernetes는 프로그램 자체가 Pod 내부의 컨테이너에서 실행되며 Application Server가 없다. 이상적으로는 Kubernetes에서 실행되는 Application은 stdout/stderr에 Log를 출력한 다음 Kubernetes가 디렉터리의 Hostnode 내부에 있는 Log File에 기록해야 한다.

- 하지만 Application이 Log file을 작성하는 경우 3가지 옵션이 있다
    1. 위의 경우와 같이 로그 파일을 추적하고이를 stdout / stderr로 리디렉션하여 Kubernetes가 처리 할 수 있도록 Sidecar Container (동일한 포드 내부의 별도 컨테이너)를 시작합니다 .
    2. 로그 파일을 추적하는 사이드카 컨테이너를 시작하고 호스트 노드 볼륨에 다시 기록합니다.
    3. 포드 내부에 호스트 노드 디렉터리를 마운트하고 애플리케이션이 해당 디렉터리에 로그 파일을 작성하도록합니다.
- 핵심은 Log file이 Pod가 실행중인 Host Node의 Volume 내에 있어야 하며 여기서 Logging Agent 이를 선택하고 전달할 수 있어야 한다는 것이다.

## #2

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%205.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%205.png)

FluentBit을 Daemonset으로 시작하여 FluentBit Agent의 Instance가 각 Node에서 실행되고 있는지를 확인한다. FluentD는 Kubernetes 클러스터 내에서 Statefulset으로 실행될 수 있도록 구성할 수 있다.

# Openshift EFK Stack

Openshift는 Kubernetes를 기반으로 하는 기업용 Container Application Platform이다. 컨테이너에서 Log를 수집하고 검색하는 문제를 해결하기 위해 EFK Stack을 사용하여 Log 집계를 배포할 수 있다. Docker log는 각 Node에서 FluentD Process에 의해 수집되고 저장을 위해 Elasticsearch로 전달되며 Kibana를 통해 UI가 제공되는 메커니즘을 갖고 있다.

![EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%206.png](EFK%20Stack%200d2a8fbda2e34819a4dba81d53137ab8/Untitled%206.png)

- Ref

[https://medium.com/@yobitelcommunications/efk-kubernetes-stack-in-cloud-native-way-scope-about-retail-industry-77c526402a27](https://medium.com/@yobitelcommunications/efk-kubernetes-stack-in-cloud-native-way-scope-about-retail-industry-77c526402a27)

[https://access.redhat.com/documentation/en-us/openshift_container_platform/4.1/html/logging/efk-logging](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.1/html/logging/efk-logging)
