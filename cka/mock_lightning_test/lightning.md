# Lightning Lab

- 1.

    kubeadm upgrade 보고 하면 됨

    ```bash
    Master Node:
    kubectl drain node master --ignore-daemonsets
    apt-get install kubeadm=1.18.0-00
    kubeadm  upgrade plan v1.18.0
    kubeadm  upgrade apply v1.18.0
    apt-get install kubelet=1.18.0-00
    kubectl uncordon master
    kubectl drain node01 --ignore-daemonsets

    Node01:
    apt-get install kubeadm=1.18.0-00
    kubeadm upgrade node --kubelet-version v1.17.0
    apt-get install kubeket=1.18.0-00

    Back on Master:
    kubectl uncordon node01
    kubectl get pods -o wide | grep gold (make sure this is scheduled on master node)
    ```

- 2

    치트 시트보고 따라하면 됨

    ```bash
    kubectl -n admin2406 get deployment -o custom-columns=DEPLOYMENT:.metadata.name,CONTAINER_IMAGE:.spec.template.spec.containers[].image,READY_REPLICAS:.status.readyReplicas,NAMESPACE:.metadata.namespace --sort-by=.metadata.name > /opt/admin2406_data
    ```

- 3.

    kubectl cluster-info —kubeconfig /root/admin.kubeconfig !!

    ```bash
    Make sure the port for the kube-apiserver is correct.

    Change port from 2379 to 6443 using below command

    vi /root/kubeconfig

    Now replace the port 2379 with 6443

    Run:

    kubectl cluster-info --kubeconfig /root/admin.kubeconfig
    ```

- 4.

    ```
    kubectl create deployment nginx-deploy --image=nginx:1.16
    kubectl set image deployment/nginx-deploy nginx=nginx:1.17 --record
    ```

- 5.

    ```bash
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      annotations:
      name: mysql-alpha-pvc
      namespace: alpha
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: slow
    ```

- 6.

    etcdctl snapshot save --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379 /opt/etcd-backup.db

- 7.

    만약 시크릿 없는 상태로 문제가 나오면, 시크릿 간단하게 조건에 맞게 생성해준 이후 아래와 유사하게 Pod에서 Secret을 참조한 이후 사용

    ```bash
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: secret-1401
      name: secret-1401
      namespace: admin1401
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: dotfile-secret
      containers:
      - command:
        - sleep
        - "4800"
        image: busybox
        name: secret-admin
        volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"
    ```
