# Mock Exam #2

- Take a backup of the etcd cluster and save it to /tmp/etcd-backup.db

    ETCDCTL_API=3 etcdctl version

    cd /etc/kubernetes/manifests

    cat etcd.yaml → copy related things `TCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshotdb`

    ETCDCTL_API=3 etcdctl --endpoints https://[127.0.0.1]:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key member list 로 확인

    ETCDCTL_API=3 etcdctl --endpoints https://[127.0.0.1]:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /tmp/etcd-backup.db 로 적용

    ETCDCTL_API=3 etcdctl --endpoints https://[127.0.0.1]:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot status /tmp/etcd-backup.db -w table 로 최종 확인

- Create a Pod Called elephant with image: redis with CPU Request set to 1 CPU and Memory Request as 200MiB

    kubectl run —generator=run-pod/v1 elephant —image=redis —dry-run -o yaml > elephant.yaml

    add request section 컨테이너 섹션 내부에

    cpu: "1"

    memory: "200MiB"

    kubectl create -f elephant.yaml

    kubectl describe pod elephant

    후에 cpu와 memory limitation을 확인할 것

- Create a Pod called redis-storage with image: redis:alpine with a Volume of type emptyDir that lasts for the life of the Pod. Specs on the right.

- Create a new pod called super-user-pod with image busybox:1.28. Allow the pod to be able to set system_time

    ```yaml
    master $ cat super-user-pod.ymlapiVersion: v1kind: Pod
    metadata:
      name: super-user-pod
    spec:
      containers:
      - image: busybox:1.28
        name: super-user-pod
        command: ['sh', '-c', 'sleep 4800']
        securityContext:
          capabilities:
            add: ["SYS_TIME"]
    ```

- A pod definition file is created at /root/use-pv.yaml. Make use of this manifest file and mount the persistent volume called pv-1. Ensure the pod is running and the PV is bound.

    kubectl get pv

    pvc 생성 → docs에서 persistentVolumeClaims 참고

    vi pvc.yaml

    apiVersion: v1

    kind: PersistentVolumeClaim

    metadata: 

      name: my-pvc

    spec:

      accessModes:

        - ReadWriteOnce

      volumeMode:

      resources:

    등등..

    pvc ——— pod 바인딩해주기

- Create a new deployment called nginx-deploy, with image nginx:1.16 and 1 replica. Record the version. Next upgrade the deployment to version 1.17 using rolling update. Make sure that the version upgrade is recorded in the resource annotation.

    kubectl run nginx-deploy —image=nginx:1.16 —replicas=1 —record

    kubectl rollout history deployment nginx-deploy

    kubectl set image deployment/nginx-deploy nginx-deploy=nginx:1.17 —record

    kubectl describe deployment nginx-deploy | grep -i image

    kubectl rollout history deployment nginx-deploy

- Create a new user called john. Grant him access to the cluster. John should have permission to create, list, get, update and delete pods in the development namespace . The private key exists in the location: /root/john.key and csr at /root/john.csr

    ls -ltr | tail -2

    1. create a certificate signing request

        CSR File

        kubectl api-versions | grep certi

        vi john.yaml

        ```yaml
        apiVersion: certificates.k8s.io/v1beta1
        kind: CertificateSigningRequest
        metadata:
          name: john-developer
        spec:
          request: <<Paste Here>>
          usages:
          - digital signature
          - key encipherment
          - server auth
        ```

        ```yaml
        cat john.csr | base64 | tr -d "\n"

        출력물 저장 master 전까지...

        그리고 request: 에 복붙
        ```

        kubectl create -f john.yaml

        kubectl get csr

        kubectl certificate approve john-developer

        kubectl get csr

        → Approved, Issued라고 뜨면 성공!!

    2. Role Name: developer ~~

    kubectl create role developer —resource=pod —verb=create,list,get,update,delete —namespace=development

    kubectl describe role developer -n development

    3. role binding

    kubectl create rolebinding developer-role-binding —role=developer —user=john —namespace=developer

    kubectl describe rolebinding.rbac.autho~~ developer~~ -n development

    4. Final Check

    kubectl auth can-i update pods —as=john

    kubectl auth can-i update pods —namespace=development —as=john

    yes라고 뜨면 성공한 것임

    kubectl auth can-i delete pods ~~ 

- Create an nginx pod called nginx-resolver using image nginx, expose it internally with a service called nginx-resolver-service. Test that you are able to look up the service and pod names from within the cluster. Use the image: busybox:1.28 for dns lookup. Record results in /root/nginx.svc and /root/nginx.pod

    1번 문제

    kubectl run —generator=run-pod/v1 nginx-resolver —image=nginx

    kubectl expose pod —name=nginx-resolver nginx-resolver-service —port=80 —target-port=80 —type=ClusterIP

    kubectl describe svc nginx-resolver-service

    10.111.54.11

    10.32.0.5:80

    kubectl get pod nginx-resolver -o wide

    → pod의 ip 확인 가능

    kubectl get svc

    2번 문제

    kubectl run —generator=run-pod/v1 test-nslookup —image=busybox:1.28 —rm -it — nslookup nginx-resolver-service

    → 출력물이 정확히 나와야 한다.

    kubectl run —generator-run-pod/v1 test-nslookup —image=busybox:1.28 —rm -it — nslookup nginx-resolver-service > /root/nginx.svc

    /root/ngnix.svc 파일에 저장해주자.

    kubectl run —generator-run-pod/v1 test-nslookup —image=busybox:1.28 —rm -it — nslookup 10-32-0-5.default.pod > /root/nginx.pod

    → 10-32-0-5 하이푼으로 변경해주고 default 는 namespace를 의미하며 그에 맞는 pod에 대해 nslookup을 해준다.

- Create a static pod on node01 called nginx-critical with image nginx. Create this pod on node01 and make sure that it is recreated/restarted automatically in case of a failure.

    kubectl get nodes

    ssh node01

    systemctl status kubelet

    → —config가 어딘지 확인한 후 이동하자.

    cd /var/lib/kubelet

    ls -ltr

    vi confit.yaml

    staticPodPaths 가 있는지 없는지 체크하자. 없을 시, 생성해주고 있으면 경로가 어딘지만 파악해주면 된다.

    cd /etc/kubernetes

    ls

    mkdir manifests

    logout 해서 master_node로 돌아오기

    kubectl run —generator=run-pod/v1 nginx-critical —image=nginx —dry-run -o yaml > nginx-critical.yaml

    cat nginx-critical.yaml

    컨텐츠 복사한 후 

    ssh node01

    cd /etc/kubernetes/manifest

    vi nginx-critical.yaml

    그 이후, contents를 paste하기

    (dnsPolicy 삭제, restartPolicy 삭제)

    docker ps | grep -i nginx-critical로 체크

    master로 돌아오기

    logout

    kubectl get pods

    → nginx-critical-node01이 생성된 것을 확인할 수 있다.
