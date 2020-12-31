# 雲端通訊整合實務(12/1)
###### tags: `docker`

## 環境準備

虛擬機：3台
處理器：2核心
記憶體：2G

| IP Address | Hostname | CPU | Ram | OS |
|:---------------:|:---------------:|:---------------:|:---------------:|:---------------:|
| 192.168.102.135 | vm1(master) | 2 | 4 | ubuntu(16.04) |
| 192.168.102.139 | vm2(worker) | 2 | 4 | ubuntu(16.04) |
| 192.168.102.140 | vm3(worker) | 2 | 4 | ubuntu(16.04) |

![](https://i.imgur.com/C5vnbeu.png)


## kubernetes安裝(ubuntu)
**若是centos的用戶可以參照這一篇安裝k8s：**
https://blog.tomy168.com/2019/08/centos-76-kubernetes.html

安裝net-tools工具
```
apt-get update
apt-get install wget net-tools nano -y
```
更改虛擬機名稱
```
hostnamectl set-hostname vm1
hostnamectl set-hostname vm2
hostnamectl set-hostname vm3
```
編輯`/etc/hosts`，在最後一行增加三台虛擬機的IP以及名稱：
```
192.168.102.140 vm3
192.168.102.135 vm1
192.168.102.139 vm2
```

編輯完後，重啟一下網路並且測試用名稱互相ping：
```
systemctl restart NetworkManager
```

```
root@vm1:/home/user# ping vm1
PING vm1 (192.168.102.135) 56(84) bytes of data.
64 bytes from vm1 (192.168.102.135): icmp_seq=1 ttl=64 time=0.063 ms
64 bytes from vm1 (192.168.102.135): icmp_seq=2 ttl=64 time=0.081 ms
^C
--- vm1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.063/0.072/0.081/0.009 ms
root@vm1:/home/user# ping vm2
PING vm2 (192.168.102.139) 56(84) bytes of data.
64 bytes from vm2 (192.168.102.139): icmp_seq=1 ttl=64 time=0.836 ms
64 bytes from vm2 (192.168.102.139): icmp_seq=2 ttl=64 time=0.885 ms
^C
--- vm2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.836/0.860/0.885/0.038 ms
root@vm1:/home/user# ping vm3
PING vm3 (192.168.102.140) 56(84) bytes of data.
64 bytes from vm3 (192.168.102.140): icmp_seq=1 ttl=64 time=0.866 ms
64 bytes from vm3 (192.168.102.140): icmp_seq=2 ttl=64 time=1.10 ms
^C
--- vm3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.866/0.983/1.100/0.117 ms
root@vm1:/home/user# 
```

檢查firewalld以及SELinux是否關閉
```
systemctl stop firewalld
systemctl status firewalld
```

```
getenforce
sed -i 's@SELINUX=enforcing@SELINUX=disabled@' /etc/sysconfig/selinux
```

編輯`/etc/fstab`，並將swap那行註解掉
```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda1 during installation
UUID=14bff259-e1db-4b35-b0a0-b79fb9c847a7 /               ext4    errors=remount-ro 0       1
# swap was on /dev/sda5 during installation
#UUID=a3684131-294c-46ce-b9d0-a0a999665ad1 none            swap    sw              0       0
/dev/fd0        /media/floppy0  auto    rw,user,noauto,exec,utf8 0       0
```

編輯完後重啟電腦`reboot`，重啟後使用`free`檢查swap的值是否都為0。

```
root@vm1:/home/user# free
              total        used        free      shared  buff/cache   available
Mem:        4028472     1445624      256292       59080     2326556     2116432
Swap:             0           0           0
```

將iptables相關功能或模組的啟用與停用

```
echo 1 > /proc/sys/net/ipv4/ip_forward
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
```
透過`sysctl -p`檢查上面指令是否成功輸入
```
root@vm1:/home/user# sysctl -p
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
```
將 br_netfilter模組特別寫入`/etc/modules-load.d`的目錄中，讓重開機後的作業系統能夠自動載入
```
modprobe br_netfilter
echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf
```
利用`lsmod`檢查br_netfilter模組是否成功載入
```
root@vm1:/home/user# lsmod | grep br_netfilter
br_netfilter           24576  0
bridge                122880  1 br_netfilter
```

若未安裝docker的用戶，可以使用以下指令安裝
```
apt-get update && apt-get install -y docker.io
```
安裝完後記得開啟docker，並且設定每次開機後自動開啟此功能
```
systemctl start docker
systemctl enable docker
```

安裝k8s相關套件(kubectl、kubeadm、kubelet)
```
apt-get update && apt-get isntall -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/
apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubectl kubeadm kubelet
```

初始化master
```
docker run -itd -p 8888:8080 -e HOST=192.168.102.135  -e PORT=8080 -v /var/run/docker.sock:/var/run/docker.sock --name visualizer dockersamples/visualizer
```

成功會出現這這樣的畫面：
```
(省略)
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.102.135:6443 --token 2hhif4.iarj1b27mtpmgu1f \
    --discovery-token-ca-cert-hash sha256:0ced03c22bbf329282db8a301ee3f7c0e80720536cc35318ca7f26637a0d0e4a
```
其中需要先將k8s的安裝路徑設定好
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

安裝通用的 flannel 容器網路介面CNI介面
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

接著就可以將token複製下來，並且貼到其他`VM(vm2、vm3)`上來建立worker

若成功後可以到master端進行查看，其結果會長這樣：
```
root@vm1:/etc/systemd/system/kubelet.service.d# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
vm1    Ready    master   2d    v1.19.4
vm2    Ready    <none>   2d    v1.19.4
vm3    Ready    <none>   2d    v1.19.4
```
> kubectl get nodes:查看節點狀況

接下來可以做個測驗來測試k8s是否可執行以及成功建立起網路的部署

```
kubectl create deployment myweb --image=httpd
kubectl expose deployment myweb --type="NodePort" --port=80
```

**創建一個部署，透過httpd的images，名稱叫做myweb**

> kubectl create deployment myweb --image=httpd

**為myweb的httpd部署提供80埠**

> kubectl expose deployment myweb --type="NodePort" --port=80

建立完後可以透過`kubectl get pods`和`kubectl get svc`查看pod和service

```
root@vm1:/etc/systemd/system/kubelet.service.d# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
myweb-6f795ffc84-dccnx   1/1     Running   0          45h
```
```
root@vm1:/etc/systemd/system/kubelet.service.d# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        2d1h
myweb        NodePort    10.110.178.159   <none>        80:31262/TCP   44h
```

> 你可以從svc看到，myweb的埠號為31262端口

因此你就可以進行httpd測試，去測試vm1～vm3的ip是否都能顯示`it works`

![](https://i.imgur.com/4mYfV9P.png)
![](https://i.imgur.com/6WcvKeG.png)
![](https://i.imgur.com/LRNkR02.png)

* 初始化 k8s：`kubeadm reset`

## k8s 基本指令


查詢節點`vm1`詳細資料
```
kubectl describe node [ master ]
```

```
root@vm1:~# kubectl describe node vm1
Name:               vm1
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=vm1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"0e:42:91:4b:3d:a7"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 192.168.102.146
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 29 Dec 2020 00:53:09 -0800
Taints:             node-role.kubernetes.io/master:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  vm1
  AcquireTime:     <unset>
  RenewTime:       Thu, 31 Dec 2020 08:24:44 -0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 29 Dec 2020 00:55:33 -0800   Tue, 29 Dec 2020 00:55:33 -0800   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Thu, 31 Dec 2020 08:20:54 -0800   Tue, 29 Dec 2020 00:53:06 -0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 31 Dec 2020 08:20:54 -0800   Tue, 29 Dec 2020 00:53:06 -0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 31 Dec 2020 08:20:54 -0800   Tue, 29 Dec 2020 00:53:06 -0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 31 Dec 2020 08:20:54 -0800   Tue, 29 Dec 2020 00:53:28 -0800   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  192.168.102.146
  Hostname:    vm1
Capacity:
  cpu:                2
  ephemeral-storage:  101016992Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             4028476Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  93097259674
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3926076Ki
  pods:               110
System Info:
  Machine ID:                 953160eec3fd473e8359d677ec90af53
  System UUID:                D3194D56-5150-0493-73C0-A9ECA9169305
  Boot ID:                    377ad6c1-abfb-4430-86cb-04f045faf5a5
  Kernel Version:             4.4.0-197-generic
  OS Image:                   Ubuntu 16.04.6 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.7
  Kubelet Version:            v1.19.4
  Kube-Proxy Version:         v1.19.4
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
Non-terminated Pods:          (8 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  kube-system                 etcd-vm1                                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         2d7h
  kube-system                 kube-apiserver-vm1                            250m (12%)    0 (0%)      0 (0%)           0 (0%)         2d7h
  kube-system                 kube-controller-manager-vm1                   200m (10%)    0 (0%)      0 (0%)           0 (0%)         2d7h
  kube-system                 kube-flannel-ds-fgxj7                         100m (5%)     100m (5%)   50Mi (1%)        50Mi (1%)      2d7h
  kube-system                 kube-proxy-wk2mm                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         2d7h
  kube-system                 kube-scheduler-vm1                            100m (5%)     0 (0%)      0 (0%)           0 (0%)         2d7h
  kubernetes-dashboard        dashboard-metrics-scraper-7445d59dfd-8ktwd    0 (0%)        0 (0%)      0 (0%)           0 (0%)         2d1h
  kubernetes-dashboard        kubernetes-dashboard-5bc88ff8f9-t8v25         0 (0%)        0 (0%)      0 (0%)           0 (0%)         2d1h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                650m (32%)  100m (5%)
  memory             50Mi (1%)   50Mi (1%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:              <none>
```

查詢`deployment`資訊
```
kubectl get deployment
```

若想要刪除`deployment`

```
root@vm1:~# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
mysql-1609235295   0/1     1            0           2d6h
web1               1/1     1            1           2d6h
root@vm1:~# kubectl delete deployment mysql-1609235295
deployment.apps "mysql-1609235295" deleted
root@vm1:~# kubectl get deployment
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web1   1/1     1            1           2d6h
```
若要更改副本數，可以使用`scale`
```
kubectl scale deployment web1 --replicas 2
```

```
root@vm1:~# kubectl scale deployment web1 --replicas 2
deployment. extenions/web1 scaled
root@vm1:~# kubectl get deployment
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web1   2/2     2            2           2d6h
```

想查詢`web1`更多資訊可使用`describe`
```
kubectl describe deployment web1
```

```
root@vm1:~# kubectl describe deployment web1
Name:                   web1
Namespace:              default
CreationTimestamp:      Tue, 29 Dec 2020 01:50:56 -0800
Labels:                 app=web1
                        app.kubernetes.io/managed-by=Helm
Annotations:            deployment.kubernetes.io/revision: 1
                        meta.helm.sh/release-name: abc-1609235456
                        meta.helm.sh/release-namespace: default
Selector:               app=web1
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=web1
  Containers:
   httpd:
    Image:        httpd:2.4.46
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web1-7469c97b99 (1/1 replicas created)
Events:          <none>
```

查看`pods`資訊
```
root@vm1:~# kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
web1-7469c97b99-d97gd   1/1     Running   0          174m
```
使用`describe`查看更多資訊
```
root@vm1:~# kubectl describe pods web1-7469c97b99-d97gd
Name:         web1-7469c97b99-d97gd
Namespace:    default
Priority:     0
Node:         vm2/192.168.102.139
Start Time:   Thu, 31 Dec 2020 06:04:41 -0800
Labels:       app=web1
              pod-template-hash=7469c97b99
Annotations:  <none>
Status:       Running
IP:           10.244.1.4
IPs:
  IP:           10.244.1.4
Controlled By:  ReplicaSet/web1-7469c97b99
Containers:
  httpd:
    Container ID:   docker://35f137ca6577ebc6f4477e10d99f7692022bca067781e09d38064a80f30cb397
    Image:          httpd:2.4.46
    Image ID:       docker-pullable://httpd@sha256:a3a2886ec250194804974932eaf4a4ba2b77c4e7d551ddb63b01068bf70f4120
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 31 Dec 2020 06:04:43 -0800
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
Events:          <none>
```

查看`web1`的`yaml`格式
```
root@vm1:~# kubectl get deployment web1 -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    meta.helm.sh/release-name: abc-1609235456
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2020-12-29T09:50:56Z"
  generation: 1
  labels:
    app: web1
    app.kubernetes.io/managed-by: Helm
  managedFields:
  - apiVersion: apps/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:meta.helm.sh/release-name: {}
          f:meta.helm.sh/release-namespace: {}
        f:labels:
          .: {}
          f:app: {}
          f:app.kubernetes.io/managed-by: {}
      f:spec:
        f:progressDeadlineSeconds: {}
        f:replicas: {}
        f:revisionHistoryLimit: {}
        f:selector:
          f:matchLabels:
            .: {}
            f:app: {}
        f:strategy:
          f:rollingUpdate:
            .: {}
            f:maxSurge: {}
            f:maxUnavailable: {}
          f:type: {}
        f:template:
          f:metadata:
            f:labels:
              .: {}
              f:app: {}
          f:spec:
            f:containers:
              k:{"name":"httpd"}:
                .: {}
                f:image: {}
                f:imagePullPolicy: {}
                f:name: {}
                f:resources: {}
                f:terminationMessagePath: {}
                f:terminationMessagePolicy: {}
            f:dnsPolicy: {}
            f:restartPolicy: {}
            f:schedulerName: {}
            f:securityContext: {}
            f:terminationGracePeriodSeconds: {}
    manager: Go-http-client
    operation: Update
    time: "2020-12-29T09:50:56Z"
  - apiVersion: apps/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:deployment.kubernetes.io/revision: {}
      f:status:
        f:availableReplicas: {}
        f:conditions:
          .: {}
          k:{"type":"Available"}:
            .: {}
            f:lastTransitionTime: {}
            f:lastUpdateTime: {}
            f:message: {}
            f:reason: {}
            f:status: {}
            f:type: {}
          k:{"type":"Progressing"}:
            .: {}
            f:lastTransitionTime: {}
            f:lastUpdateTime: {}
            f:message: {}
            f:reason: {}
            f:status: {}
            f:type: {}
        f:observedGeneration: {}
        f:readyReplicas: {}
        f:replicas: {}
        f:updatedReplicas: {}
    manager: kube-controller-manager
    operation: Update
    time: "2020-12-31T14:04:44Z"
  name: web1
  namespace: default
  resourceVersion: "48095"
  selfLink: /apis/apps/v1/namespaces/default/deployments/web1
  uid: d1133f74-8c6f-471c-899e-197e7905be39
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: web1
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web1
    spec:
      containers:
      - image: httpd:2.4.46
        imagePullPolicy: IfNotPresent
        name: httpd
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2020-12-29T09:50:56Z"
    lastUpdateTime: "2020-12-29T09:50:58Z"
    message: ReplicaSet "web1-7469c97b99" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2020-12-31T14:04:44Z"
    lastUpdateTime: "2020-12-31T14:04:44Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

將`web1`輸出成`yaml`檔，並且編輯成`web2`
```
kubectl get deployment web1 -o yaml > myweb.yml
```
> 輸出格式：-o yaml

透過`yaml`腳本創建

```
root@vm1:~# kubectl apply -f myweb.yaml 
deployment.apps/web2 created
```

當你創建好之後，可以查看`deployment`和`pods`是否存在新的名叫`web2`
```
root@vm1:~# kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web1   1/1     1            1           2d7h
web2   1/1     1            1           7s
```

查看`pods`是否運行在哪個節點上
```
root@vm1:~# kubectl get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP           NODE   NOMINATED NODE   READINESS GATES
web1-7469c97b99-d97gd   1/1     Running   0          3h43m   10.244.1.4   vm2    <none>           <none>
web2-58b88d6994-d58bt   1/1     Running   0          2m20s   10.244.2.5   vm3    <none>           <none>
```

可使用`curl`測試你的服務是否成功執行

```
root@vm1:~# curl 127.0.0.1
<html><body><h1>It works!</h1></body></html>
```

### Reference 

1. https://blog.tomy168.com/2019/08/centos-76-kubernetes.html
2. https://kubernetes.io/docs/tutorials/kubernetes-basics/