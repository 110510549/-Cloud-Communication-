# 雲端通訊整合實務(11/24)
###### tags: `docker`、`docker swarm`


## Docker Swarm

* `docker swarm` v.s. `global mode`
* `rolling update` v.s. `rollback`
* `label`

---

### 建立 httpd service 服務
myweb1 : (docker swarm)、(bridge)

```
root@vm1:~# docker service create --replicas 3 --network bridge --name myweb1 -p 8001:80 httpd
1mwk8oo6vnpl7xf4ecv7p1c4e
overall progress: 3 out of 3 tasks 
1/3: running   
2/3: running   
3/3: running   
verify: Service converged 
```
myweb2 : (global)、(bridge)
```
root@vm1:~# docker service create --mode global --network bridge --name myweb2 -p 8002:80 httpd
2ik0ao32swx50qm5u2mdn9pqf
overall progress: 3 out of 3 tasks 
fqgx813hv9ma: running   
y77dc3wdwtl5: running   
wcxi0azvsafu: running   
verify: Service converged 
```
查看建立的`service`
```
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
1mwk8oo6vnpl        myweb1              replicated          3/3                 httpd:latest        *:8001->80/tcp
oesztde5fv0f        myweb2              global              3/3                 httpd:latest        *:8002->80/tcp
```

將其中一台 worker 關掉，並且測試`bridge`模式下，不同模式的 httpd 副本會如何進行部署

