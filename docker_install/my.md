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

* 安装docker-machine
```
base=https://github.com/docker/machine/releases/download/v0.14.0 &&
curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine &&
sudo install /tmp/docker-machine /usr/local/bin/docker-machine
```   

* 下载pip

* docker底层技术支持   
  * Namespaces：主要做网络隔离
  * Control groups：做资源限制，例如设置占用多少内存/CPU
  * Union file systems：image和container分层
* docker image   
  文件和meta data集合    
  查看本地所有image：docker image ls    
* 拉取新镜像并配置go环境   
  拉取镜像：docker pull ubuntu:14.04     
  安装go语言环境：下载go-linux.tar.gz，winSCP上传到/usr/local下，解压tar -zxvf go.tar.gz     
  配置环境变量：vi /etc/profile : GOROOT=/usr/local/go   PATH=$PATH:$GOROOT/bin    export PATH    
  使配置文件生效：. /etc/profile     
  mkdir /home/hello-world     
  vi hello.go
* 创建docker file   
  vi Dockerfile
  ```
  # 从最简开始
  FROM scratch
  # 加到根目录
  ADD hell /
  # 运行hello
  CMD ["/hello"]
  ```
* 构建image   
  docker build -t 本地镜像名 
