# 雲端通訊整合實務(11/17)
###### tags: `docker`、`docker swarm`

## docker swarm


初始化 docker swarm：
```
docker swarm init --advertise-addr 192.168.102.135
```
我們選擇`VM1`當作manager：
```
root@vm1:~# docker swarm init --advertise-addr 192.168.102.135
Swarm initialized: current node (zjtlil44moo62yxh1m4qxtw6t) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4jp4gdvzynvbeh5pwenqexsx2dnpljno5oo4t782ix9478vj97-5rveeqdg1k1uvdsk3w8re7vwq 192.168.102.135:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

將 token 的 command 丟到`VM2`、`VM3`上：
`VM2`為 worker：

```
root@vm2:/home/user/test-dockercompose# docker swarm join --token SWMTKN-1-4jp4gdvzynvbeh5pwenqexsx2dnpljno5oo4t782ix9478vj97-5rveeqdg1k1uvdsk3w8re7vwq 192.168.102.135:2377
This node joined a swarm as a worker.
```

`VM3`為 worker：

```
root@vm3:/home/user# docker swarm join --token SWMTKN-1-4jp4gdvzynvbeh5pwenqexsx2dnpljno5oo4t782ix9478vj97-5rveeqdg1k1uvdsk3w8re7vwq 192.168.102.135:2377
This node joined a swarm as a worker.
```

這時可以做一個簡單的測試，去測試 manager 與 worker 的差異：
對於 manager 來說，是可以使用這項指令`docker node ls`
```
root@vm1:~# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zjtlil44moo62yxh1m4qxtw6t *   vm1                 Ready               Active              Leader              19.03.7
zo2tgolcnqyj2kcutdy1dssh4     vm2                 Ready               Active                                  19.03.7
7m0j5hd9olqgdm8b77mnw7f3u     vm3                 Ready               Active                                  19.03.13

