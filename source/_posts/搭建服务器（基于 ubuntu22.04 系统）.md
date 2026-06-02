---
title: 搭建服务器（基于 ubuntu22.04 系统）
date: 2024-04-29 10:03
categories:
  - 学习
---

### nginx：

更新软件源：
```shell
sudo apt update
```

安装nginx：
```shell
sudo apt install nginx
```

### docker:
1：更新软件源：
```shell
sudo apt update
```
2：安装基本软件
```shell
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common lrzsz -y
```
3：指定使用阿里云镜像
```shell
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```
4：更新软件源
```shell
sudo apt update
```
5：安装docker-ce
```shell
sudo apt-get install docker-ce -y
```
6：经过上面的步骤，已经完成了docker的安装，可以通过以下命令查看docker的版本。
```shell
docker version
```
7：更换镜像源：
修改 docker daemon.json 文件
```shell
vim /etc/docker/daemon.json
```
将其中内容修改为：
```shell
{
    "registry-mirrors": [
        "https://docker.1panel.live",
        "https://hub.rat.dev"
    ]
}
```
重启docker
```shell
systemctl daemon-reload
systemctl restart docker
```