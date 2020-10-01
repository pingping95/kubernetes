# Kubernetes Monitoring 개념, Architecture

- Prometheus : Open source 기반의 Monitoring 시스템

    ELK와 같은 Logging이 아니라 대상 시스템으로부터 각종 Monitoring 지표를 수집하여 저장하고 검색할 수 있는 시스템이다.

    - 구조가 간단
    - 강력한 Query 기능 탑재
    - Grafana를 통한 시각화를 지원

    많은 시스템을 모니터링할 수 있는 다양한 플러그인을 가지고 있는 것이 가장 큰 특징이며 간편함 때문에 k8s의 메인 Monitoring 시스템으로 많이 사용되면서 요즘 더욱 주목받고 있다. Prometheus를 알아보기 전에 Monitoring에 대해 간단하게 살펴보자.

# Monitoring

![Untitled](https://user-images.githubusercontent.com/67780144/94678152-04a7a380-0359-11eb-805b-947dc9e1d76a.png)


## k8s Cluster에서 Data를 수집할 수 있는 Object

- Host : Node의 CPU, Memory, Disk, Network, 사용량, Node os와 Kernel에 대한 모니터링
- Container : 각 컨테이너에 대한 정보, CPU, Memory, Disk, Network 사용량을 모두 모니터링
- Application : 컨테이너에서 구동되는 Application에 대한 모니터링
- Kubernetes : 컨테이너를 Orchestrating하는 k8s 자체에 대한 모니터링 ( Service, Pod, Account 등...)

⇒ Metric (수집 지표) 라고 한다.

### Metrics

- System Metrics : 노드나 컨테이너의 cpu, 메모리 사용량 같은 일반적인 시스템 관련 메트릭.
    - core metric : 쿠버네티스의 내부 컴포넌트들이 사용하는 메트릭 (kubectl top에서 사용하는 메트릭 값)
- Service Metrics : application을 모니터링 하는데 필요한 메트릭

### Pipeline

- Resource metric pipeline : 핵심 요소들에 대한 모니터링 (kubelet, metircs-server, metricAPI 등...)
    - 스케줄러나 HPA 등의 기초자료로 활용
- Full metric pipeline : 클러스터의 사용자들이 필요한 모니터링을 하는데 활용
    - k8s가 관리하지 않고 외부 모니터링 시스템을 연계해서 이용하는 것을 권장
    - 시스템, 서비스 메트릭 둘 다 수집 가능

# Prometheus Monitoring Simple Architecture

![Untitled 1](https://user-images.githubusercontent.com/67780144/94678156-05d8d080-0359-11eb-8df8-6ca6ce44bdbd.png)

## Metric 수집 부분

수집을 하려는 대상 시스템이 Target System이다. MySQL이나 Tomcat 또는 VM과 같이 여러가지 자원이 모니터링 대상이 될 수 있다. 이 대상 시스템에서 메트릭을 프로메테우스로 전송하기 위해서는 Exporter를 사용한다.

### Pulling 방식

: Prometheus가 Target System에서 Metric을 수집하는 방식을 Pulling 방식을 사용함. Prometheus가 주기적으로 Exporter로부터 Metric을 읽어와서 수집하는 방식이다.

- 보통 모니터링 System의 Agent들은 Agent가 모니터링 System으로 Metric을 보내는 Push 방식을 이용한다. ⇒ Push 방식은 Service가 Auto Scailing 등으로 가변적일 때 유리...
- 하지만 Pulling 방식의 경우 만약 VM이 추가 된다면 추가된 VM들은 설정 파일에 ip가 들어있지 않기 때문에 모니터링 대상에서 제외될 수 있다는 단점이 있다.

    ⇒ 이러한 문제에 대한 방안은 Service Discovery로 해결한다. 특정 시스템이 현재 기동중인 서비스들의 목록과 IP 주소를 가지고 있으면 된다. (예 : vm들을 내부 dns에 등록해놓고 새로운 vm이 생성될 때에도 DNS에 등록을 하도록 하면 DNS에서 기동중인 vm 목록을 얻어와서 그 목록의 ip들로 Pulling 하면 되는 구조이다.)

    ⇒ 위의 내용이 Prometheus가 Service Discovery와 통합하는 이유이다.

### Exporter

Monitoring Agent로 Target System에서 Metric을 읽어서 Prometheus가 Pulling할 수 있도록 한다. Exporter는 단순히 HTTP GET으로 Metric을 Text 형태로 Prometheus에게 Return한다. 요청 당시의 Data를 Return 하는 것일 뿐 기존 값 (History)를 저장하는 등의 기능은 없다.

### Retrieval

Service Discovery 시스템으로부터 Monitoring 대상 목록을 받아오고, Exporter로부터 주기적으로 그 대상으로부터 Metric을 수집하는 Module이 Prometheus 내의 Retrieval 이라는 컴포넌트이다.

## 저장

Prometheus 내의 Memory와 Local Disk에 저장된다. 뒷단에 별도의 데이타 베이스등을 사용하지 않고, 그냥 로컬 디스크에 저장하는데, 그로 인해서 설치가 매우 쉽다는 장점이 있지만 반대로 스케일링이 불가능하다는 단점을 가지고 있다. 대상 시스템이 늘어날 수 록 메트릭 저장 공간이 많이 필요한데, 단순히 디스크를 늘리는 방법 밖에 없다.

## 서빙

이렇게 저장된 메트릭은 PromQL 쿼리 언어를 이용해서 조회가 가능하고, 이를 외부 API나 프로메테우스 웹콘솔을 이용해서 서빙이 가능하다. 또한 그라파나등과 통합하여 대쉬보드등을 구성하는 것이 가능하다.

이 외에도 메트릭을 수집하기 위한 gateway, 알람을 위한 Alert manager 등의 컴포넌트등이 있지만, 기본적인 엔진 구조를 이해하는데는 위의 컴포넌트들이 중요하다.

## 추가 Architecture

- Kubernetes monitoring with Prometheus: Architecture overview

![Untitled 2](https://user-images.githubusercontent.com/67780144/94678160-06716700-0359-11eb-878f-475253997fb1.png)

- Monitoring Kubernetes components on a Prometheus stack

![Untitled 3](https://user-images.githubusercontent.com/67780144/94678163-07a29400-0359-11eb-8544-2050253cee05.png)

- Reference

    [https://gruuuuu.github.io/cloud/monitoring-k8s1/#](https://gruuuuu.github.io/cloud/monitoring-k8s1/#)

    [https://blog.naver.com/alice_k106/221521978267](https://blog.naver.com/alice_k106/221521978267)

    [https://bcho.tistory.com/1372](https://bcho.tistory.com/1372)

    [https://sysdig.com/blog/kubernetes-monitoring-prometheus](https://sysdig.com/blog/kubernetes-monitoring-prometheus)
