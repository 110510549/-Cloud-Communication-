# 雲端通訊整合實務(12/8)
###### tags: `docker`


## kubernetes
### Basic Command


創建一個 deployment
```
kubectl create deployment myweb --image=httpd
```

部署 deployment，並且可以連到外網
```
kubectl expose deployment myweb --type="NodePort" --port=80
```
> type: NodePort(連到外網的)
> type: ClusterPort(集群使用)

查看 k8s 部署

```
kubectl get deployment
```

```
root@vm1:~# kubectl get deployment
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web1   1/1     1            1           4d6h
web2   1/1     1            1           46h
```

查看 k8s 服務

```
kubectl get svc
```

```
root@vm1:~# kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP        4d6h
mysql-1609235295   ClusterIP   10.110.216.211   <none>        3306/TCP       4d6h
web1               NodePort    10.97.190.191    <none>        80:30280/TCP   4d5h
```

更改副本數可以使用`scale`
```
kubectl scale deployment myweb --replicas 3
```

```
root@vm1:~# kubectl scale deployment web1 --replicas 3
deployment.apps/web1 scaled
root@vm1:~# kubectl get deployment
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web1   3/3     3            3           4d6h
web2   1/1     1            1           46h
```
> 當妳生成一個部署時，內定只會產生一個pod，因此你可以使用`scale`進行更改變數


### Rolling Update

`k8s`和`docker swarm`一樣也支援了 rolling update 和 rolling back 等功能

因此我們先創建一個 httpd 的部署，版本選擇較舊的`2.4.43`版本

```
root@vm1:~# kubectl create deployment myweb1 --image=httpd:2.4.43
deployment.apps/myweb1 created
root@vm1:~# kubectl get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myweb1   0/1     1            0           7s
web1     3/3     3            3           4d6h
web2     1/1     1            1           46h
root@vm1:~# kubectl get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myweb1   1/1     1            1           9s
web1     3/3     3            3           4d6h
web2     1/1     1            1           46h
```

創建完之後可以檢查`pod`狀況
```
root@vm1:~# kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
myweb1-65b49bc59d-c94j7   1/1     Running   0          3m1s
web1-7469c97b99-74h9p     1/1     Running   0          24h
web1-7469c97b99-ss4js     1/1     Running   0          11m
web1-7469c97b99-z5xdw     1/1     Running   0          11m
web2-58b88d6994-d58bt     1/1     Running   0          46h
```

若你想確認版本是否是指定的版本，可以使用`describe`這項指令查看
```
root@vm1:~# kubectl describe pod myweb1-65b49bc59d-c94j7
Name:         myweb1-65b49bc59d-c94j7
Namespace:    default
Priority:     0
Node:         vm3/192.168.102.145
Start Time:   Sat, 02 Jan 2021 08:14:39 -0800
Labels:       app=myweb1
              pod-template-hash=65b49bc59d
Annotations:  <none>
Status:       Running
IP:           10.244.2.7
IPs:
  IP:           10.244.2.7
Controlled By:  ReplicaSet/myweb1-65b49bc59d
Containers:
  httpd:
    Container ID:   docker://cf56d4349f4cc9dfb77a60ef32d71cad37c046db142683ba300118b8f1ec3918
    Image:          httpd:2.4.43
    Image ID:       docker-pullable://httpd@sha256:cd88fee4eab37f0d8cd04b06ef97285ca981c27b4d685f0321e65c5d4fd49357
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 02 Jan 2021 08:14:47 -0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-f9h8d (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-f9h8d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-f9h8d
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m15s  default-scheduler  Successfully assigned default/myweb1-65b49bc59d-c94j7 to vm3
  Normal  Pulling    3m15s  kubelet            Pulling image "httpd:2.4.43"
  Normal  Pulled     3m9s   kubelet            Successfully pulled image "httpd:2.4.43" in 6.164628757s
  Normal  Created    3m9s   kubelet            Created container httpd
  Normal  Started    3m8s   kubelet            Started container httpd
```

> Image : httpd:2.4.43

在開始滾動更新之前，有件事你必須先知道！

- kubectl set image deployment <deployment> **container**=image

---

若你不曉得容器名稱是什麼，可以透過`-o yaml`的方式去進行查找
```
root@vm1:~# kubectl get deployment myweb1 -o yaml | grep name
              k:{"name":"httpd"}:
                f:name: {}
  name: myweb1
  namespace: default
  selfLink: /apis/apps/v1/namespaces/default/deployments/myweb1
        name: httpd
```
> name : httpd (container name)

進行滾動更新至`2.4.46`版本

