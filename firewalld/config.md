## firewalld基本使用
* 启动：`systemctl start firewalld`
* 查看状态：`systemctl status firewalld` / `firewall-cmd --state`
* 停止：`systemctl disable firewalld`
* 禁用：`systemctl stop firewalld`
## 端口操作
* 添加：`firewall-cmd --zone=public --add-port=5005/tcp --permanent` # permanent表示永久生效
* 临时添加外部访问权限：`firewall-cmd --add-port=5005/tcp`
* 重新载入/更新防火墙规则：`firewall-cmd --reload`
* 查看指定端口：`firewall-cmd --zone=public --query-port=80/tcp`
* 删除：`firewall-cmd --zone=public --remove-port=80/tcp`
* 查看开启了哪些服务：`firewall-cmd --list-services`，每个服务对应了/usr/lib/firewalld/services下的xml文件
* 查看开启了哪些端口：`firewall-cmd --list-ports` / `firewall-cmd --zone=public --list-ports`
* 查看还可以开启哪些服务：`firewall-cmd --get-services`
* 查看区域信息：`firewall-cmd --get-active-zones`
* 查看指定接口所属区域：`firewall-cmd --get-zone-of-interface=eth0`
* 拒绝所有包：`firewall-cmd --panic-on`
* 取消拒绝状态：`firewall-cmd --panic-off`
* 查看拒绝状态：`firewall-cmd --query-panic`
## 服务操作
* 启动服务：`systemctl start firewalld.service`
* 关闭服务：`systemctl stop firewalld.service`
* 重启服务：`systemctl restart firewalld.service`
* 服务状态：`systemctl status firewalld.service`
* 查看已启动的服务列表：`systemctl list-unit-files|grep enabled`
* 查看启动失败的服务列表：`systemctl --failed`
