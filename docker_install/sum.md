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
  flask做web服务，redis做自增（同一机器）
命令集示例
```
docker run -d --name redis redis  #运行自定义名为redis的redis容器

mkdir /home/flask-redis
cd flask-redis
vi app.py   #创建flask web应用
    from flask import Flask
    from redis import Redis
    import os
    import socket

    app=Flask(__name__)
    redis=Redis(host=os.environ.get('REDIS_HOST', '127.0.0.1'), port=6379)

    @app.route('/')
    def hello():
        redis.incr('hits')
        return 'Hello Container World! I have been seen %s times and my hostname is %s.\n' % (redis.get('hits'), sockert.gethostname())

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=5000, debug=True)

vi Dockerfile   #创建Dockerfile
    FROM python:2.7
    LABEL maintaner="xxx@github.com"
    COPY . /app
    WORKDIR /app
    RUN pip install flask redis
    EXPOSE 5000
    CMD ["python", "app.py"]

docker build -t xxx/flask-redis .  #构建image
docker image ls #查看已经构建的image
docker run -d --link redis --name flask-redis -e REDIS_HOST=redis xxx/flask-redis #以link redis的方式运行xxx/flask-redis
docker ps #查看已运行的docker进程
docker exec -it flask-redis /bin/sh  #交互式进入发布服务的容器
env  #查看容器环境，可以查到REDIS_HOST=redis
ping redis #可以ping关联的redis容器
curl 127.0.0.1:5000 #在本地访问web服务
exit #退出容器

curl 127.0.0.1:5000 #容器外部无法访问web 容器中的web服务
docker stop flask-redis  #停止容器
docker run -d --link redis -p 5000:5000 --name flask-redis2 -e REDIS_HOST=redis xxx/flask-redis  #映射本地端口

docker stop $(docker container ls -aq)  #停止容器
docker rm $(docker container ls -aq) #删除已停容器
```
* 多机器多容器通信
  一台做redis，一台做web处理
  overlay网络和etcd通信  

命令集示例
```
多机情况下，要保证IP不冲突，需要记录所有IP，使用分布式存储记录

etcd tar包放到/usr/local下
tar -zxvf etcd-vxx.tar.gz #解压etcd包
cd etcd-vxx

#etcd启动命令(node1),需要修改ip-192.168.174.141为主机ip，192.168.174.142为另一主机IP
nohup ./etcd --name dcker-node1 --initial-advertise-peer-urls http://192.168.174.141:2380 \
  --listen-peer-urls http://192.168.174.141:2380 \
  --listen-client-urls http://192.168.174.141:2379, http://127.0.0.1:2379 \
  --advertise-client-urls http://192.168.174.141:2379 \
  --initial-cluster-token etcd-cluster \
  --initial-cluster docker-node1=http://192.168.174.141:2380,docker-node2=http://192.168.174.142:2380 \
  --initial-cluster-state new&
#启动后查看状态
./etcdctl cluster-health

#etcd启动命令(node2),需要修改ip-192.168.174.142为主机ip，192.168.174.141为另一主机IP
nohup ./etcd --name dcker-node1 --initial-advertise-peer-urls http://192.168.174.142:2380 \
  --listen-peer-urls http://192.168.174.142:2380 \
  --listen-client-urls http://192.168.174.142:2379, http://127.0.0.1:2379 \
  --advertise-client-urls http://192.168.174.142:2379 \
  --initial-cluster-token etcd-cluster \
  --initial-cluster docker-node1=http://192.168.174.141:2380,docker-node2=http://192.168.174.142:2380 \
  --initial-cluster-state new&
#启动后查看状态
./etcdctl cluster-health

#停止node1 docker做一些配置
service docker stop

#docker启动命令(node1)，修改ip-192.168.174.141为主机IP
/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://192.168.174.141:2379 --cluster-advertise=192.168.174.141:2375&

#查看node1 docker网络状态
docker network create -d overlay demo #创建名为demo的overlay集群网络
docker network ls #查看网络状况

doker run -d --name test7 --net demo busybox sh -c "while true; sleep 3600; done"  #node1运行名为test7的busybox容器

-----

#停止node2 docker做一些服务
service docker stop

#docker启动命令(node2)
/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://192.168.174.142:2379 --cluster-advertise=192.168.174.142:2375&

#查看node2 docker网络状态，已经有名为demo的overlay网络
docker network ls

#node2中运行相同命名的容器时会提示已经存在
doker run -d --name test7 --net demo busybox sh -c "while true; sleep 3600; done"

 #node2运行名为test8的busybox容器
doker run -d --name test8 --net demo busybox sh -c "while true; sleep 3600; done"
docker exec -it test8 #进入容器test8
ip a #查看网络配置和node1相互错开
```
## docker持久化存储和数据共享
* 数据持久化方案
  * 基于本地文件系统的volume
  * 基于plugin的volume

