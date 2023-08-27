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
curl -L http://github.com/docker/compose/release/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose  #获取docker-compose
```
  * 安装后需要赋权限
```
chmod +x docker-compose  #添加执行权限
docker-compose --version  #任意目录查看工具是否可用
```
  * 用docker-compose的方式部署wordpress
```
mkdir /home/wordpress
cd /home/wordpress
vi docker-compose.yml

version: '3'  #使用版本3的docker-compose

services:  #类似container
  wordpress:  #类似container名
    image: wordpress  #镜像
    ports:  #端口映射
      - 80:80
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD: admin
    networks:
      - my-bridge   #和mysql在一个网段
  mysql:
    image: mysql:5.5
    environment:
      MYSQL_ROOT_PASSWORD: admin
      MYSQL_DATABASE: wordpress
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - my-bridge

volumes:
  mysql-data:

networks:
  my-bridge:
    driver: bridge  #指定的网络类型
```
编辑完docker-compose.yml文件后使用`docker-compose up`执行，或`docker-compose -f docker-compose2.yml up`指定对应的yml文件。     
执行后通过`docker ps`可以查看对应已启用的container，使用`ctrl+c`停止，停过后可以使用`docker ps -a`找到记录。  
使用`docker-compose down`停止对应的container并删除。     
使用`docker-compose -f docker-compose.yml up` + `docker-compose exec mysql bash`/`docker-compose exec wordpress bash`交互式运行


docker-compose结合Dockerfile使用的命令集示例
```
以/home/fask-redis/app.py + /home/fask-redis/Dockerfile为示例
同级目录下创建vi docker-compose.yml

version: '3'

services: 
  redis: 
    image: redis
  web: 
    build: 
      context: .
      dockerfile: Dockerfile
      ports: 
        - 8080:5000
      environment: 
        REDIS_HOST: redis
```
编写docker-compose后启动,`docker-compose -f docker-compose.yml up`   

* 容器扩展和负载均衡
  * 容器扩展  scale
命令集示例
```
#docker-compose.yml中将ports信息删除，扩展时使其自动分配端口
version: '3'

services: 
  redis: 
    image: redis
  web: 
    build: 
      context: .
      dockerfile: Dockerfile
      environment: 
        REDIS_HOST: redis

#使用scale扩展，容器内的5000端口可以自动映射，另需要负载均衡器配置访问
docker-compose up --scale web=3 -d
```
  * 负载均衡
`vi /home/flask-redis/app.py`修改`app.run(host="0.0.0.0", port=80, debug=True)`使其默认跑在80端口
`vi /home/flask-redis/Dockerfile`修改`EXPOSE 80`
`vi /home/flask-redis/docker-compose.yml`添加负载均衡器`dokercloud/haproxy`，配置和redis镜像同级缩进    
```
lb:
  image: dockercloud/haproxy
  ports:
    - 8080:80
  links:
    - web
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```
配置好后`docker-compose up`运行
可以运行时动态扩展：`docker-compose up --scale web=3 -d`，轮询执行    
可以运行时动态缩容：`docker-compose up --scale web=1 -d`，会删除多余容器

* 复杂应用部署
![image](https://github.com/ayiio/studyDocker/assets/61615400/8172abb7-9421-463e-92a7-325c52768e71)

总docker-compose.yml示例
```
front-tier表示外部可访问，back-tier表示内部可访问

version: "3"

services:
  voting-app:
    build: ./voting-app/.
    volumes:
      - ./voting-app:/app
    ports:
      - "5000:80"
    links:
      - redis
    networks:
      - front-tier
      - back-tier

  result-app:
    build: ./result-app/.
    volumes:
      - ./result-app:/app
    ports:
      - "5001:80"
    links:
      - db
    networks:
      - front-tier
      - back-tier

  worker:
    build: ./worker
    links:
      - db
      - redis
    networks:
      - back-tier

  redis:
    image: redis
    ports: ["6379"]
    networks:
      - back-tier

  db:
    image: postgres:9.4
    volumes:
      - "db-data:/var/lib/postgresql/data"
    networks:
      - back-tier

volumes:
  db-data:

networks:
  front-tier:
  back-tier:
```

woker模块，Dockerfile示例
```
目录层级：
worker:
    src:
    Dockerfile
    pom.xml

FROM java:7

RUN apt-get update -qq && apt-get install -y maven && apt-get clean

WORKDIR /code

ADD pom.xml /code/pom.xml
RUN ["mvn", "dependency:resolve"]
RUN ["mvn", "verify"]

# Adding source, compile and package into a fat jar
ADD src /code/src
RUN ["mvn", "package"]

CMD ["/usr/lib/jvm/java-7-openjdk-amd64/bin/java", "-jar", "target/worker-jar-with-dependencies.jar"]
```

voting-app模块，Dockerfile示例
```
FROM python:2.7

WORKDIR /app

ADD requirements.txt /app/requirements.txt
RUN pip install -r requirements.txt

# copy code from current floder to /app inside the container
ADD . /app

# make port 5000 available for links and/or publish
EXPOSE 80

CMD ["python", "app.py"]
```
## 容器编排 docker swarm
* 编排swarm简介    
![image](https://github.com/ayiio/studyDocker/assets/61615400/ef3b363c-0ec8-4eff-8229-99c2339b6bb8)


* 服务创建和调度
* 在swarm manage做决策，决定将worker部署到哪里    
![image](https://github.com/ayiio/studyDocker/assets/61615400/f1c3ab63-9097-4f0b-90f7-617e18f12f8f)

* 三节点swarm集群搭建
示例：3台主机，ip分别为192.168.16.65/66/67，示例将65作为主机，66和67作为从机
代码集示例：
```
#主机初始化
docker swarm init --advertise-addr=192.168.16.65  #swarm集群初始化    

#从机分别加入swarm
docker swarm join --token XXXXXXXXXXXXX-XXXXXXXXX  192.168.16.65:2377  #会提示This node joined a swarm as a worker

#从机退出集群
docker swarm leave -f  #从机退出集群，提示node leave the swarm

#主节点查看集群情况
docker node ls  #主机，从机退出，状态会标识为Down
```
* 创建维护Service并扩展
代码集示例：
```
docker service create --name demo busybox sh -c "while true; do sleep 3600; done"  #集群创建容器
docker service ls  #查看集群中的容器
docker ps #查看容器运行在哪个节点

#扩容
docker service scale demo=5  #使用scale进行扩容
docker service ls  #REPLICAS提示5/5，共5个实例，根据集群中各个节点状态自行分配

#强制关停某一节点上的容器
docker rm -f containerID

#主机中使用docker service ls查看存在短暂REPLICAS=4/5的状态，并迅速恢复为5/5
docker service ls
```
* 通过DockerStack部署Voting app
  * 多机集群方式部署应用
命令集示例：
```
docker stop $(docker container ls -aq) #停掉docker容器
docekr rm $(docker container ls -aq)   #删除已有的容器，需要先删除子node上的容器再删除主机上的

docker swarm swarm leave -f #各个节点各自leave后，不再重新创建新的集群环境

mkdir /
#编辑新的docker-compose.yml
version: "3"
services:
  redis:
    image: redis:alpine
    port:
      - "6379"
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deply:
      placement:
        constraints: [node.role == manager]

  vote:
    image: dockersample/examplevotingapp_vote:before
    ports:
      - 5000:80
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure

  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - 5001:80
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersample/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

networks:
  frontend
  backend

volumes:
  db-data:
```
* 使用DockerStack部署可视化应用

* 使用并管理DockerSecret


