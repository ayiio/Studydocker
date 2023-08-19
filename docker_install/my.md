## docker在CentOS上安装
* 官方文档：https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository
* 云端docker：https://labs.play-with-docker.com/
* 更新yum：yum update
* 卸载旧的版本：
```
  yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```   

* 安装依赖：
```
yum install -y yum-utils \
      device-mapper-persistent-data \
      lvm2
```

* 添加repository
```
yum-config-manager \
  --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

* 安装和运行示例
```
列出docker版本：yum list docker-ce --showduplicates | sort -r
安装：yum -y install docker-ce-(version)
启动：systemctl start docker
开机启动： systemctl enable docker
查看docker：docker version
运行hello world：docker run hello-world
```

## docker概念和操作
* docker底层技术支持   
  * Namespaces：主要做网络隔离
  * Control groups：做资源限制，例如设置占用多少内存/CPU
  * Union file systems：image和container分层
* docker image   
  文件和meta data集合    
  查看本地所有image：`docker image ls`       
* 拉取新镜像并配置go环境   
  拉取镜像：`docker pull ubuntu:14.04`        
  安装go语言环境：下载go-linux.tar.gz，winSCP上传到/usr/local下，解压`tar -zxvf go.tar.gz`      
  配置环境变量：vi /etc/profile : `GOROOT=/usr/local/go   PATH=$PATH:$GOROOT/bin    export PATH`     
  使配置文件生效：`. /etc/profile`         
  `mkdir /home/hello-world`        
  `vi hello.go`   
* 创建docker file   
  vi Dockerfile
```
  # 从最简开始
  FROM scratch
  # 加到根目录
  ADD hello /
  # 运行hello
  CMD ["/hello"]
```
* 构建image   
  构建到本地image：`docker build -t 本地镜像名 .`    
  根据当前目录下的dockerfile构建image到仓库：`docker build -t docker仓库用户名/hello-world .`
  查看分层的详细信息： `docker history ImageID`
  运行image：`docker run ImageName`
     
  container变更后，可以使用docker commit构建新的image：`docker commit containerID docker仓库用户名/centos-hg .`   
```
  [Dockerfile]
  FROM centos
  RUN yum -y install lrzsz

  docker build -t docker仓库用户名/centos-lrzsz .
```
  
* 查看container
  查看所有，包括运行结束的：`docker container ls -a`
  查看所有运行中/运行过的container ID：`docker container ls -aq`  
  交互性运行： docker run -it 镜像名     
* 删除container运行记录  
  删除一个：`docker container rm containerID`
  删除多个：`docker container rm $(docker container ls -aq)`
  排除正在运行的container，删除操作：`docker container ls -f "status=exited" -q` 获取已经退出的containerID，`docker container rm $(docker container ls -f "status=exited" -q)`
* Dockerfile详解
  * FROM：文件的开始
```
FROM scratch  #从头开始制作一个最简的
FROM centos   #使用centos作为系统，如果没有则拉取
FROM centos:7.0 #指定系统+版本号
```
  * LABEL：相当于注释或说明信息
```
LABEL version="1.0" #1.0版本
LABEL descrip="xxx"
LABEL author="xxx"
```
  * RUN：执行命令，每执行一条RUN，会多一层
```
RUN yum -y update && yum -y install lrzsz \
   net-tools
```
  * WORKDIR：进入或创建目录
```
WORKDIR /root  #进入/root目录
WORKDIR /test  #自动创建目录
WORKDIR demo   #创建/test/demo目录
RUN pwd        #/test/demo
```
  * ADD and COPY：将本地文件添加到镜像里，ADD可以解压缩文件
```
ADD hello / #将hello加到镜像根目录
ADD xxx tar.ge / #添加并解压到根目录

WORKDIR /root/test
COPY hello .  # /root/test/hello
```
  * ENV：指定版本
```
ENV MYSQL_VERSION 5.6  #设置常量
RUN apt-get -y install mysql-server="${MYSQL_VERSION}"
```
  * CMD and ENTRYPOINT：执行命令
```
-----#1.Shell和Exec格式：
#Shell
RUN apt-get -y install lrzsz
CMD echo "hello.docker"
ENTRYPOINT echo "hello.docker"

#Exec
RUN ["atp-get", "-y", "install", "lrzsz"]

#Shell格式Dockerfile
FROM centos
ENV name Docker
ENTRYPOINT echo "hello $name"

docker build -t centos-entry-shell .
docker image ls
docker run centos-entry-shell

#Exec格式Dockerfile
FROM centos
ENV name Docker
ENTRYPOINT ["bin/bash", "-c", "echo hello $name"]
# ENTRYPOINT ["bin/echo", "hello $name"]

docker build -t centos-entry-exec .
docker image ls
docker run centos-entry-exec

-----#2.CMD and ENTRYPOINT
如果docker指定了其他命令，CMD会被忽略
如果定义多个CMD，只会执行最后一个CMD

#CMD正常示例
FROM centos
ENV name Docker
CMD echo "hello $name"

docker build -t centos-cmd1 .
docker run centos-cmd1

#CMD被忽略示例
FROM centos
ENV name Docker
ENV name2 Docker2
ENTRYPOINT echo "hello $name"
CMD echo "hello $name2"

docker build -t centos-cmd2 .
docker run centos-cmd2

#CMD多个只执行最后一个
FROM centos
ENV name Docker
ENV name2 Docker2
CMD echo "hello $name"
CMD echo "hello $name2"

docker build -t centos-cmd3 .
docker run centos-cmd3
```
* 分享docker image
  image名一定要以自己docker hub的用户名开头， hub.docker.com     

```
  在docker官网注册账户后
  docker image ls
  docker login
  docker image push username/imagexxx  #前缀必须是账号名
  docker pull username/imagexxx #从docker hub拉取镜像到本地
```
* 分享Dockerfile
  分享Dockerfile更安全
* 搭建私有docker registry
```
  https://hub.docker.com/_/registry :
  私有docker库物理机：
  docker run -d -p 5000:5000 --restart always --name registry registry:2

  打包image:
  docker build -t 私有docker库ip:5000/imagexxx .

  本机docker.daemon.json添加docker库地址：
  vim /etc/docker/daemon.json
  {
    "insecure-registries": ["私有docker库ip:5000"]
  }
  sudo systemctl daemon-reload
  sudo systemctl restart docker

  发送image到本地仓库：
  docker push 私有docker库ip:5000/imagexxx

  拉取本地仓库的image：
  docker pull 私有docker库ip:5000/imagexxx

  chrome: http://私有docker库ip:5000/v2/_catalog
```
* docker源更改
```
vi /etc/docker/daemon.json

{
  "registry-mirrors": ["https://阿里云个人镜像加速器.mirror.aliyuncs.com"]
}

sudo systemctl daemon-reload
sudo systemctl restart docker
```
## 构建
* 下载pip
  `yum -y install wget`   
  `wget https://bootstrap.pypa.io/get-pip.py --no-check-certificate` 或 `wget https://bootstrap.pypa.io/pip/2.7/get-pip.py`
  `pip install flask`
* 编写简单的python web app服务
```
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
        return "hello py_docker"
if __name__ == '__main__':
        app.run(host="app运行地址", port=端口)
```
* 运行服务并访问
  `python app.py`，并用物理机浏览器访问`app运行地址:端口`
  
  
