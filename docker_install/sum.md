## docker网络
* 分类
  * 单机-桥接：Bridge Network
  * 单机-宿主：Host Network
  * 单机-无：None Network
  * 多机：Overlay Network
* 网络命名空间 namespace
  * docker底层重要概念，容器间隔离
```
docker run -d --name test1 busybox /bin/sh -c "while true; do sleep 3600; done"
docker run -d --name test2 busybox /bin/sh -c "while treu; do sleep 3600; done"

#交互
vm1: docker exec -it test1 /bin/sh  -> ifconfig/ ip a，默认image中的eth0配置为172.17.0.2
vm2: docker exec -it test2 /bin/sh  -> ifconfig/ ip a，默认image中的eth0配置为172.17.0.3
本机：ifconfig(yum search ifconfig->yum -y install net-workXXX)，默认image中的docker0配置为172.17.0.1，eth0配置为192.168.x.x
通过veth pair与物理机交互
![image](https://github.com/ayiio/studyDocker/assets/61615400/881f45b2-d096-4fb0-8aa0-8350bb777992)

# ip netns
ip netns list
ip netns add imageXXX #创建imageXXX 命名空间
ip netns delete imageXXX
ip netns exec imageXXX ip a #查看imageXXX 命名空间ip配置
ip netns exec imageXXX ip link set dev lo up

ip link add veth-imageXXX1 type veth peer veth-imageXXX2  #指定veth对

#本机ip link
ip link
```
命令集示例：
```
#test1/2 image
docker exec -it test1 /bin/sh
docker exec -it test2 /bin/sh
docker ps
docker exec stop $(docker contanier ls -aq)

#创建test1/2的命名空间
ip netns list #查看已创建的命名空间
ip netns add test1  #创建test1命名空间
ip netns list #查看已创建的命名空间
ip netns exec test1 ip a #查看test1命名空间ip状态
ip netns exec test1 ip link set dev lo up #启动test1命名空间ip
ip netns exec test1 ip a #启动后查看test1命名空间的ip状态
ip netns add test2  #创建test2命名空间
ip netns list #查看已创建的命名空间

#link两个命名空间
ip link add veth-test1 type veth peer veth-test2 #添加veth对
ip link #本机ip link
ip link set veth-test1 netns test1 #设置veth对到tes1命名空间
ip netns exec test1 ip link #test1命名空间状态
ip link set veth0 netns test2 #设置veth对到test2命名空间
ip netns exec test2 ip link
ip netns exec test1 ip addr add 本机网段.1.1/端口 dev veth-test1  #为veth对分配ip
ip netns exec test2 ip addr add 本机网段.1.2/端口 dev veth0  #为veth对分配ip
ip netns exec test1 ip link set dev veth-test1 up #启动veth对1
ip netns exec test2 ip link set dev veth0 up #启动veth对2
ip netns exec test1 ip a #查看test1网络状态
```
* Bridge
  多容器通信    
  容器通信有时并不知道请求的IP地址，可以通过主机名进行访问
```
cat /etc/hostname #主机名
```
  
  命令集示例
```
docker start test1 #启动test1容器
docker ps  #查看已启动的容器
docker network ls #查看本机docker网络
docker network inspect NetworkId #查看网络状态
ip a #本机ip信息，存在veth对与容器中的veth配对
docker exec test1 ip a #查看容器test1的网络状态

yum -y install bridge-utils #查看网络方式工具
brctl show #查看bridge内容

docker stop test1 #停止tes1容器

docker run -d --name test3 busybox /bin/sh -c "while true; do sleep 3600; done;"  #启动一直运行的容器
docker run -d --name test4 --link test3 busybox /bin/sh -c "while true; do sleep 3600; done;" #启动关联的容器
docker ps #查看启动的容器状态
docker exec -it test4 /bin/sh  #登录test4容器
ip a #查看test4容器内ip配置
ping test3:3306 #在test4容器内ping test3容器的3306端口，通过主机名进行访问
```
* 端口映射
  外界访问

  命令集示例
```
docker run --name web -d nginx  #启动命名为web的nginx容器
docker network inspect bridge #查看本机bridge配置信息，container ip 172.17.0.2
telnet 172.17.0.2 80 #nginx容器的80端口，quit退出
curl http://172.17.0.2 #访问nginx容器的欢迎页面，本机docker0网卡可以成功访问，外部浏览器无法访问
docker stop web #停掉web容器
docker run --name web2 -d -p 80:80 nginx  #本机和容器映射相同端口，通过eno网卡，外部可以访问到docker容器
```
* None和Host
  none应用场景：安全性要求极高，存储绝密数据等
  host网络类似于NAT

  命令集示例
```
docker run -d --name test5 --network none busybox /bin/sh -c "while true; do sleep 3600; done"  #启动一个none类型网络的容器
docker network inspect none #查看本机network none网络信息，含有test5
docker exec -it test5 /bin/sh #进入test5容器，ip a命令可以看到只有一个127.0.0.1的网络配置

docker run -d --name test6 --network host busybox /bin/sh -c "while true; do sleep 3600; done"  #启动一个host类型网络的容器
docker network inspect host #查看本机network host网络信息，含有test6
docker exec -it test6 /bin/sh #进入test6容器，ip a命令可以看到只有一个与宿主机相通的网络配置
```
* 多容器部署和应用
  flask做web服务，redis做自增
* 多机器多容器通信
  一台做redis，一台做web处理
## docker持久化存储和数据共享
* 数据持久化方案

* volume类型

* 数据持久化 - data volume

* 数据持久化 - bind mouting

## docker Compose多容器部署
* docker部署wordpress

* Compose介绍

* docker compose安装和使用

* 容器扩展和负载均衡

* 复杂应用部署

## 容器编排 docker swarm
* 编排swarm简介

* 三节点swarm集群搭建

* 创建维护service并扩展