```
kubectl set image deployment myweb1 httpd=httpd:2.4.46
```

```
root@vm1:~# kubectl set image deployment myweb1 httpd=httpd:2.4.46
deployment.apps/myweb1 image updated
```

更新完之後，可以透過`pod`去查看名稱

```
root@vm1:~# kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
myweb1-5c58cfb79c-nk498   1/1     Running   0          5m41s
web1-7469c97b99-74h9p     1/1     Running   0          24h
web1-7469c97b99-ss4js     1/1     Running   0          42m
web1-7469c97b99-z5xdw     1/1     Running   0          42m
web2-58b88d6994-d58bt     1/1     Running   0          47h
```

並且透過`describe`的方式去觀察版本是否更新至 2.4.46 版本
```
root@vm1:~# kubectl describe pods myweb1-5c58cfb79c-nk498
Name:         myweb1-5c58cfb79c-nk498
Namespace:    default
Priority:     0
Node:         vm2/192.168.102.139
Start Time:   Sat, 02 Jan 2021 08:43:12 -0800
Labels:       app=myweb1
              pod-template-hash=5c58cfb79c
Annotations:  <none>
Status:       Running
IP:           10.244.1.7
IPs:
  IP:           10.244.1.7
Controlled By:  ReplicaSet/myweb1-5c58cfb79c
Containers:
  httpd:
    Container ID:   docker://cffde66e6e1ebf40f2e1ee3d3b3e09bbac0ffd82a0d9706ef17045d007ce0f0a
    Image:          httpd:2.4.46
    Image ID:       docker-pullable://httpd@sha256:a3a2886ec250194804974932eaf4a4ba2b77c4e7d551ddb63b01068bf70f4120
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 02 Jan 2021 08:43:13 -0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-f9h8d (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-f9h8d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-f9h8d
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  6m1s  default-scheduler  Successfully assigned default/myweb1-5c58cfb79c-nk498 to vm2
  Normal  Pulled     6m1s  kubelet            Container image "httpd:2.4.46" already present on machine
  Normal  Created    6m1s  kubelet            Created container httpd
  Normal  Started    6m    kubelet            Started container httpd

```

回滾功能

```
root@vm1:~# kubectl rollout undo deployment myweb1
deployment.apps/myweb1 rolled back
```

回滾完之後，一樣透過以下方式進行查找

```
root@vm1:~# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
myweb1-65b49bc59d-8g995   1/1     Running   0          2m25s
web1-7469c97b99-74h9p     1/1     Running   0          24h
web1-7469c97b99-ss4js     1/1     Running   0          49m
web1-7469c97b99-z5xdw     1/1     Running   0          49m
web2-58b88d6994-d58bt     1/1     Running   0          47h
root@vm1:~# kubectl describe pods myweb1-65b49bc59d-8g995
Name:         myweb1-65b49bc59d-8g995
Namespace:    default
Priority:     0
Node:         vm3/192.168.102.145
Start Time:   Sat, 02 Jan 2021 08:53:08 -0800
Labels:       app=myweb1
              pod-template-hash=65b49bc59d
Annotations:  <none>
Status:       Running
IP:           10.244.2.8
IPs:
  IP:           10.244.2.8
Controlled By:  ReplicaSet/myweb1-65b49bc59d
Containers:
  httpd:
    Container ID:   docker://f61cb900ccc2cfd5e1aa446b835dcb361c5d8551d72195fbf75be0be73761c3e
    Image:          httpd:2.4.43
    Image ID:       docker-pullable://httpd@sha256:cd88fee4eab37f0d8cd04b06ef97285ca981c27b4d685f0321e65c5d4fd49357
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 02 Jan 2021 08:53:09 -0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-f9h8d (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-f9h8d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-f9h8d
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m43s  default-scheduler  Successfully assigned default/myweb1-65b49bc59d-8g995 to vm3
  Normal  Pulled     2m42s  kubelet            Container image "httpd:2.4.43" already present on machine
  Normal  Created    2m42s  kubelet            Created container httpd
  Normal  Started    2m42s  kubelet            Started container httpd
```

此時，你的 httpd 版本又回復到先前的 2.4.43 版本了

---
### Kubernetes 補充介紹

查看 namespace
```
kubectl get ns
```
```
kubectl get namespace
```

```
root@vm1:~# kubectl get ns
NAME                   STATUS   AGE
default                Active   4d8h
kube-node-lease        Active   4d8h
kube-public            Active   4d8h
kube-system            Active   4d8h
kubernetes-dashboard   Active   4d8h
root@vm1:~# kubectl get namespace
NAME                   STATUS   AGE
default                Active   4d8h
kube-node-lease        Active   4d8h
kube-public            Active   4d8h
kube-system            Active   4d8h
kubernetes-dashboard   Active   4d8h
```

