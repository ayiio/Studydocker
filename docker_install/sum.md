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
ip link add veth-test1 type veth peer veth-test2
ip link
ip link set veth-test1 netns test1 #设置veth对到tes1命名空间
ip netns exec test1 ip link #test1命名空间状态
ip link set veth0 netns test2 #设置veth对到test2命名空间
ip netns exec test2 ip link
ip netns exec test1 ip addr add 本机网段.1.1/端口 dev veth-test1
ip netns exec test2 ip addr add 本机网段.1.2/端口 dev veth0
ip netns exec test1 ip link set dev veth-test1 up #启动veth对1
ip netns exec test2 ip link set dev veth0 up #启动veth对2
ip netns exec test1 ip a #查看test1网络状态

```
* Bridge
  多容器通信
  容器通信有时并不知道请求的IP地址
* 端口映射
  外界访问
* None和Host
  none应用场景：安全性要求极高，存储绝密数据等
  host网络类似于NAT
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
