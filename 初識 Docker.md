# 雲端通訊整合實務(9/15)
###### tags: `docker`、`docker-compose`
# 上課所需
> VM1:
> NAT(enp0s3)
> Host-Only(enp0s8)
> 內部網路(enp0s9)

> VM2:
> 內部網路(enp0s3)


# 實驗環境
![](https://i.imgur.com/9ceGHOQ.png)

# VM1設定

ip(內部網路) 設置
```
ip addr add 192.168.1.1/24 brd + dev enp0s9
```

將第一台vm設置成router
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```


> 查看vm是否已設置成router可使用：
```
cat /proc/sys/net/ipv4/ip_forward
```

**iptables的設定需要先配置好第二台VM，請先將第二台設定好在接著步驟。**

# VM2設定

ip(內部網路) 設置
```
ip addr add 192.168.1.2/24 brd + dev enp0s3
```

ip route 路由規則設定
```
ip route add default via 192.168.1.1
```
> 查看路由規則表可使用：
```
ip route show
```


# 測試1
讓VM2透過VM1的網卡連到外網的前置設定
```
iptables -A FORWARD -o enp0s3 -i enp0s9 -s 192.168.1.0/24 -m conntrack --ctstate NEW -j ACCEPT
```

```
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

```
iptables -t nat -A POSTROUTING -o enp0s3 -s 192.168.1.0/24 -j MASQUERADE
```

**做完以上步驟後可以先測試以下幾點來確認上面步驟可否成功:**
1. VM2 ping VM1
2. VM1 ping VM2
3. VM2 ping 外網

# Docker 設定

**請在VM1環境安裝docker、docker-compose**

安裝docker可參考：https://docs.docker.com/engine/install/centos/
安裝docker-compose可參考：https://docs.docker.com/compose/install/

安裝完請開啟docker
```
systemctl start docker
```

dockerhub下載adguard官方鏡像，可參考：
https://hub.docker.com/r/adguard/adguardhome
```
docker pull adguard/adguardhome
```
> 若docker pull 不成功的話，原因是需要先登入dockerhub才可下載，因此可先使用：

```
docker login:[帳號]、[密碼]
```

接著創建指定的路徑，以便adguard來存放data

```
mkdir -p /my/own/workdir
mkdir -p /my/own/confdir
```

啟動adguard鏡像

```
docker run --name adguardhome -v /my/own/workdir:/opt/adguardhome/work -v /my/own/confdir:/opt/adguardhome/conf -p 53:53/tcp -p 53:53/udp -p 67:67/udp -p 68:68/tcp -p 68:68/udp -p 80:80/tcp -p 443:443/tcp -p 853:853/tcp -p 3000:3000/tcp -d adguard/adguardhome
```

> 若出現埠號佔用的報錯，例如:68埠、67埠、53埠，請先使用`docker ps -a`，將啟動的adguard容器砍掉，**刪除容器**可使用`docker rm -f [容器名稱]`，再執行：

查看埠號資訊
```
netstat -tunlp | grep 68
netstat -tunlp | grep 67
netstat -tunlp | grep 53
```
刪除佔用的埠
```
kill -9 [68埠的ID]
kill -9 [67埠的ID]
kill -9 [53埠的ID]
```

# 測試2

1. 測試adguard有無成功，請在網頁上測試：`VM1的本地ip:3000`

成功後會出現以下畫面：
![](https://i.imgur.com/Mo0CTVC.png)

2. 設定完成後，測試VM2是否可阻擋廣告

成功後會出現以下畫面：
![](https://i.imgur.com/iM3muqg.jpg)

# 參考文獻
[上課影片](https://drive.google.com/drive/folders/1KAFgGqWMBBATryLE9tJwxxVP77FfxHCA?usp=sharing)