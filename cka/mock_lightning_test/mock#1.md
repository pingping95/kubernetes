# Mock Exam #1

- Deploy a pod named nginx-pod using the nginx:alpine image.

    docs 보면 쉽게 품

    kubectl create -f nginx.yml

    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-pod
    spec:
      containers:
        - name: nginx-pods
          image: nginx:alpine
    ```

- Deploy a messaging pod using the redis:alpine image with the labels set to tier=msg.

    < docs 보면 쉽게 풀음 >

    ```bash
    master $ cat redis.yml
    apiVersion: v1
    kind: Pod
    metadata:
      name: messaging
      labels:
        tier: msg
    spec:
      containers:
      - name: messagings
        image: redis:alpine
    ```

- Create a namespace named apx-x9984574

    kubectl create namespace apx-x9984574

- Get the list of nodes in JSON format and store it in a file at /opt/outputs/nodes-z3444kd9.json

    kubectl get nodes -o json > ~~

- Create a service messaging-service to expose the messaging application within the cluster on port 6379.

    ```bash
    kubectl expose pod messaging --port=6379 --name messaging-service
    ```

- Create a deployment named hr-web-app using the image kodekloud/webapp-color with 2 replicas

    ```bash
    kubectl create deployment hr-web-app --image=kodekloud/webapp-color

    kubectl scale deployment/hr-web-app --replicas=2
    ```

- Create a static pod named static-busybox on the master node that uses the busybox image and the command sleep 1000

    ```bash
    kubectl run --restart=Never --image=busybox \
    static-busybox --dry-run -o yaml --command -- sleep 1000 > \
    /etc/kubernetes/manifests/static-busybox.yaml
    ```

- Create a POD in the finance namespace named temp-bus with the image redis:alpine.

    ```bash
    kubectl run temp-bus --image=redis:alpine \
    --namespace=finance --restart=Never
    ```

- A new application orange is deployed. There is something wrong with it. Identify and fix the issue.

    ```bash
    kubectl get pods
    kubectl get nodes
    kubectl describe pods orange
    -> initContainer 체크

    kubectl get pod orange -o yaml > orange.yml

    삭제 -> 파일 수정 -> 재 기동
    ```

- Expose the hr-web-app as service hr-web-app-service application on port 30082 on the nodes on the cluster.

    the web app listens on port 8080

    ```bash
    kubectl expose deployment hr-web-app \
    --type=NodePort --port=8080 --target-port 8080 --name=hr-web-app-service \
    --dry-run=client -o yaml > hr-web-app-service.yaml

    vi hr-web-app-service.yaml로 들어가서
    nodePort: 30082 추가

    후 kubectl apply -f ~
    ```

- Use JSON PATH query to retrieve the osImages of all the nodes and store it in a file /opt/outputs/nodes_os_x43kj56.txt

    ```bash
    jsnopath에 대해서 알아놓을 필요가 있다.
    kubernetes cheet sheets를 한번 주욱 보자.

    kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.osImage}' > \
    /opt/outputs/nodes_os_x43kj56.txt
    ```

- Create a Persistent Volume with the given specification.
    - Volume Name: pv-analytics
    - Storage: 100Mi
    - Access modes: ReadWriteMany
    - Host Path: /pv/data-analytics

    ```bash
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-analytics
    spec:
      capacity:
        storage: 100Mi
      accessModes:
        - ReadWriteMany
    	hostPath:
    	    path: /pv/data-ananystics
    ```

    kubectl apply -f ~~.yml
