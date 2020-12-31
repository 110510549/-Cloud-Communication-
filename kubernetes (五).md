# 雲端通訊整合實務(12/29)
###### tags: `docker`

## Dashboard
### Dashboard 安裝：Kubernetes 圖形化

創建一個文件，名稱為`dashboard.yaml`
```
gedit dashboard.yaml
```

dashboard.yaml：
```
apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30310
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.0.0-beta7
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.1
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
```

編輯完後，啟動 dashboard：
```
kubectl apply -f dashboard.yaml
```

```
root@vm1:~# kubectl apply -f dashboard.yaml 
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

使用`kubectl get deployment`進行查詢dashboard是否成功開啟
```
kubectl get deployment -n kubernetes-dashboard | grep dashboard
```

```
root@vm1:~# kubectl get deployment -n kubernetes-dashboard | grep dashboard
dashboard-metrics-scraper   1/1     1            1           20s
kubernetes-dashboard        1/1     1            1           20s
```

**查看所有的dashboard詳細資訊**
```
kubectl get all -n kubernetes-dashboard
```

```
root@vm1:~# kubectl get all -n kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-7445d59dfd-vvgjk   1/1     Running   0          92s
pod/kubernetes-dashboard-5bc88ff8f9-dstfn        1/1     Running   0          92s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.106.36.226   <none>        8000/TCP        92s
service/kubernetes-dashboard        NodePort    10.104.59.164   <none>        443:30310/TCP   92s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dashboard-metrics-scraper   1/1     1            1           92s
deployment.apps/kubernetes-dashboard        1/1     1            1           92s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/dashboard-metrics-scraper-7445d59dfd   1         1         1       92s
replicaset.apps/kubernetes-dashboard-5bc88ff8f9        1         1         1       92s
```

編輯`admin-role.yaml`：

```
gedit admin-role.yaml
```
admin-role.yaml：
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1                                                                           
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

編輯完後，啟動 admin-role：
```
kubectl apply -f admin-role.yaml
```

```
root@vm1:~# kubectl apply -f admin-role.yaml 
clusterrolebinding.rbac.authorization.k8s.io/admin created
serviceaccount/admin created
```

取得帳戶token
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin | awk '{print $1}') | grep token: | awk -F : '{print $2}' | xargs echo
```

取得的token如下：

```
eyJhbGciOiJSUzI1NiIsImtpZCI6InZtank1dk82a1diUTkzNnA4akpTQ2E2TmRUdmtZc0NOdkdXY0tZR3JYNlEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1ieDlzbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjY2MzAwOTg4LWI5MTQtNGNmNy1hYmU5LTRiNmQ1NDI0YmNhZSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.HRtTURv_RUpYTGQdNo4b0IVi_w_OsfvNKYjEP-hrRPtPlhNgF9Cl74e96Fcf3I9h_jIIm4Dur_XYsjahiSQG1OnLxbXAgHUH65tJaIY1fLtdMLSY3lGXjFuzP4aD17bBjuZE2c-hUzrLL597G4zW0WO5uXEYRFJ-OuRIDv-3-OMiUwNQoKlID7kINEaa15EGABXjJA6s2n1c7WvJtc_L02qnjKZnzCEpLZTeFLVzwZe3v-hUh-9Q_-SlzGA5Bbi8kldfMh3R0g1wjqS_cwPF7qJQzssMCkvT9GWVo7Eng1BZ-nwvAmAFRB8SIw3CxhOxEAXasZ7QcTAlHRbHBW6yjw
```

`VM1`瀏覽器輸入以下URL：
```
https://192.168.102.146:30310/#!/login
```

