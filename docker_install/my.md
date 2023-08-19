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

