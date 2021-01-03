# 雲端通訊整合實務(12/22)
###### tags: `docker`

## PersistentVolume of Kubernetes
### nfs-server

安裝NFS **(`vm1~vm3`都要安裝!!!)**

```
sudo apt-get install nfs-kernel-server nfs-common
```

#### VM1
設定分享的目錄，例如`/var/nfsshare/`

```
sudo mkdir -p /var/nfsshare
sudo chmod -R 777 /var/nfsshare/
```

開啟`/etc/exports`，並加入以下內容
```
/var/nfsshare 192.168.102.0/24(rw,sync,no_root_squash,no_all_squash)
```

> 本地端`vm1`會有一個目錄`/var/nfsshare/`，而`vm2`、`vm3`可以透過此路徑來進行掛載


重啟服務
```
systemctl restart rpcbind
systemctl restart nfs-server
```

重啟完確認一下服務是否啟動

```
root@vm1:~# systemctl status rpcbind
● rpcbind.service - RPC bind portmap service
   Loaded: loaded (/lib/systemd/system/rpcbind.service; enabled; vendor preset: 
  Drop-In: /run/systemd/generator/rpcbind.service.d
           └─50-rpcbind-$portmap.conf
   Active: active (running) since Mon 2020-12-28 21:34:55 PST; 4 days ago
 Main PID: 608 (rpcbind)
    Tasks: 1
   Memory: 1.1M
      CPU: 292ms
   CGroup: /system.slice/rpcbind.service
           └─608 /sbin/rpcbind -f -w

Warning: Journal has been rotated since unit was started. Log output is incomple
root@vm1:~# systemctl status nfs-server
● nfs-server.service - NFS server and services
   Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor prese
   Active: active (exited) since Mon 2020-12-28 21:34:55 PST; 4 days ago
 Main PID: 831 (code=exited, status=0/SUCCESS)
    Tasks: 0
   Memory: 0B
      CPU: 0
   CGroup: /system.slice/nfs-server.service

Warning: Journal has been rotated since unit was started. Log output is incomple
root@vm1:~# 
```

#### VM2、VM3

創建客戶端的存放路徑
```
sudo mkdir -p /mnt/nfs/var/nfsshare
```

掛載到`VM1`上

```
mount -t nfs 192.168.102.146:/var/nfsshare /mnt/nfs/var/nfsshare
```
掛載完之後，`cd`切換到`/mnt/nfs/var/nfsshare`路徑下，並且使用`touch`進行測試

```
touch a b c d
```


完成後，看到這樣的結果就代表你的客戶端`vm2`、`vm3`以及服務端`vm1`已經同步掛載了

Client:
```
root@vm2:/mnt/nfs/var/nfsshare# touch a b c d
root@vm2:/mnt/nfs/var/nfsshare# ls
a  b  c  d
```
Server:
```
root@vm1:/var/nfsshare# pwd
/var/nfsshare
root@vm1:/var/nfsshare# ls
a  b  c  d
```


* 確認能成功掛載完後記得使用`umount /mnt/nfs/var/nfsshare`進行卸載
---

### 在 kubernetes 提供網路儲存(持久化)
編輯3個檔案，分別為`1.yaml`、`2.yaml`以及`3.yaml`

1.yaml:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: test
  nfs:
    path: /var/nfsshare
    server: 192.168.102.135
```
> 建立一個 my-pv 服務，儲存空間為5G

2.yaml:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: test 
```
> 向 my-pv 索取1G的空間，若成功索取到，便會生成一個 my-pvc 的服務


3.yaml:

```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: my-pvc
  containers:
    - name: task-pv-container
      image: httpd:2.4.46
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/local/apache2/htdocs"
          name: task-pv-storage
```
> 將 my-pvc 更改成 task-pv-pod，並且與 httpd 預設路徑做綁定


編輯完後，啟動 1.yaml

```
kubectl apply -f 1.yaml
```

```
root@vm1:~/pv# kubectl apply -f 1.yaml 
persistentvolume/my-pv created
root@vm1:~/pv# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
my-pv   5Gi        RWX            Recycle          Bound    default/my-pvc   test                    9s
```

建立好pv服務後，啟動 2.yaml 檔
```
root@vm1:~/pv# kubectl apply -f 2.yaml 
persistentvolumeclaim/my-pvc created
root@vm1:~/pv# kubectl get pvc
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    my-pv    5Gi        RWX            test           5s
```

完成後可以看到，my-pv 以及 my-pvc 已經綁定再一起了

接著啟動 3.yaml

```
root@vm1:~/pv# kubectl apply -f 3.yaml 
pod/task-pv-pod created
root@vm1:~/pv# kubectl get pods
NAME                      READY   STATUS              RESTARTS   AGE
task-pv-pod               1/1     Running             0          5s
```

接著查看 pod 資訊
```
root@vm1:~/pv# kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
task-pv-pod               1/1     Running   0          2m37s   10.244.2.9    vm3    <none>           <none>

```

進行測試
```
root@vm3:/home/user# curl 10.244.2.9/hi.html
hi
root@vm3:/home/user# curl 10.244.2.9/hello.html
hello
```

