# 12/15(Note) - Cloud Communication

###### tags: `docker`、`kubernetes`
## Pre-Prepare
| IP Address | Hostname | CPU | Ram | OS |
|:---------------:|:---------------:|:---------------:|:---------------:|:---------------:|
| 192.168.102.135 | vm1(master) | 2 | 4 | ubuntu(16.04) |
| 192.168.102.139 | vm2(worker) | 2 | 4 | ubuntu(16.04) |
| 192.168.102.140 | vm3(worker) | 2 | 4 | ubuntu(16.04) |

## Intro
### the Ingress in kubernetes
Ingress is one of kubernetes using, it usually processing of distributing a set of tasks over a set of resources, like [load balance](https://en.wikipedia.org/wiki/Load_balancing_(computing)). but in the kubernetes, it's call **Ingress**.

1. Normally, source and destination using packet transmission sounds like:

![](https://i.imgur.com/zD7daBq.png)

> request once, and response once. but still have some problem, if no response packet back when you deliver a request packet, it must be wait for the packet, so now is a better way to improve it. ↓

2. with the Ingress in kubernetes, it could be like:

![](https://i.imgur.com/bBEb6e6.png)

> With Ingress, source just only need to transmission once, and then, Ingress will progressing the packet of destination. Ingress it can calculate the packet of response. if have, back to the source; if not, just keep wait. it will be more efficiency than upper way.

## Implement
### Ingress

1. Download test-ingress-nginx.zip📦 from **telegram**, and unzip the file:

![](https://i.imgur.com/BMMESzb.png)

2. pull the file into the `vm1`, and type this command:
```
cp -r '/var/run/vmblock-fuse/blockdir/pTM1hY/test-ingress-nginx' /root
```

![Imgur](https://i.imgur.com/f6xbhl7.mp4)

> cp(copy) : copy a directory to some specified path.
> ⇢ cp -r /(source) path /(destination) path
> 
> -r(recursive processing) : Process all files in the specified directory with subdirectories.

3. switch to the `/root`, and `cd` into test-ingress-nginx file, then type:
```
kubectl apply -f mandatory.yaml
kubectl apply -f service-nodeport.yaml
```
> -f : file

after then, you can type this command `kubectl get deployments` ,and you will found something wrong.

```
root@vm1:~# kubectl get deployments
No resources found in default namespace.
```
when you build the `mandatory.yaml` and `service-nodeport.yaml`, Normally it will be have a deployment in kubernetes, but in fact is, you need to type `kubectl get deployment -n ingress-nginx`.

```
root@vm1:~# kubectl get deployment -n ingress-nginx
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress-controller   1/1     1            1           2d2h
```
4. when you successful to build the deployment, you can also check `kubectl get pods -n ingress-nginx`
```
root@vm1:~# kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-54b86f8f7b-8d7xv   1/1     Running   0          3d20h
```
> **some tips you should know**: when u build a deployment, the pods will be create by deployment, so when u delete a pod, it will be create again, because of the deployment, so if u delete the deployment first, then delete pods after, then the pods will not create again.

you also can see the service in kubernetes, with `kubectl get svc -n ingress-nginx`:
```
root@vm1:~# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.106.251.88   <none>        80:31080/TCP,443:32527/TCP   3d20h
```

> mandatory.yaml: build a deployment of ingress-nginx
> service-nodeport.yaml: build a service of ingress-nginx
> 
> ⇢ see more detail, using `gedit mandatory.yaml service-nodeport.yaml &` to check how it work.


5. so now, you can using `kubectl apply -f httpd.yaml` to create service of httpd.
```
root@vm1:~/test-ingress-nginx# kubectl apply -f httpd.yaml 
deployment.apps/httpd created
service/httpd created
```

then, you can type `kubectl get svc`to check the httpd's `cluster-IP`

```
root@vm1:~/test-ingress-nginx# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
httpd        ClusterIP   10.106.208.70   <none>        80/TCP    6m33s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   10d
```

6. then, u can using `curl 10.106.208.70` to test, the result it will be like this:

```
root@vm1:~/test-ingress-nginx# curl 10.106.208.70
<html><body><h1>It works!</h1></body></html>
```

it means ur `httpd` is work on kubernetes, and then u can type `kubectl apply -f ingress-httpd.yaml` to build ingress-nginx on httpd.

```
root@vm1:~/test-ingress-nginx# kubectl apply -f ingress-httpd.yaml 
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.extensions/nginx-ingress created
```
> ⇢ the important things is before u type the `kubectl apply -f ingress-httpd.yaml`, u need to `gedit ingress-httpd.yaml` first, and change the "serviceName" like that.
> ![](https://i.imgur.com/shQjMfY.png)


and result it will be like this:
```
root@vm1:~/test-ingress-nginx# kubectl get ingress
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME            CLASS    HOSTS       ADDRESS   PORTS   AGE
nginx-ingress   <none>   www.a.com             80      24s
```
7. open terminal on ur macbook, and type `vim /etc/hosts`, then add the command `ip www.a.com` in the last line.

![](https://i.imgur.com/ikyTN7f.png)

u can using `ping www.a.com` to check the command is work on the `/etc/hosts`.

![](https://i.imgur.com/NJqbqQT.png)

after that, check the port of the ingress-nginx, using `kubectl get svc -n ingress-nginx`, and test the ingress is work on httpd successfully? the result it will be like that:

![](https://i.imgur.com/WCJcwDr.png)

## How it work on the pods of Kubernetes?
### work on where?

1. to prove the Ingress is work on different pods, first we can type `kubectl get pods -o wide` to see more information of pods.

```
root@vm1:~/test-ingress-nginx# kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE   NOMINATED NODE   READINESS GATES
httpd-6d7786f6cf-mcwvr   1/1     Running   0          63m   10.244.1.8   vm3    <none>           <none>
httpd-6d7786f6cf-nhznv   1/1     Running   0          63m   10.244.2.9   vm2    <none>           <none>
```

then, u can using `kubectl exec httpd-6d7786f6cf-mcwvr -it -- bash` to get into the pods of `vm3`.

```
root@vm1:~/test-ingress-nginx# kubectl exec httpd-6d7786f6cf-mcwvr -it -- bash
root@httpd-6d7786f6cf-mcwvr:/usr/local/apache2# cd htdocs/
root@httpd-6d7786f6cf-mcwvr:/usr/local/apache2/htdocs# ls
index.html
```

`cd` into `htdocs` file, and type `echo "vm3 www.a.com" > hi.htm`, then `exit`.

```
root@httpd-6d7786f6cf-mcwvr:/usr/local/apache2/htdocs# echo "vm3 www.a.com" > hi.htm
root@httpd-6d7786f6cf-mcwvr:/usr/local/apache2/htdocs# ls
hi.htm	index.html
root@httpd-6d7786f6cf-mcwvr:/usr/local/apache2/htdocs# exit
exit
```
another pods also like that.

```
root@vm1:~/test-ingress-nginx# kubectl exec httpd-6d7786f6cf-nhznv -it -- bash
root@httpd-6d7786f6cf-nhznv:/usr/local/apache2# cd htdocs/
root@httpd-6d7786f6cf-nhznv:/usr/local/apache2/htdocs# ls
index.html
root@httpd-6d7786f6cf-nhznv:/usr/local/apache2/htdocs# echo "vm2 www.a.com" > hi.htm
root@httpd-6d7786f6cf-nhznv:/usr/local/apache2/htdocs# exit
exit
```

2. test the website:

![Imgur](https://imgur.com/oaSInnj.mp4)

## Testing time
### Make belong with ur own Website

1. first, using `cp` to copy `httpd.yaml` and `ingress-httpd.yaml` and make other one like `httpd2.yaml`、`ingress-httpd2.yaml`.

```
cp httpd.yaml httpd2.yaml
cp ingress-httpd.yaml ingress-httpd2.yaml
```

and `gedit http2.yaml`, change any "httpd" to "httpd2".
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd2
  labels:
    app: httpd2
spec:
  selector:
    matchLabels:
      app: httpd2
  replicas: 1	
  template:
    metadata:
      labels:
        app: httpd2
    spec:
      containers:
      - name: httpd2
        image: httpd:2.4.46
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpd2
spec:
  selector:
    app: httpd2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

and `ingress-httpd2.yaml` need to change the "name", "host" and "serviceName" like that.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress2
spec:
  rules:
  - host: www.b.com
    http:
      paths:
      - path: /
        backend:
          serviceName: httpd2
          servicePort: 80
```
2. using `kubectl apply -f http2.yaml` and `kubectl apply -f ingress-httpd2.yaml` to build the deployment and service of kubernetes.
```
root@vm1:~/test-ingress-nginx# kubectl apply -f httpd2.yaml 
deployment.apps/httpd2 created
service/httpd2 created
root@vm1:~/test-ingress-nginx# kubectl apply -f ingress-httpd2.yaml 
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.extensions/nginx-ingress2 created
```

and when u built it, u can see the result from `kubectl get svc`、`kubectl get deployment` and `kubectl get ingress` to see more detail about `httpd2` and `ingress2`.
```
root@vm1:~/test-ingress-nginx# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
httpd        ClusterIP   10.110.120.93    <none>        80/TCP    37m
httpd2       ClusterIP   10.106.205.148   <none>        80/TCP    13m
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   46m
root@vm1:~/test-ingress-nginx# kubectl get deployment
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
httpd    2/2     2            2           37m
httpd2   1/1     1            1           13m
root@vm1:~/test-ingress-nginx# kubectl get ingress
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME             CLASS    HOSTS       ADDRESS          PORTS   AGE
nginx-ingress    <none>   www.a.com   10.106.253.124   80      42m
nginx-ingress2   <none>   www.b.com   10.106.253.124   80      18m
```

3. check the pods of httpd2, and simulation like httpd.

```
root@vm1:~/test-ingress-nginx# kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
httpd-6d7786f6cf-5fj65    1/1     Running   0          44m
httpd-6d7786f6cf-xtrhx    1/1     Running   0          44m
httpd2-84c8957db7-c2jlw   1/1     Running   0          20m
root@vm1:~/test-ingress-nginx# kubectl exec httpd2-84c8957db7-c2jlw -it -- bash
root@httpd2-84c8957db7-c2jlw:/usr/local/apache2# cd htdocs/
root@httpd2-84c8957db7-c2jlw:/usr/local/apache2/htdocs# ls
index.html
root@httpd2-84c8957db7-c2jlw:/usr/local/apache2/htdocs# echo "www.b.com" > hi.htm 
root@httpd2-84c8957db7-c2jlw:/usr/local/apache2/htdocs# ls
hi.htm	index.html
root@httpd2-84c8957db7-c2jlw:/usr/local/apache2/htdocs# exit
exit
```

![Imgur](https://imgur.com/kzZeNxZ.mp4)

## Reference 

Ingress 控制器介紹: https://www.cnblogs.com/along21/p/10333086.html

###### written by yihaotu.