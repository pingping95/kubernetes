# Trouble Shooting

# 1. Application Failure

- Service Status 체크

```bash
curl http://web-service-ip:node-port

kubectl describe service web-service
```


- Pod 체크

```bash
kubectl get pods

kubectl describe pod web

kubectl logs web -f --previous
```


# 2. Control Plane Failure

- Node Status 체크

```bash
kubectl get nodes

kubectl describe nodes
```

- Control Plane Pods 체크

```bash
kubectl get pods -n kube-system
```

- Control Plane Services 체크

```bash
service kube-apiserver status

service kube-controller-manager status

service kube-scheduler status
```

- Control Plane Services 체크

```bash
service kubelet status

service kube-proxy status
```

- Service Logs 체크

```bash
kubectl logs kube-apiserver-master -n kube-system

sudo journalctl -u kube-apiserver
```

# 3. Worker Node Failure

- Node Status 체크

```bash
kubectl get nodes

kubectl describe node worker-1, ..
```

- Node 체크

```bash
df -h

top
```

- kubelet Status 확인

```bash
service kubelet status

sudo journalctl -u kubelet
```

- Certificates 확인

```bash
openssl x509 -in /var/lib/kubelet/worker-1.crt -text
```

# 4. Networking Failure

- 외부가 아닌 **클러스터 내부에서 서비스의 클러스터 IP에 연결하고 있는지 확인하십시오.**서비스에 액세스 할 수 있는지 확인 하기 위해 서비스 IP를 핑하지 마십시오 (서비스의 클러스터 IP는 가상 IP이며 ping은 작동하지 않습니다).

- 준비 상태 프로브를 정의한 경우 성공하는지 확인하십시오. 그렇지 않으면 포드가 서비스의 일부가되지 않습니다.
- 포드가 서비스의 일부인지 확인하려면 **kubectl get endpoints**를 사용하여 해당 Endpoints 객체를 검사합니다.
- FQDN 또는 그 일부 (예 : myservice.mynamespace.svc.cluster.local 또는 myservice.mynamespace)를 통해 서비스에 액세스하려고 하는데 작동하지 않는 경우 **해당 서비스를 사용하여 액세스 할 수 있는지 확인하십시오.** FQDN 대신 클러스터 IP.

- 대상 포트가 아닌 서비스에 의해 **노출 된 포트에 연결하고 있는지 확인하십시오.**
- 포드 IP에 직접 연결하여 포드가 올바른 포트에서 연결을 수락하는지 확인하십시오.
- 포드의 IP를 통해 앱에 액세스 할 수없는 경우 앱이 localhost에만 바인딩되지 않는지 확인하세요.