為了證明**持久化**的特性，我們先刪除 pod
```
root@vm1:~/pv# kubectl delete pods task-pv-pod
pod "task-pv-pod" deleted
root@vm1:~/pv# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
```

再重新啟動一次 3.yaml

```
root@vm1:~/pv# kubectl apply -f 3.yaml 
pod/task-pv-pod created
root@vm1:~/pv# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
task-pv-pod               1/1     Running   0          5s
```

查看 pod 資訊，會看到 task-pv-pod 的`IP`生成新的，因此你可以再重新測試一次
```
root@vm1:~/pv# kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
task-pv-pod               1/1     Running   0          9s      10.244.2.10   vm3    <none>           <none>
```

一樣可以看到是可行的！
```
root@vm3:/home/user# curl 10.244.2.10/hello.html
hello
root@vm3:/home/user# curl 10.244.2.10/hi.html
hi
```

> **持久化**目前較常用的是資料庫，當在寫動態網頁時發現資料庫掛了，可以透過這樣的方式將資料庫重新啟動，而資料依舊存在。

---

### 在 kubernetes 提供網路儲存(持久化)-2

與上面的程式不同在於，pvc.yaml 程式多了 **deployment**，因此當你 pod 死掉後，deployment 會再重新生成一個新的 pod，並且維持固定的副本數

pv.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /var/nfsshare
    server: 192.168.102.146
```

pvc.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-dep2
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 1
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
      - name: httpd
        image: httpd:2.4.46
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/local/apache2/htdocs
        ports:
        - containerPort: 80
      volumes:
        - name: wwwroot
          persistentVolumeClaim:
            claimName: my-pvc

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

---

### configMap

1. 建立 yaml 檔並啟動 configmap 服務

創建一個檔案`configmap.yaml`
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: cm-demo
  namespace: default
data:
  data.1: hello
  data.2: world
  config: |
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

啟動服務
```
root@vm1:~# kubectl apply -f configmap.yaml 
configmap/cm-demo created
root@vm1:~# kubectl get configmap
NAME                    DATA   AGE
cm-demo                 3      12s
root@vm1:~# kubectl get cm
NAME                    DATA   AGE
cm-demo                 3      26s
```
> `kubectl get cm` = `kubectl get configmap`


查看詳細資訊

```
root@vm1:~# kubectl describe cm cm-demo
Name:         cm-demo
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
config:
----
property.1=value-1
property.2=value-2
property.3=value-3

data.1:
----
hello
data.2:
----
world
Events:  <none>
```
---
2. 搭配 file 來生成 configmap 服務

創建一個資料夾`testcm`，並創建兩個檔案`mysql.conf`以及`redis.conf`，其內容如下
```
root@vm1:~/testcm# ls
mysql.conf  redis.conf
root@vm1:~/testcm# cat mysql.conf 
host=127.0.0.1
port=3306
root@vm1:~/testcm# cat redis.conf 
host=127.0.0.1
port=6379
```

創建完後，回到上層並啟動
```
*root@vm1:~# kubectl create configmap cm-demo1 --from-file=testcm
configmap/cm-demo1 created
root@vm1:~# kubectl get cm
NAME                    DATA   AGE
cm-demo                 3      7m58s
cm-demo1                2      9s
```

查看新建的 cm-demo1 資訊
```
root@vm1:~/testcm# kubectl describe cm cm-demo1
Name:         cm-demo1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
mysql.conf:
----
host=127.0.0.1
port=3306

redis.conf:
----
host=127.0.0.1
port=6379

Events:  <none>
```
---

3. 直接用 command 生成

```
kubectl create configmap cm-demo3 --from-literal=db.host=localhost --from-literal=db.port=3306
```

```
root@vm1:~/testcm# kubectl create configmap cm-demo3 --from-literal=db.host=localhost --from-literal=db.port=3306
configmap/cm-demo3 created
root@vm1:~/testcm# kubectl get cm
NAME                    DATA   AGE
cm-demo                 3      15m
cm-demo1                2      8m5s
cm-demo3                2      8s
root@vm1:~/testcm# kubectl describe cm cm-demo3
Name:         cm-demo3
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
db.host:
----
localhost
db.port:
----
3306
Events:  <none>
root@vm1:~/testcm
```
---

#### 如何使用?

創建 testpod.yaml

```
apiVersion: v1
kind: Pod
metadata:
  name: testcm1-pod
spec:
  containers:
    - name: testcm1
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.port
```

啟動服務
```
kubectl apply -f testpod.yaml
```
```
root@vm1:~# kubectl apply -f testpod.yaml 
pod/testcm1-pod created
```


```
root@vm1:~# kubectl get pods
NAME          READY   STATUS             RESTARTS   AGE
testcm1-pod   0/1     Completed          0          2m14s
```

使用`kubectl logs`進行查看
```
root@vm1:~# kubectl logs testcm1-pod
HOSTNAME=testcm1-pod
DB_PORT=3306
```

可以透過這樣的方式將 pod 的環境變量初始化，以及系統環境的設定