```
但是對於 worker 來說，`docker node ls`這項指令會出現報錯：
```
root@vm2:/home/user/test-dockercompose# docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
root@vm2:/home/user/test-dockercompose# 
```
```
root@vm3:/home/user# docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
root@vm3:/home/user# 
```
> 因為 `worker` 權限不夠去執行這行指令，只有`manager`去給予權限時才可使用。

更新vm1的docker swarm狀態為`drain`：
```
docker node update --availablity drain vm1
```
> drain：代表任何節點都不會在你的drain機器上工作，只有當你將drain狀態改回active後，才會回到該節點上工作。


```
root@vm1:~# docker node update --availability drain vm1
vm1
root@vm1:~# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zjtlil44moo62yxh1m4qxtw6t *   vm1                 Ready               Drain               Leader              19.03.7
zo2tgolcnqyj2kcutdy1dssh4     vm2                 Ready               Active                                  19.03.7
7m0j5hd9olqgdm8b77mnw7f3u     vm3                 Ready               Active                                  19.03.13
p57x8bdqz57au5stc4j08l1c0     vm3                 Down                Active                                  19.03.13
root@vm1:~# 
```

若想要移除當前的節點以及工作站：

1. 刪除`worker`節點:
```
root@vm1:~# docker node rm vm2
vm2
root@vm1:~# docker node rm vm3
vm3
```
2. 離開當前工作站:
```
root@vm1:~# docker swarm leave --force
Node left the swarm.
```
> force:強制執行
> `manager`和`worker`都要執行

## docker swarm visualizer：
從 **dockerhub** 去 `pull` visualizer 套件：
```
docker pull dockersamples/visualizer
```
```
root@vm1:~# docker pull dockersamples/visualizer
Using default tag: latest
latest: Pulling from dockersamples/visualizer
cd784148e348: Pull complete 
f6268ae5d1d7: Extracting  9.437MB/18.87MB
f6268ae5d1d7: Pull complete 
97eb9028b14b: Pull complete 
9975a7a2a3d1: Pull complete 
ba903e5e6801: Pull complete 
7f034edb1086: Pull complete 
cd5dbf77b483: Pull complete 
5e7311667ddb: Pull complete 
687c1072bfcb: Pull complete 
aa18e5d3472c: Pull complete 
a3da1957bd6b: Pull complete 
e42dbf1c67c4: Pull complete 
5a18b01011d2: Pull complete 
Digest: sha256:54d65cbcbff52ee7d789cd285fbe68f07a46e3419c8fcded437af4c616915c85
Status: Downloaded newer image for dockersamples/visualizer:latest
docker.io/dockersamples/visualizer:latest
```

將 docker swarm virtualizer 執行起來：
```
root@vm1:~# docker run -itd -p 8888:8080 -e HOST=192.168.102.135  -e PORT=8080 -v /var/run/docker.sock:/var/run/docker.sock --name visualizer dockersamples/visualizer
5250b5694fbb0a2c1dc3ce7ae242ca18f94b477435477bcd725a7321fe7bc970
```
確認一下容器已經執行：
```
root@vm1:~# docker ps -a
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS                   PORTS                    NAMES
5250b5694fbb        dockersamples/visualizer   "npm start"              3 minutes ago       Up 3 minutes (healthy)   0.0.0.0:8888->8080/tcp   visualizer
b3e97042c8f5        testgolang                 "./testgolang"           7 hours ago         Up 7 hours               0.0.0.0:8001->8001/tcp   testgolang
b8f4ca51448e        iris                       "/bin/sh -c 'python …"   25 hours ago        Up 25 hours              0.0.0.0:5000->5000/tcp   iris
becebb97850a        httpd                      "httpd-foreground"       30 hours ago        Up 30 hours              0.0.0.0:8080->80/tcp     laughing_tereshkova
```

接下來可以到本地端網頁進行查看：

![](https://i.imgur.com/NsOBntX.png)

出現這個結果代表已經成功執行了 docker swarm virtualizer 了。

使用docker swarm啟動web服務：
```
docker service create --name myweb -p 8880:80 httpd
```
啟動完後，用`docker service ls`進行查看：
```
root@vm1:~# docker service create --name myweb -p 8880:80 httpd
t6xl05inn6u0g1p8366kpohnn
overall progress: 1 out of 1 tasks 
1/1: running   
verify: Service converged 
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
t6xl05inn6u0        myweb               replicated          1/1                 httpd:latest        *:8880->80/tcp
root@vm1:~# 
```
而 myweb 會運行在 virtualizer 上：

![](https://i.imgur.com/DmQlGov.png)

若你想增加 myweb 的數目，可以使用：
```
docker service scale myweb=5
```
```
root@vm1:~# docker service scale myweb=5
myweb scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   
2/5: running   
3/5: running   
4/5: running   
5/5: running   
verify: Service converged 
root@vm1:~# 
```

其結果會像這樣：

![](https://i.imgur.com/AcYAEm2.png)

而你會發現，vm1上都沒有任何工作節點，是因為上面有將vm1的狀態設置成`drain`，因此任何工作節點都不會分配給vm1。

而你可以將vm1狀態改回`active`：
```
docker node update --availability active vm1
```

```
root@vm1:~# docker node update --availability active vm1
vm1
root@vm1:~# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zjtlil44moo62yxh1m4qxtw6t *   vm1                 Ready               Active              Leader              19.03.7
zo2tgolcnqyj2kcutdy1dssh4     vm2                 Ready               Active                                  19.03.7
7m0j5hd9olqgdm8b77mnw7f3u     vm3                 Ready               Active                                  19.03.13
p57x8bdqz57au5stc4j08l1c0     vm3                 Down                Active                                  19.03.13
root@vm1:~# docker service scale myweb=4
myweb scaled to 4
overall progress: 4 out of 4 tasks 
1/4: running   
2/4: running   
3/4: running   
4/4: running   
verify: Service converged 
root@vm1:~# docker service scale myweb=5
myweb scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   
2/5: running   
3/5: running   
4/5: running   
5/5: running   
verify: Service converged 
root@vm1:~# 
```

我們只需要確認狀態是`active`後，將節點數重新設置一次，你就會發現到：

![](https://i.imgur.com/uRayjT2.png)

自從vm1狀態改回`active`，工作節點`myweb`就會重新在vm1上執行了。

## docker swarm 應用

查看節點`myweb`運行狀態：

```
docker service ls myweb
```

而docker swarm有一個特點，就是當你其中一個vm上執行了一個節點，其他vm都能互相通用，以`myweb`為例：
當你輸入了`vm1`的ip以及`vm2`、`vm3`，

![](https://i.imgur.com/C5oNSDA.png)

![](https://i.imgur.com/MLpYbhD.png)

![](https://i.imgur.com/u0JxgwC.png)

你可以看到說，當你打上不同`vm`的ip時，它都是可以通的。


## docker swarm 網路環境

根據上個例子，明明只有一個節點工作在vm2，但是為什麼其他vm1、vm3可以同樣運行，原因是docker swarm內建的loan balance + Routing Mesh幫我們完成這些事。

**docker swarm network environment：**

![](https://i.imgur.com/Q3Sj3WY.png)


查看docker Network：
```
docker network ls
```
```
root@vm1:~# docker network ls
NETWORK ID          NAME                         DRIVER              SCOPE
b899f2b11ad8        bridge                       bridge              local
1a6eeeb1e7d1        docker_gwbridge              bridge              local
631d77694a9e        harbor_harbor                bridge              local
cadc2c574a1e        host                         host                local
e6p8a0mx0fze        ingress                      overlay             swarm
d6559cfe24a2        mynet                        bridge              local
c7c68c4b8261        none                         null                local
ace41fd2ee84        test-dockercompose_default   bridge              local
```

查詢某個網路的資訊，例如`overlay`：
```
docker network inspect e6p
```

```
(省略)
...
"IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.0.0.0/24",
                    "Gateway": "10.0.0.1"
                }
            ]
        },
