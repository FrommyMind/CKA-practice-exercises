# Storage (10%)

## Understand storage classes, persistent volumes

Doc: https://kubernetes.io/docs/concepts/storage/storage-classes/
Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

## Understand volume mode, access modes and reclaim policies for volumes

Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes
Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaiming
Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#volume-mode

## Understand persistent volume claims primitive

Doc: https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/

## Know how to configure applications with persistent storage

Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Questions:
- Create a pod and mount a volume with hostPath directory.
- Check that the contents of the directory are accessible through the pod.

<details><summary>Solution</summary>
<p>

pv-pod.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pv-pod
  name: pv-pod
spec:
  containers:
  - image: busybox:latest
    name: pv-pod
    args:
      - sleep
      - "3600"
    volumeMounts:
    - name: data
      mountPath: "/data"
  volumes:
  - name: data
    hostPath:
      path: "/home/ubuntu/data/"
```

```bash
# Create directory and file inside it on worker nodes
mkdir /home/ubuntu/data
touch data/file

kubectl apply -f pv-pod.yaml
kubectl exec pv-pod -- ls /data
file
```

</p>
</details>

Questions:
- Create a persistent volume from hostPath and a persistent volume claim corresponding tothat PV. Create a pod that uses the PVC and check that the volume is mounted in the pod.
- Create a file from the pod in the volume then delete it and create a new pod with the same volume and show the created file by the first pod.

<details><summary>Solution</summary>
<p>

pv-data.yaml:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  storageClassName: "local"
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/ubuntu/data"

```

pvc-data.yaml:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  storageClassName: "local"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

pvc-pod.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pvc-pod
  name: pvc-pod
spec:
  containers:
  - image: busybox:latest
    name: pvc-pod
    args:
      - sleep
      - "3600"
    volumeMounts:
    - name: data
      mountPath: "/data"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-data

```

Create a pod with the PVC. Create a file on volume. Delete the pod and create a new one with the same volume. Check that the file has persisted.

```bash
kubectl apply -f pv-data.yaml
kubectl apply -f pvc-data.yaml
kubectl apply -f pvc-pod.yaml

kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
pv-data   1Gi        RWO            Retain           Bound    default/pvc-data   local                   20m

kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-data   Bound    pv-data   1Gi        RWO            local          20m

# Check that the volume has been mounted
kubectl exec pvc-pod -- ls /data/
file

# Create a new file
kubectl exec pvc-pod -- touch /data/file2

# Delete the pod
kubectl delete -f pvc-pod.yaml

# Copy the pvc-pod.yaml and change the name of the pod to pvc-pod-2
kubectl apply -f pvc-pod-2.yaml

# Check that the file from previous pod has persisted on volume
kubectl exec pvc-pod-2 -- ls /data/
file
file2
```

</p>
</details>

Questions:
- Create a pod name pod-log, container name log-producer use image busybox, output the important information at /log/data/output.log. Then another container name log-consumer can use image busybox, load the output.log at /log/data/output.log. Note, this log file only can be share within the pod.

<details><summary>Solution</summary>
<p>
1. create a yaml file with below command

```bash
$ kubectl run pod-log --image=busybox --dry-run=client -o yaml > pod-log.yaml
```
then we will get a file like this.
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-log
  name: pod-log
spec:
  containers:
  - image: busybox
    name: pod-log
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
2. modify the file

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-log
  name: pod-log
spec:
  containers:
  - image: busybox
    name: pod-producer
    command: ["sh", "-c", "echo important information >> /log/data/output.log; sleep 3600"]
    volumeMounts:
    - name: data-log
      mountPath: /log/data
    resources: {}
  - image: busybox
    name: pod-consumer
    command: ["sh", "-c", "cat /log/data/output.log; sleep 3600"]
    volumeMounts:
    - name: data-log
      mountPath: /log/data
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: data-log
    emptyDir: {}
status: {}
```
3. apply this yaml 

```bash
$ kubectl apply -f pod-log.yaml
```

4. check the output

```bash
## check the volume
kubectl describe pod pod-log

```

```yaml
Name:         pod-log
Namespace:    default
Priority:     0
Node:         k8s-2/10.191.73.224
Start Time:   Thu, 06 Jan 2022 15:47:06 +0800
Labels:       run=pod-log
Annotations:  <none>
Status:       Running
IP:           10.244.1.54
IPs:
  IP:  10.244.1.54
Containers:
  pod-producer:
    Container ID:  docker://9f9c67ee46034f1301fc3d2346c775091600a5ea857b2dc0a03f87a37ece8a46
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      echo important information >> /log/data/output.log; sleep 3600
    State:          Running
      Started:      Thu, 06 Jan 2022 15:47:11 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /log/data from data-log (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rxcs8 (ro)
  pod-consumer:
    Container ID:  docker://c74d7018a10f4ed6f987a6dff95efbd174f1d0f20231b69650b15f63160aaa57
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      cat /log/data/output.log; sleep 3600
    State:          Running
      Started:      Thu, 06 Jan 2022 15:47:14 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /log/data from data-log (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rxcs8 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  data-log:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  kube-api-access-rxcs8:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m18s  default-scheduler  Successfully assigned default/pod-log to k8s-2
  Normal  Pulling    2m25s  kubelet            Pulling image "busybox"
  Normal  Pulled     2m23s  kubelet            Successfully pulled image "busybox" in 2.771629026s
  Normal  Created    2m23s  kubelet            Created container pod-producer
  Normal  Started    2m22s  kubelet            Started container pod-producer
  Normal  Pulling    2m22s  kubelet            Pulling image "busybox"
  Normal  Pulled     2m20s  kubelet            Successfully pulled image "busybox" in 2.651887275s
  Normal  Created    2m20s  kubelet            Created container pod-consumer
  Normal  Started    2m19s  kubelet            Started container pod-consumer

```
get the container log
```bash
$ kubectl logs  pod-log pod-consumer
important information
```

</p>
</details>