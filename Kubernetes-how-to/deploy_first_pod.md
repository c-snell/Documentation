## Deploying our first Pod


A pod is a collection of containers sharing a network and mount namespace and is the basic unit of deployment in Kubernetes. All containers in a pod are scheduled on the same node.

To launch a pod using the container image mhausenblas/simpleservice:0.5.0 and exposing a HTTP API on port 9876, execute:

kubectl run sise --image=mhausenblas/simpleservice:0.5.0 --port=9876

Lets create a simple nginx webserver.

```
notepad first-nginx-pod.yml
```

Copy and paste the following
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: first-nginx-pod
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-first-pod
  template:
    metadata:
      labels:
        run: nginx-first-pod
    spec:
      containers:
      - image: nginx
        name: nginx
```        
Save and exit

Create the pod, execute:
```
kubectl apply -f first-nginx-pod.yml
```

We can now see the pod running:
```
kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
first-nginx-pod-5d77bbb868-k48zk   1/1     Running   0          6m39s
```

We can inspect the pod further using the kubectl describe command:
```
kubectl describe pod first-nginx-pod-5d77bbb868-k48zk
NAME                               READY   STATUS    RESTARTS   AGE
first-nginx-pod-5d77bbb868-k48zk   1/1     Running   0          6m39s
PS C:\Users\Administrator> kubectl describe pod first-nginx-pod-5d77bbb868-k48zk
Name:           first-nginx-pod-5d77bbb868-k48zk
Namespace:      default
Priority:       0
Node:           kube-g7-node2/10.90.200.75
Start Time:     Mon, 14 Oct 2019 13:35:18 -0500
Labels:         pod-template-hash=5d77bbb868
                run=nginx-first-pod
Annotations:    <none>
Status:         Running
IP:             10.233.126.2
IPs:            <none>
Controlled By:  ReplicaSet/first-nginx-pod-5d77bbb868
Containers:
  nginx:
    Container ID:   docker://158bdc3f5ba180316b1836a4f75fe37941224403069091709cf8c07cbe01bfd5
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:aeded0f2a861747f43a01cf1018cf9efe2bdd02afd57d2b11fcc7fcadc16ccd1
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 14 Oct 2019 13:35:34 -0500
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-5b8gb (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-5b8gb:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-5b8gb
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From                    Message
  ----    ------     ----   ----                    -------
  Normal  Scheduled  7m18s  default-scheduler       Successfully assigned default/first-nginx-pod-5d77bbb868-k48zk to kube-g7-node2
  Normal  Pulling    7m17s  kubelet, kube-g7-node2  Pulling image "nginx"
  Normal  Pulled     7m3s   kubelet, kube-g7-node2  Successfully pulled image "nginx"
  Normal  Created    7m3s   kubelet, kube-g7-node2  Created container nginx
  Normal  Started    7m3s   kubelet, kube-g7-node2  Started container nginx
```