![](https://i.imgur.com/TnxToby.png)

將取得的token貼至瀏覽器下即可登入，登入結果如下

![](https://i.imgur.com/QD50s0a.png)


![](https://i.imgur.com/AeRTF9A.png)


## Helm(Version3)
### Helm 介紹

Helm 是 Kubernetes 的包管理工具，有點類似 ubuntu 的`apt`和 centos 的`yum`安裝，可以方便的將之前打包好的yaml文件部署到 k8s 上，並透過 `chart` 的形式發佈，讓大家可以方便在 k8s 上安裝特定的服務。

### Helm 可以解決哪些問題？

1. 使用 helm 可以把這些`yaml`做一個整體管理
2. 實現`yaml`高效復用
3. 使用 helm 應用及別的版本管理

### Helm Version3 (in 2019)

1. v3刪除了`tiller`
2. release 可以在不同命名空間重用
3. 將`chart`推送到docker倉庫中

* v3之前的版本：

![](https://i.imgur.com/CQDGDMb.png)


* v3版本：

![](https://i.imgur.com/4SUCqcp.png)
===
(圖出處:[anida huang](https://anida-huang.gitbook.io/cloud-communication/qi-mo/20201229-kubernetes-wu#helm)提供)


### Helm 安裝


下載 Helm 壓縮包
```
wget https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
```

```
root@vm1:~# wget https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
--2020-12-29 01:36:13--  https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
Resolving get.helm.sh (get.helm.sh)... 152.199.39.108, 2606:2800:247:1cb7:261b:1f9c:2074:3c
Connecting to get.helm.sh (get.helm.sh)|152.199.39.108|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 13317454 (13M) [application/x-tar]
Saving to: ‘helm-v3.4.2-linux-amd64.tar.gz’

helm-v3.4.2-linux-a 100%[===================>]  12.70M  3.61MB/s    in 3.5s    

2020-12-29 01:36:18 (3.61 MB/s) - ‘helm-v3.4.2-linux-amd64.tar.gz’ saved [13317454/13317454]
```

解壓縮

```
tar xvfz helm-v3.4.2-linux-amd64.tar.gz
```

```
root@vm1:~# tar xvfz helm-v3.4.2-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/README.md
linux-amd64/LICENSE
```

進入到`linux-amd64/`資料夾，並且搬移 `helm` 到`/usr/local/bin`路徑下

```
root@vm1:~/linux-amd64# mv helm /usr/local/bin
```

測試helm是否可執行，可以用`helm version`進行測試：

```
root@vm1:~# helm version
version.BuildInfo{Version:"v3.4.2", GitCommit:"23dd3af5e19a02d4f4baa5b2f242645a1a3af629", GitTreeState:"clean", GoVersion:"go1.14.13"}
root@vm1:~# helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories
```

初始化

```
helm repo add stable https://charts.helm.sh/stable
```

```
root@vm1:~# helm repo add stable https://charts.helm.sh/stable
"stable" has been added to your repositories
```

使用 helm repo 命令管理
```
helm search repo stable
```

```
root@vm1:~# helm search repo stable
NAME                                 	CHART VERSION	APP VERSION            DESCRIPTION                                       
stable/acs-engine-autoscaler         	2.2.2        	2.1.1                  DEPRECATED Scales worker nodes within agent pools 
stable/aerospike                     	0.3.5        	v4.5.0.5               DEPRECATED A Helm chart for Aerospike in Kubern...
stable/airflow                       	7.13.3       	1.10.12                DEPRECATED - please use: https://github.com/air...
stable/ambassador                    	5.3.2        	0.86.1                 DEPRECATED A Helm chart for Datawire Ambassador   
stable/anchore-engine                	1.7.0        	0.7.3                  Anchore container analysis and policy evaluatio...

˙˙˙
(省略)
˙˙˙
```


新增 `aliyun` 到本地倉儲
```
root@vm1:~# helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
"aliyun" has been added to your repositories
```

進行查詢

```
root@vm1:~# helm repo ls
NAME  	URL                                                   
stable	https://charts.helm.sh/stable                         
aliyun	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

移除
```
helm repo remove [repoName]
```

更新

```
root@vm1:~# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "aliyun" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
```


安裝 chart
```
root@vm1:~# helm install stable/mysql --generate-name
WARNING: This chart is deprecated
NAME: mysql-1609235295
LAST DEPLOYED: Tue Dec 29 01:48:17 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mysql-1609235295.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql-1609235295 -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h mysql-1609235295 -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/mysql-1609235295 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```


查詢資料庫
```
helm search repo mysql
```
```
root@vm1:~# helm search repo mysql
NAME                            	CHART VERSION	APP VERSION	DESCRIPTION                                       
aliyun/mysql                    	0.3.5        	           	Fast, reliable, scalable, and easy to use open-...
stable/mysql                    	1.6.9        	5.7.30     	DEPRECATED - Fast, reliable, scalable, and easy...
stable/mysqldump                	2.6.2        	2.4.1      	DEPRECATED! - A Helm chart to help backup MySQL...
stable/prometheus-mysql-exporter	0.7.1        	v0.11.0    	DEPRECATED A Helm chart for prometheus mysql ex...
aliyun/percona                  	0.3.0        	           	free, fully compatible, enhanced, open source d...
aliyun/percona-xtradb-cluster   	0.0.2        	5.7.19     	free, fully compatible, enhanced, open source d...
stable/percona                  	1.2.3        	5.7.26     	DEPRECATED - free, fully compatible, enhanced, ...
stable/percona-xtradb-cluster   	1.0.8        	5.7.19     	DEPRECATED - free, fully compatible, enhanced, ...
stable/phpmyadmin               	4.3.5        	5.0.1      	DEPRECATED phpMyAdmin is an mysql administratio...
aliyun/gcloud-sqlproxy          	0.2.3        	           	Google Cloud SQL Proxy                            
aliyun/mariadb                  	2.1.6        	10.1.31    	Fast, reliable, scalable, and easy to use open-...
stable/gcloud-sqlproxy          	0.6.1        	1.11       	DEPRECATED Google Cloud SQL Proxy                 
stable/mariadb                  	7.3.14       	10.3.22    	DEPRECATED Fast, reliable, scalable, and easy t...
```

安裝完即可用`helm ls`查看
```
helm ls
```

```
root@vm1:~# helm ls
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
mysql-1609235295	default  	1       	2020-12-29 01:48:17.594460578 -0800 PST	deployed	mysql-1.6.9	5.7.30     
```

創建資料夾為abc，並且進入到`templates`資料夾內刪除所有資料並且新增`deployment.yaml`和`service.yaml`

```
root@vm1:~# helm create abc
Creating abc
root@vm1:~# ls
abc                             linux-amd64
admin-role.yaml                 ngrok
dashboard.yaml                  ngrok-stable-linux-amd64.zip
harbor                          pv
harbor1.9.0.tgz                 test-ingress-nginx
helm-v3.4.2-linux-amd64.tar.gz  zabbix-release_4.0-3+xenial_all.deb
iris
root@vm1:~# cd abc
root@vm1:~/abc# cd templates/
root@vm1:~/abc/templates# rm -rf *
root@vm1:~/abc/templates# gedit deployment.yaml service.yaml &
[2] 82488
```

deployment.yaml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web1
  name: web1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web1
    spec:
      containers:
      - image: httpd:2.4.46
        name: httpd
        resources: {}
status: {}
```

service.yaml:
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: web1
  name: web1
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: web1
  type: NodePort
status:
  loadBalancer: {}
```

安裝 abc
```
helm install abc --generate-name
```


用`helm ls`進行查看
```
helm ls
```
```
root@vm1:~# helm ls
NAME            	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
abc-1609235456  	default  	1       	2020-12-29 01:50:56.304874079 -0800 PST	deployed	abc-0.1.0  	1.16.0     
mysql-1609235295	default  	1       	2020-12-29 01:48:17.594460578 -0800 PST	deployed	mysql-1.6.9	5.7.30     
```
查看 deployment
```
kubectl get deployment
```
```
root@vm1:~# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
web1               1/1     1            1           26s
```


查看 service

```
root@vm1:~# kubectl get service
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP        58m
mysql-1609235295   ClusterIP   10.110.216.211   <none>        3306/TCP       3m18s
web1               NodePort    10.97.190.191    <none>        80:30280/TCP   39s
```

用`curl`進行測試

```
root@vm3:/home/user# curl 127.0.0.1:30280
<html><body><h1>It works!</h1></body></html>
```

### Reference

1. [上課影片](https://drive.google.com/drive/folders/1oM_ejAeSIhGbDGAVBDnVRv4DuoXaDwYw?usp=sharing)
2. https://godleon.github.io/blog/Kubernetes/k8s-Deploy-and-Access-Dashboard/
3. https://kenwu0310.wordpress.com/2019/01/16/centos-7-安裝-kubernetes-二-安裝-kubernetes-dashboard/
4. https://github.com/helm/helm/releases
5. https://helm.sh/zh/docs/intro/quickstart/



---

###### written by yihaotu