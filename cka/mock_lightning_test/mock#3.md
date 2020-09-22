# Mock Exam #3

- Create a new service account with the name pvviewer. Grant this Service account access to list all PV in the cluster by creating an appropriate cluster role called pvviewer-role and ClusterRoleBinding called pvviewer-role-Binding. Next, create a pod called pvviewer with the image: redis and serviceaccount: pvviewer in the default namespace

    kubectl create serviceaccount pvviewer

    kubectl create clusterrole pvviewer-role —resource=persistentvolume —verb=list

    kubectl create clusterrolebinding pvviewer-role-binding —clusterrole=pvviewer-role —serviceaccount=default:pvviewer

    kubectl run —generator=run-pod/v1 pvviewer —image=redis —dry-run -o yaml > pod.yaml

    vi pod.yaml

    구글 docs에 들어가서 serviceAccountName: pvviewer 추가해서 넣을 수 있도록 참고하기

    kubectl create -f pod.yaml 해서 파드 생성

    kubectl describe pod pvviewer → 컨테이너 정보 확인하기...

- List the InternalIP of all nodes of the cluster. Save the result to a file /root/node_ips

- Create a pod called multi-pod with two containers. Container 1, name: alpha, image: nginx Container 2: beta, image: busybox, command sleep 4800. + env variable

    kubectl run —generator=run-pod/v1 alpha —image=nginx —dry-run=client -o yaml > multi-pod.yaml

    vi multi-pod.yaml

    spec:

    containers:

    _ image: nginx

    name: alpha

    env:

    _ name: name

    value: alpha

    _ image: busybox

    name: beta

    command: ["sleep", "4800"]

    env:

    _ name: name

    value: beta

    kubectl create -f multi-pod.yaml

    kubectl describe pod multi-pod 

- Create a Pod Called lion with image redis:alpine. Set the CPU Limit to 2 CPU and Memory Limit to 500MiB

    spec.containers.resources

    ++ limits:

       cpu:

       memory:

    kubectl create -f lion.yaml

    kubectl describe lion

- We have deployed a new pod called np-test-1 and a service called np-test-service. Incoming connections to this service are not working. Troubleshoot and fix it.
Create NetworkPolicy, by the name ingress-to-nptest that allows incoming connections to the service over port 80

    kubectl get pods

    kubectl describe pod np-test-1

    kubectl describe svc np-test-service

    kubectl run —generator=run-pod/v1 test-np —image=busybox:1.28 —rm -it — sh

    nc -z -v -w 2 np-test-service:80

    → Connection timed out → Ingress not working...

    kubectl get netpol

    kubectl describe netpol default-deny

    docs : network policy resource 검색

    podSelector

    ingress → all ports open, all ip open

    kubectl get pod np-test-1 —show-labels 

    라벨 확인

    vi np.yaml

    ```bash
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: ingress-to-nptest
      namespace: default
    spec:
    	podSelector:
    		matchLabels:
    			**run:np-test-1**
    	policyTypes:
    	- Ingress
    	ingess:
    	- ports:
    		- port: 80
    			protocol: TCP
    ```

    kubectl create -f np.yaml

    kubectl run —generator=run-pod/v1 test-np —image=busybox:1.28 —rm -it — sh

    nc -z -v -w 2 np-test-service:80

- Taint the worker node node01 to be Unschedulable. Once done, create a pod called dev-redis, image redis:alpine to ensure workloads are not scheduled to this worker node. Finally, create a new pod called prod-redis and image redis:alpine with toleration to be scheduled on node01.

    kubectl get nodes

    kubectl taint node node01 env_type=production:NoSchedule

    kubectl describe node node01 | grep -i taint

    kubectl run —generator=run-por/v1 dev-redis —image=redis:alpine

    kubectl get pods -o wide

    kubectl get pod dev-redis -o yaml > prod-redis.yaml

    vi prod-redis.yaml

    → toleration filed만 남겨놓는다.

    toleration

    _ effect: NoSchedule

    key: env_type

    operator: Equal

    value: production

    kubectl create -f prod-redis.yaml

    kubectl get pods -o wide

- Create a pod called hr-pod in hr namespace belonging to the production environment and frontend tier. image: redis:alpine

    kubectl get ns

    kubectl create ns hr

    kubectl run —generator=run-pod/v1 hr-pod —image=redis:alpine —labels=environment=production,tier=frontend —namespace=hr

    kubectl -n hr get pods —show-labels

- A kubeconfig file called super.kubeconfig has been created in /root. here is something wrong with the configuration. Troubleshoot and fix it.

    kubectl get nodes

    cd .kube/

    ls-ltr

    cat config

    cd /root

    kubectl cluster-info —kubeconfig-/root/super.kubeconfig

    → port를 바꿔야 한다.

    vi super.kubeconfig

    server: [https://172.~~:6443](https://172.~~:6443)

    → API SERVER의 port를 기입해준다.

    kubectl cluster-info —kubeconfig-/root/super.kubeconfig

- We have created a new deployment called nginx-deploy. scale the deployment to 3 replicas. Has the replica's increased? Troubleshoot the issue and fix it.

    kubectl get deployments.

    kubectl scale deployment nginx-deploy —replicas=3

    kubetl get deployments.

    → AVAILABLE이 안뜬다.

    kubectl describe deployments.nginx-deploy

    kubectl get pods

    kubectl -n kube-system get pods

    → controller-manager-master 의 상태가 맛이 갔음..

    cd /etc/kubernetes/manifest/

    ls -ltr

    vi kube-controller-manager.yaml

    sed -i 's/kube-contro1ler-manager/kube-controller-manager/g' kube-controller-manager.yaml

    kubectl -n kube-system get pods

    kubectl get deployments.

    kubectl get pods