* volume类型
  * 受管理的data volume，由docker后台自动创建
  * 绑定挂载的volume，具体挂载位置可以由用户指定

* 数据持久化 - data volume
  * https://hub.docker.com/搜索mysql，  可以看到官方的Dockerfile中也定义了VOLUME

mysql镜像，命令集示例
```
docker stop $(docker container ls -aq)  #停止所有容器
docker rm $(docker container ls -aq)  #删除所有已停止的容器
docker run -d --name mysql1 -e MYSQL_ALLOW_EMPTY_PASSWORD=yes mysql #启动无密码的mysql容器
docker ps  #查看已运行的mysql image进程
docker volume ls #查看生成的volume
docker volume inspect volumeNameID  #查看volume存放目录和创建时间
docker stop mysql1  #停掉mysql容器
docker rm mysql1 #删除mysql容器，容器删除后，volume仍存在
docker volume prune  #删除不再引用的空间

docker run -d -v mysql:/var/lib/mysql --name mysql1 -e MYSQL_ALLOW_EMPTY_PASSWORD=yes mysql #启动无密码且指定文件目录名的mysql容器
docker exec -it mysql /bin/bash #进入mysql container，
mysql -uroot  #无密码登录mysql命令行, create database test1; 创建新库后退出
docker stop mysql1  #停掉mysql容器
docker rm mysql1  #删除mysql容器
docker run -d -v mysql:/var/lib/mysql --name mysql1 -e MYSQL_ALLOW_EMPTY_PASSWORD=yes mysql #再次生成无密码且指定相同文件目录名的mysql容器，test1库会自行读入，实现了数据的持久化，后续创建指定到同一位置，会自动读取原数据
```
* 数据持久化 - bind mouting
  * 可以指定一个与容器同步的目录，容器变化，文件同步变化

命令集示例
```
mkdir /home/nginx
cd /home/nginx
vi index.html  #在/home/nginx下创建一个简单的欢迎页
    <h1>Hello docker</h1>

vi Dockerfile  #在/home/nginx下创建Dockerfile
FROM nginx:latest
WORKDIR /usr/share/nginx/html
COPY index.html index.html

docker build -t nginx .  #编译nginx镜像
docker image ls #查看nginx image
docker run -d -p 80:80 --name web nginx  #启动nginx image，映射容器80端口到本机80端口
curl 127.0.0.1  #将显示被/home/nginx/index.html替换过的内容

docker stop web  #停掉web容器，使用bind monting
docker rm web  #删除web容器
docker run -d -v /home/nginx:/usr/share/nginx/html -p 80:80 --name web nginx  #映射容器内/usr/share/nginx/html目录到本地/home/nginx目录
docker exec -it web /bin/bash  #进入容器，touch test.txt创建test.txt后退出
ls -ltr #本地查看，也会有test.txt文件，实现了容器内数据/文件的本地持久化
```
## docker Compose多容器部署
* docker部署wordpress
  * wordpress是一个博客网站

命令集示例
```
docker pull wordpress #拉取wordpress镜像
docker pull mysql:5.5  #使用老版本避免冲突，但wordpress默认读取mysql latest版本，需要手动指定mysql:5.5为latest来解决wordpress读取mysql错误的问题
docker tag mysql:5.5 mysql:latest #标记mysql:5.5为latest版本

docker run -d name mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=yes mysql #运行mysql容器
docker run -d -e WORKPRESS_DB_HOST=mysql:3306 --like mysql -p 8080:80 wordpress #映射容器80端口为本地8080端口，并连接mysql容器，外部浏览器可以通过ip:8080访问wordpress容器
```
* Compose介绍
  * docker compose类似批处理方式，简化了多容器APP的部署和管理

* docker compose安装和使用
  * 下载

命令集示例
```
    curl -L http://github.com/docker/compose/release/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

```
  * 安装后需要赋权限
  * 用docker-compose的方式部署wordpress

* 容器扩展和负载均衡

* 复杂应用部署

## 容器编排 docker swarm
* 编排swarm简介

* 三节点swarm集群搭建

* 创建维护service并扩展