...
(省略)
```
你可以看到`overlay`的subnet為`10.0.0.0/24`，以及gateway為`10.0.0.1`



```
root@vm1:~# docker ps 
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS                 PORTS                    NAMES
5250b5694fbb        dockersamples/visualizer   "npm start"              2 hours ago         Up 2 hours (healthy)   0.0.0.0:8888->8080/tcp   visualizer
b3e97042c8f5        testgolang                 "./testgolang"           9 hours ago         Up 9 hours             0.0.0.0:8001->8001/tcp   testgolang
b8f4ca51448e        iris                       "/bin/sh -c 'python …"   27 hours ago        Up 27 hours            0.0.0.0:5000->5000/tcp   iris
becebb97850a        httpd                      "httpd-foreground"       32 hours ago        Up 32 hours            0.0.0.0:8080->80/tcp     laughing_tereshkova
root@vm1:~# 
```

查看docker的`svc`資訊
```
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
f8qkvqabta11        web1                replicated          0/2                 httpd:latest        *:8000->80/tcp
```

刪除`service`服務

```
root@vm1:~# docker service rm web1
web1
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
```

`service`創建`httpd`服務，名稱叫`myweb`
```
root@vm1:~# docker service create --name myweb httpd
5pexf763sw4kmcudj511vetea
overall progress: 1 out of 1 tasks 
1/1: running   
verify: Service converged
```

更新`service`
```
docker service update --publish-add 8880:80 myweb
```

```
root@vm1:~# docker service update --publish-add 8880:80 myweb
myweb
overall progress: 1 out of 1 tasks 
1/1: running   
verify: Service converged 
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
5pexf763sw4k        myweb               replicated          1/1                 httpd:latest        *:8880->80/tcp
```

查看 `myweb` 的`log`

```
root@vm1:~# docker service logs 5pe
myweb.1.9u8pvtnanpzu@vm1    | AH00557: httpd: apr_sockaddr_info_get() failed for 1323f15512d9
myweb.1.9u8pvtnanpzu@vm1    | AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message
myweb.1.bmt564ann0wd@vm1    | AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.2. Set the 'ServerName' directive globally to suppress this message
myweb.1.9u8pvtnanpzu@vm1    | AH00557: httpd: apr_sockaddr_info_get() failed for 1323f15512d9
myweb.1.bmt564ann0wd@vm1    | AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.2. Set the 'ServerName' directive globally to suppress this message
myweb.1.9u8pvtnanpzu@vm1    | AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message
myweb.1.bmt564ann0wd@vm1    | [Fri Jan 01 07:57:27.762173 2021] [mpm_event:notice] [pid 1:tid 140315680138368] AH00489: Apache/2.4.46 (Unix) configured -- resuming normal operations
myweb.1.9u8pvtnanpzu@vm1    | [Fri Jan 01 08:09:59.074157 2021] [mpm_event:notice] [pid 1:tid 140496198902912] AH00489: Apache/2.4.46 (Unix) configured -- resuming normal operations
myweb.1.9u8pvtnanpzu@vm1    | [Fri Jan 01 08:09:59.074470 2021] [core:notice] [pid 1:tid 140496198902912] AH00094: Command line: 'httpd -D FOREGROUND'
myweb.1.bmt564ann0wd@vm1    | [Fri Jan 01 07:57:27.762306 2021] [core:notice] [pid 1:tid 140315680138368] AH00094: Command line: 'httpd -D FOREGROUND'
myweb.1.bmt564ann0wd@vm1    | [Fri Jan 01 08:09:56.249892 2021] [mpm_event:notice] [pid 1:tid 140315680138368] AH00492: caught SIGWINCH, shutting down gracefully
```

若要隨時監控 log 檔的話，可以使用`-f`:
![](https://i.imgur.com/9dX5vSd.png)