其中的`kube-system`又稱命名空間，裡面存放了 k8s 所有的`pod`

```
kubectl get pods -n kube-system
```

```
root@vm1:~# kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
etcd-vm1                      1/1     Running   0          4d8h
kube-apiserver-vm1            1/1     Running   8          4d8h
kube-controller-manager-vm1   1/1     Running   7          4d8h
kube-flannel-ds-fgxj7         1/1     Running   0          4d8h
kube-flannel-ds-jxst8         1/1     Running   4          4d8h
kube-flannel-ds-rk25b         1/1     Running   0          4d8h
kube-proxy-98rzn              1/1     Running   0          4d8h
kube-proxy-vpqct              1/1     Running   4          4d8h
kube-proxy-wk2mm              1/1     Running   0          4d8h
kube-scheduler-vm1            1/1     Running   7          4d8h
```

> -n : namespace

創建自己的命名空間
```
kubectl create ns myns
```

```
root@vm1:~# kubectl create ns myns
namespace/myns created
root@vm1:~# kubectl get ns
NAME                   STATUS   AGE
default                Active   4d8h
kube-node-lease        Active   4d8h
kube-public            Active   4d8h
kube-system            Active   4d8h
kubernetes-dashboard   Active   4d8h
myns                   Active   5s
```


創建一個 httpd 部署，並且指定存放命名空間在 myns 內
```
kubectl create deployment myweb2 --image=httpd -n myns
```

這時你就可以測試，分別測試`kubectl get deployment`以及`kubectl get deployment -n myns`
```
root@vm1:~# kubectl create deployment myweb2 --image=httpd -n=myns
deployment.apps/myweb2 created
root@vm1:~# kubectl get deployments
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myweb1   1/1     1            1           64m
web1     3/3     3            3           4d7h
web2     1/1     1            1           47h
root@vm1:~# kubectl get deployments -n myns
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myweb2   1/1     1            1           17s
```

此時你會發現當你使用`kubectl get deployment`時會沒有 myweb2，因為你將命名空間指定到 myns 的空間裡，因此`kubectl get deployment`只會出現預設的 (default) 項目

> `kubectl get deployment = kubectl get deployment -n default`

同理 pod 也是一樣情形！

```
root@vm1:~# kubectl get pods -n default
NAME                      READY   STATUS    RESTARTS   AGE
myweb1-65b49bc59d-8g995   1/1     Running   0          32m
web1-7469c97b99-74h9p     1/1     Running   0          25h
web1-7469c97b99-ss4js     1/1     Running   0          79m
web1-7469c97b99-z5xdw     1/1     Running   0          79m
web2-58b88d6994-d58bt     1/1     Running   0          47h
root@vm1:~# kubectl get pods -n myns
NAME                     READY   STATUS    RESTARTS   AGE
myweb2-7958b5fc8-pm2nc   1/1     Running   0          7m1s
```

查看全部的命名空間，可以使用`--all-namespace`

```
kubectl get pods --all-namespace
kubectl get deployment --all-namespace
```

---

進入 pods 的任意容器內

```
root@vm1:~# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
myweb1-65b49bc59d-8g995   1/1     Running   0          39m
web1-7469c97b99-74h9p     1/1     Running   0          25h
web1-7469c97b99-ss4js     1/1     Running   0          86m
web1-7469c97b99-z5xdw     1/1     Running   0          86m
web2-58b88d6994-d58bt     1/1     Running   0          47h
root@vm1:~# kubectl exec -it myweb1-65b49bc59d-8g995 -- bash
root@myweb1-65b49bc59d-8g995:/usr/local/apache2# 
```

> 這邊要注意的事，其中的`-- bash`與前面的`kubectl exec -it myweb1-65b49bc59d-8g995` 完全沒關係，`--`的功能在於讓 k8s 去處理後面要做的指令，例如`bash`。

例如顯示日期等功能，或是顯示"hi"等方式

```
root@vm1:~# kubectl exec -it myweb1-65b49bc59d-8g995 -- date
Sat Jan  2 17:47:11 UTC 2021
```

```
root@vm1:~# kubectl exec -it myweb1-65b49bc59d-8g995 -- echo "hi"
hi
```

**會看到使用`exec`並非都會像搭配`bash`一樣進入容器內，而是取決在使用者要如何去使用 `--` 後面的指令了！**