![Imgur](https://imgur.com/xEfanJK)

這邊可以看到，當關掉`vm2`後，myweb1 會先重新部署在其他`vm`上，而當重新開啟`vm2`時，myweb2 便會重新部署在該節點上

![](https://i.imgur.com/UvgJW6w.png)


### 滾動更新(Rolling Update)

首先，先下載 httpd:2.4.43 以及 httpd:2.4.46 版本
```
root@vm1:~# docker pull httpd:2.4.43
2.4.43: Pulling from library/httpd
bf5952930446: Pull complete 
3d3fecf6569b: Pull complete 
b5fc3125d912: Pull complete 
3c61041685c0: Pull complete 
34b7e9053f76: Pull complete 
Digest: sha256:cd88fee4eab37f0d8cd04b06ef97285ca981c27b4d685f0321e65c5d4fd49357
Status: Downloaded newer image for httpd:2.4.43
docker.io/library/httpd:2.4.43
root@vm1:~# docker pull httpd:2.4.46
2.4.46: Pulling from library/httpd
Digest: sha256:a3a2886ec250194804974932eaf4a4ba2b77c4e7d551ddb63b01068bf70f4120
Status: Downloaded newer image for httpd:2.4.46
docker.io/library/httpd:2.4.46
```

創建一個 httpd 服務，其中版本為 2.4.43

```
root@vm1:~# docker service create --name myweb3 --replicas 3 -p 8003:80 httpd:2.4.43
1feqyxs90hpznl97ff8lcewui
overall progress: 3 out of 3 tasks 
1/3: running   
2/3: running   
3/3: running   
verify: Service converged 
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
1mwk8oo6vnpl        myweb1              replicated          3/3                 httpd:latest        *:8001->80/tcp
oesztde5fv0f        myweb2              global              3/3                 httpd:latest        *:8002->80/tcp
1feqyxs90hpz        myweb3              replicated          3/3                 httpd:2.4.43        *:8003->80/tcp
```

將 httpd:2.4.43 版本更新至 httpd:2.4.46版本
```
root@vm1:~# docker service update --image httpd:2.4.46 myweb3
myweb3
overall progress: 3 out of 3 tasks 
1/3: running   
2/3: running   
3/3: running   
verify: Service converged 
```

![Imgur](https://imgur.com/T1AuvhT)

其中可以看到，當在滾動更新時，副本數是一項一項更新的，更新完後可以透過 visualizer 查看或是透過以下指令`docker service ls`去進行查看

![](https://i.imgur.com/mJJKnyW.png)


> 原本的 httpd 版本為2.4.43，更新後變為2.4.46

若想查看更多詳細的資訊，可以透過以下指令

```
docker service inspect myweb3
```

```
root@vm1:~# docker service inspect myweb3
[
    {
        "ID": "1feqyxs90hpznl97ff8lcewui",
        "Version": {
            "Index": 246
        },
        "CreatedAt": "2021-01-01T17:19:43.255972559Z",
        "UpdatedAt": "2021-01-01T17:26:28.522216482Z",
        "Spec": {
            "Name": "myweb3",
            "Labels": {},
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "httpd:2.4.46@sha256:a3a2886ec250194804974932eaf4a4ba2b77c4e7d551ddb63b01068bf70f4120",
                    "Init": false,
                    "StopGracePeriod": 10000000000,
                    "DNSConfig": {},
                    "Isolation": "default"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "Delay": 5000000000,
                    "MaxAttempts": 0
                },
                "Placement": {
                    "Platforms": [
                        {
                            "Architecture": "amd64",
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "Architecture": "arm64",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "386",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "mips64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "ppc64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "s390x",
                            "OS": "linux"
                        }
                    ]
                },
                "ForceUpdate": 0,
                "Runtime": "container"
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 3
                }
            },
            "UpdateConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "RollbackConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "EndpointSpec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 8003,
                        "PublishMode": "ingress"
                    }
                ]
            }
        },
        "PreviousSpec": {
            "Name": "myweb3",
            "Labels": {},
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "httpd:2.4.43@sha256:cd88fee4eab37f0d8cd04b06ef97285ca981c27b4d685f0321e65c5d4fd49357",
                    "Init": false,
                    "DNSConfig": {},
                    "Isolation": "default"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "Placement": {
                    "Platforms": [
                        {
                            "Architecture": "amd64",
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "OS": "linux"
                        },
                        {
                            "Architecture": "arm64",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "386",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "mips64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "ppc64le",
                            "OS": "linux"
                        },
                        {
                            "Architecture": "s390x",
                            "OS": "linux"
                        }
                    ]
                },
                "ForceUpdate": 0,
                "Runtime": "container"
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 3
                }
            },
            "EndpointSpec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 8003,
                        "PublishMode": "ingress"
                    }
                ]
            }
        },
        "Endpoint": {
            "Spec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 8003,
                        "PublishMode": "ingress"
                    }
                ]
            },
            "Ports": [
                {
                    "Protocol": "tcp",
                    "TargetPort": 80,
                    "PublishedPort": 8003,
                    "PublishMode": "ingress"
                }
            ],
            "VirtualIPs": [
                {
                    "NetworkID": "xmkrxzgs5r2oujawwoyicsd2f",
                    "Addr": "10.0.0.45/24"
                }
            ]
        },
        "UpdateStatus": {
            "State": "completed",
            "StartedAt": "2021-01-01T17:26:07.367557225Z",
            "CompletedAt": "2021-01-01T17:26:28.522197036Z",
            "Message": "update completed"
        }
    }
]
```

> Spec : 當前版本資訊
> PreviousSpec : 上一個版本資訊


### 更有效率的方法(Rolling Update)

重新生成 httpd:2.4.43，副本數為6個
```
root@vm1:~# docker service create --name myweb1 --replicas 6 -p 8001:80 httpd:2.4.43
1o7d2ckrtv4d5flhuh5ekct1d
overall progress: 6 out of 6 tasks 
1/6: running   
2/6: running   
3/6: running   
4/6: running   
5/6: running   
6/6: running   
verify: Service converged 
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
1o7d2ckrtv4d        myweb1              replicated          6/6                 httpd:2.4.43        *:8001->80/tcp
```

這次採用的方法是一次滾動更新兩個，每次更新完後會延遲20秒

```
docker service update --image httpd:2.4.46 --update-parallelism 2 --update-delay 10s myweb1
```

```
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
1o7d2ckrtv4d        myweb1              replicated          6/6                 httpd:2.4.43        *:8001->80/tcp
root@vm1:~# docker service update --image httpd:2.4.46 --update-parallelism 2 --update-delay 10s myweb1
myweb1
overall progress: 6 out of 6 tasks 
1/6: running   
2/6: running   
3/6: running   
4/6: running   
5/6: running   
6/6: running   
verify: Service converged 
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
1o7d2ckrtv4d        myweb1              replicated          6/6                 httpd:2.4.46        *:8001->80/tcp
```

![Imgur](https://imgur.com/8kSlHbQ)

### 回滾(Rolling Back)

若你的版本覺得太新，可以使用**回滾功能**回復到上一個版本
```
docker service update --rollback myweb1
```

```
root@vm1:~# docker service update --rollback myweb1
myweb1
rollback: manually requested rollback 
overall progress: rolling back update: 6 out of 6 tasks 
1/6: running   
2/6: running   
3/6: running   
4/6: running   
5/6: running   
6/6: running   
verify: Service converged 
root@vm1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
1o7d2ckrtv4d        myweb1              replicated          6/6                 httpd:2.4.43        *:8001->80/tcp
```

![](https://i.imgur.com/7xj5BzV.png)

可以看到，原本副本版本為2.4.46，使用`Rolling Back`後，版本回復到2.4.43

### 節點打標籤

先使用`docker node ls`查看節點名稱，接著為`vm2`打上標籤 test，`vm3`打上標籤 prod

```
docker node update --label-add env=test vm2
```

```
docker node update --label-add env=prod vm3
```

```
root@vm1:~# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
fqgx813hv9matyuzusja6e62p *   vm1                 Ready               Active              Leader              19.03.7
y77dc3wdwtl5eu6dcbgo9w8yb     vm2                 Ready               Active                                  19.03.7
wcxi0azvsafuygzetrdxfo1ds     vm3                 Ready               Active                                  19.03.13
root@vm1:~# docker node update --label-add env=test vm2
vm2
root@vm1:~# docker node update --label-add env=prod vm3
vm3
```

可以透過`docker node inspect [nodeName]`進行查看

vm2
```
root@vm1:~# docker node inspect vm2
[
    {
        "ID": "y77dc3wdwtl5eu6dcbgo9w8yb",
        "Version": {
            "Index": 384
        },
        "CreatedAt": "2021-01-01T08:36:00.600330768Z",
        "UpdatedAt": "2021-01-02T06:53:13.336117502Z",
        "Spec": {
            "Labels": {
                "env": "test"
            },
            "Role": "worker",
            "Availability": "active"
        },
        "Description": {
            "Hostname": "vm2",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
            },
            "Resources": {
                "NanoCPUs": 2000000000,
                "MemoryBytes": 4143312896
            },
            "Engine": {
                "EngineVersion": "19.03.7",
                "Plugins": [
                    {
                        "Type": "Log",
                        "Name": "awslogs"
                    },
                    {
                        "Type": "Log",
                        "Name": "fluentd"
                    },
                    {
                        "Type": "Log",
                        "Name": "gcplogs"
                    },
                    {
                        "Type": "Log",
                        "Name": "gelf"
                    },
                    {
                        "Type": "Log",
                        "Name": "journald"
                    },
                    {
                        "Type": "Log",
                        "Name": "json-file"
                    },
                    {
                        "Type": "Log",
                        "Name": "local"
                    },
                    {
                        "Type": "Log",
                        "Name": "logentries"
                    },
                    {
                        "Type": "Log",
                        "Name": "splunk"
                    },
                    {
                        "Type": "Log",
                        "Name": "syslog"
                    },
                    {
                        "Type": "Network",
                        "Name": "bridge"
                    },
                    {
                        "Type": "Network",
                        "Name": "host"
                    },
                    {
                        "Type": "Network",
                        "Name": "ipvlan"
                    },
                    {
                        "Type": "Network",
                        "Name": "macvlan"
                    },
                    {
                        "Type": "Network",
                        "Name": "null"
                    },
                    {
                        "Type": "Network",
                        "Name": "overlay"
                    },
                    {
                        "Type": "Volume",
                        "Name": "local"
                    }
                ]
            },
            "TLSInfo": {
                "TrustRoot": "-----BEGIN CERTIFICATE-----\nMIIBajCCARCgAwIBAgIUSFKIDtFHCjWqUNaUDwgyWjvCMnMwCgYIKoZIzj0EAwIw\nEzERMA8GA1UEAxMIc3dhcm0tY2EwHhcNMjEwMTAxMDgzMDAwWhcNNDAxMjI3MDgz\nMDAwWjATMREwDwYDVQQDEwhzd2FybS1jYTBZMBMGByqGSM49AgEGCCqGSM49AwEH\nA0IABD4ra8NXNqftfKRfV6cz4CafLJLnSL5bhTUUJ08OeBjxPU9k0c9j2CnHimSV\n2EOqV/R6r/3OOBp/mQ//4GgdoTujQjBAMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMB\nAf8EBTADAQH/MB0GA1UdDgQWBBSLgbX/8UA/yoBN8wNiLRLWe/oFITAKBggqhkjO\nPQQDAgNIADBFAiEAlIl9TndW/nTYJV5B6RYbHC3WaMgDTOiPZpVt5yuHdZkCICXk\nKtAH6jfINwW8WI8Yk5ELA4kxrpulaZu0791y8xlF\n-----END CERTIFICATE-----\n",
                "CertIssuerSubject": "MBMxETAPBgNVBAMTCHN3YXJtLWNh",
                "CertIssuerPublicKey": "MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEPitrw1c2p+18pF9XpzPgJp8skudIvluFNRQnTw54GPE9T2TRz2PYKceKZJXYQ6pX9Hqv/c44Gn+ZD//gaB2hOw=="
            }
        },
        "Status": {
            "State": "ready",
            "Addr": "192.168.102.139"
        }
    }
]
```

> 打上的標籤會在這顯示 ↓
> "Labels" : "env": "test"

vm3
```
root@vm1:~# docker node inspect vm3
[
    {
        "ID": "wcxi0azvsafuygzetrdxfo1ds",
        "Version": {
            "Index": 387
        },
        "CreatedAt": "2021-01-01T08:36:05.984421139Z",
        "UpdatedAt": "2021-01-02T06:53:14.174365255Z",
        "Spec": {
            "Labels": {
                "env": "prod"
            },
            "Role": "worker",
            "Availability": "active"
        },
        "Description": {
            "Hostname": "vm3",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
            },
            "Resources": {
                "NanoCPUs": 2000000000,
                "MemoryBytes": 4143312896
            },
            "Engine": {
                "EngineVersion": "19.03.13",
                "Plugins": [
                    {
                        "Type": "Log",
                        "Name": "awslogs"
                    },
                    {
                        "Type": "Log",
                        "Name": "fluentd"
                    },
                    {
                        "Type": "Log",
                        "Name": "gcplogs"
                    },
                    {
                        "Type": "Log",
                        "Name": "gelf"
                    },
                    {
                        "Type": "Log",
                        "Name": "journald"
                    },
                    {
                        "Type": "Log",
                        "Name": "json-file"
                    },
                    {
                        "Type": "Log",
                        "Name": "local"
                    },
                    {
                        "Type": "Log",
                        "Name": "logentries"
                    },
                    {
                        "Type": "Log",
                        "Name": "splunk"
                    },
                    {
                        "Type": "Log",
                        "Name": "syslog"
                    },
                    {
                        "Type": "Network",
                        "Name": "bridge"
                    },
                    {
                        "Type": "Network",
                        "Name": "host"
                    },
                    {
                        "Type": "Network",
                        "Name": "ipvlan"
                    },
                    {
                        "Type": "Network",
                        "Name": "macvlan"
                    },
                    {
                        "Type": "Network",
                        "Name": "null"
                    },
                    {
                        "Type": "Network",
                        "Name": "overlay"
                    },
                    {
                        "Type": "Volume",
                        "Name": "local"
                    }
                ]
            },
            "TLSInfo": {
                "TrustRoot": "-----BEGIN CERTIFICATE-----\nMIIBajCCARCgAwIBAgIUSFKIDtFHCjWqUNaUDwgyWjvCMnMwCgYIKoZIzj0EAwIw\nEzERMA8GA1UEAxMIc3dhcm0tY2EwHhcNMjEwMTAxMDgzMDAwWhcNNDAxMjI3MDgz\nMDAwWjATMREwDwYDVQQDEwhzd2FybS1jYTBZMBMGByqGSM49AgEGCCqGSM49AwEH\nA0IABD4ra8NXNqftfKRfV6cz4CafLJLnSL5bhTUUJ08OeBjxPU9k0c9j2CnHimSV\n2EOqV/R6r/3OOBp/mQ//4GgdoTujQjBAMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMB\nAf8EBTADAQH/MB0GA1UdDgQWBBSLgbX/8UA/yoBN8wNiLRLWe/oFITAKBggqhkjO\nPQQDAgNIADBFAiEAlIl9TndW/nTYJV5B6RYbHC3WaMgDTOiPZpVt5yuHdZkCICXk\nKtAH6jfINwW8WI8Yk5ELA4kxrpulaZu0791y8xlF\n-----END CERTIFICATE-----\n",
                "CertIssuerSubject": "MBMxETAPBgNVBAMTCHN3YXJtLWNh",
                "CertIssuerPublicKey": "MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEPitrw1c2p+18pF9XpzPgJp8skudIvluFNRQnTw54GPE9T2TRz2PYKceKZJXYQ6pX9Hqv/c44Gn+ZD//gaB2hOw=="
            }
        },
        "Status": {
            "State": "ready",
            "Addr": "192.168.102.145"
        }
    }
]
```

> 打上的標籤會在這顯示 ↓
> "Labels" : "env": "prod"



你也可以透過 visualizer 介面觀看

![](https://i.imgur.com/iCRKIYL.png)

---

建立一個httpd服務，並且設在test節點內
```
root@vm1:~# docker service create --constraint node.labels.env==test --replicas 2 --name web1 -p 8000:80 httpd
eaec22zl3ocachbpkezbe9rhw
overall progress: 2 out of 2 tasks 
1/2: running   
2/2: running   
verify: Service converged 
```

![](https://i.imgur.com/kGgvaIy.png)


將 test 標籤註解掉

```
root@vm1:~# docker service update --constraint-rm node.labels.env==test web1
web1
overall progress: 2 out of 2 tasks 
1/2: running   
2/2: running   
verify: Service converged 
```
![](https://i.imgur.com/l9kgXhj.png)


打上新的標籤 prod
```
root@vm1:~# docker service update --constraint-add node.labels.env==prod web1
web1
overall progress: 2 out of 2 tasks 
1/2: running   
2/2: running   
verify: Service converged 
```

![Imgur](https://imgur.com/dmUdd10)


移除節點標籤

```
docker node update --label-rm env vm2
docker node update --label-rm env vm3
```

![](https://i.imgur.com/G34YFZF.png)
