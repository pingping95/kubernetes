# Prometheus + Grafana

# 개요

- Kubernetes Monitoring 툴로써 거의 표준화 되어있는 Prometheus와 시각화 툴인 Grafana를 연동하는 실습을 진행해보았다.

# Prerequisites

- Host OS : Ubuntu 18.04
- Hypervisior : KVM
- 가상머신
    - k-master
        - 192.168.100.10/24, RAM : 3GB, 2 vCPU
    - k-worker01
        - 192.168.100.20/24, RAM : 2GB, 2 vCPU
    - k-worker02
        - 192.168.100.21/24, RAM : 2GB, 2 vCPU
- Kubernetes Cluster Version : 1.19.3

# Architecture

- Kubernetes 클러스터의 Metric 값 → Prometheus가 수집 → Grafana로 시각화 (Web UI)

![prometheus](https://user-images.githubusercontent.com/67780144/98873670-28174f80-24bc-11eb-90ac-67c6bcc83620.png)

# 전체적인 순서

1. Prometheus
    - namespace 생성

        ⇒ prometheus 만의 논리적인 공간을 할당해준다.

    - ClusterRole, ClusterRoleBinding, ServiceAccount 생성

        ⇒ Kubernetes Cluster 내의 api에 접근할 수 있는 권한을 Prometheus가 부여받기 위한 작업

        ⇒ ClusterRole ↔ ClusterRoleBinding ↔ ServiceAccount

        ⇒ ClusterRoleBinding은 바인딩만 해주는 api 리소스이므로 namespace를 지정하지 않음

    - ConfigMap 생성

        ⇒ Configuration File 정의

        ⇒ prometheus.rules : Metric에 대한 Alarm 조건을 지정, 특정 조건 달성 시 AlertManager로 알람을 보냄

        ⇒ prometheus.yml : Metric의 종류, 수집 주기

    - Deployment 생성
    - Service 생성
2. node exporter

    → Kubernetes 기본 System metric 외의 것들을 수집하기 위해 Agent를 따로 둠

    → 각각 하나씩 DaemonSet으로 띄워줌

    - DaemonSet 생성
    - Service 생성
3. kube-state-metrics

    → 쿠버네티스 클러스터 내 Object (Pod, ..)에 대한 지표 정보를 생성하는 서비스

    → Pod 상태 정보를 Monitoring하기 위해 kube-state-metrics가 있어야 함

    - ClusterRole, ClusterRoleBinding, ServiceAccount 생성
    - Deployment, Service 생성
4. Grafana
    - Deployment, Service 생성
5. Prometheus - Grafana 연동

    Grafana Dashboard에서 Prometheus의 정보를 넣어주어야 한다.

    Endpoint는 Prometheus Service의 ClusterIP를 기입해주어야 서로 파드 간의 통신이 가능하다.

6. Dashboard 추가

# 설치

- yaml 디렉터리 내부에 grafana, kube-state, prometheus 디렉터리가 있다.
- 각각 yaml 파일로 정의해주고 kubectl create -f << ~~.yaml >> 해주면 된다.

```bash
## prometheus directory
kubectl create -f prometheus-cm.yaml 
kubectl create -f prometheus-crb.yaml 
kubectl create -f prometheus-cr.yaml 
kubectl create -f prometheus-deployment.yaml 
kubectl create -f prometheus-svc.yaml
kubectl create -f prometheus-node-exporter.yaml

## kube-state directory
kubectl create -f kube-state-crb.yaml
kubectl create -f kube-state-cr.yaml
kubectl create -f kube-state-sa.yaml
kubectl create -f kube-state-deployment.yaml
kubectl create -f kube-state-svc.yaml

## grafana directory
kubectl create -f grafana-deployment.yaml
kubectl create -f grafana-svc.yaml
```

- tree로 yaml 파일 확인

```bash
[root@k-master monitoring]# tree
.
├── grafana
│   ├── grafana-deployment.yaml
│   └── grafana-svc.yaml
├── kube-state
│   ├── kube-state-crb.yaml
│   ├── kube-state-cr.yaml
│   ├── kube-state-deployment.yaml
│   ├── kube-state-sa.yaml
│   └── kube-state-svc.yaml
└── prometheus
    ├── prometheus-cm.yaml
    ├── prometheus-crb.yaml
    ├── prometheus-cr.yaml
    ├── prometheus-deployment.yaml
    ├── prometheus-node-exporter.yaml
    └── prometheus-svc.yaml

3 directories, 13 files
```

# Test

- Prometheus server 접속

    nodePort를 통해 접속

![Untitled](https://user-images.githubusercontent.com/67780144/98873674-29487c80-24bc-11eb-8912-bd5df7f84cad.png)

- kube-state-metrics

    Prometheus 서버 상단 메뉴 → Status → Target

    kube-state-metrics Healthy 한 것을 확인

![Untitled 1](https://user-images.githubusercontent.com/67780144/98873676-29e11300-24bc-11eb-881b-d5cdc9512ff6.png)

- Grafana

    nodePort를 통해 접속

![Untitled 2](https://user-images.githubusercontent.com/67780144/98873678-2a79a980-24bc-11eb-8308-fa3539d453a9.png)

- Grafana - Prometheus 연동 작업
    - 연동은 Grafana의 Dashboard에서 Prometheus의 Endpoint를 기입해주면 된다.
    - Pod와 Pod 간의 통신이므로 ClusterIP를 기입해주어야 한다.

    ```bash
    [root@k-master monitoring]# kubectl get svc -n monitoring 
    NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    grafana              NodePort   10.111.13.96     <none>        3000:30004/TCP   14m
    prometheus-service   NodePort   10.104.146.174   <none>        8080:30003/TCP   35m
    ```

![Untitled 3](https://user-images.githubusercontent.com/67780144/98873679-2b124000-24bc-11eb-9800-93a449926ea1.png)

- Dashboard를 Import해주자.

    Grafana 홈페이지 >> Dashboard로 이동하여 마음에 드는 Dashboard를 가져옴.

![Untitled 4](https://user-images.githubusercontent.com/67780144/98873681-2baad680-24bc-11eb-8a9a-2d67899c21dc.png)

- 최종 Test

![Untitled 5](https://user-images.githubusercontent.com/67780144/98873684-2baad680-24bc-11eb-8f6f-153e6232a513.png)

- Reference

[https://gruuuuu.github.io/cloud/monitoring-02/#](https://gruuuuu.github.io/cloud/monitoring-02/#)