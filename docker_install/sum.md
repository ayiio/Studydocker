## docker网络
* 分类
  * 单机：Bridge Network
  * 单机：Host Network
  * 单机：None Network
  * 多机：Overlay Network
* 网络命名空间 namespace
  * docker底层重要概念，容器间隔离
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
