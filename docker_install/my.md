## docker在CentOS上安装
* 官方文档：https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository
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
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
* 安装docker-machine
