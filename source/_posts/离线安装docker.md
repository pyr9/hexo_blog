---
title: 离线安装docker
date: 2025-06-09 21:16:20
tags:
categories: docker
---

# 1. 下载Docker

找到一台可以联网的 CentOS 机器：用于下载 Docker 及其依赖。

地址：https://download.docker.com/linux/static/stable/x86_64/



# 2. 创建一个目录来存放需要的文件

```shell
mkdir docker-packages
cd docker-packages
```

# 3. 准备需要的脚本

## 3.1. 创建 systemd 服务文件docker.service

```shell
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
```

## 3.2.  一键安装脚本

install.sh

```shell
#!/bin/sh
 
echo '解压tar包'
tar -xvf $1
echo '将docker目录下所有文件复制到/usr/bin目录'
cp docker/* /usr/bin
echo '将docker.service 复制到/etc/systemd/system/目录'
cp docker.service /etc/systemd/system/
echo '添加文件可执行权限'
chmod +x /etc/systemd/system/docker.service
echo '重新加载配置文件'
systemctl daemon-reload
echo '启动docker'
systemctl start docker
echo '设置开机自启'
systemctl enable docker.service
echo 'docker安装成功'
docker -v
```

## 3.3.  一键卸载脚本

uninstall.sh:

```shell
#!/bin/sh
 
echo '停止docker'
systemctl stop docker
echo '删除docker.service'
rm -f /etc/systemd/system/docker.service
echo '删除docker文件'
rm -rf /usr/bin/docker*
echo '重新加载配置文件'
systemctl daemon-reload
echo '卸载成功'
```

# 4. 上传文件和安装包到服务器上

你可以通过 USB 设备、SCP、FTP 等方式将下载好的包及脚本传输到目标 CentOS 机器上。

![img](https://panyuro.oss-cn-beijing.aliyuncs.com/1749446823708-033483db-393b-4b2b-b37c-4d0c08c377ae.png)

# 5. 给脚本赋予执行权限

分别给install.sh和uninstall.sh赋予可执行权限。

```shell
chmod +x install.sh
chmod +x uninstall.sh
```

# 6. 执行安装

```shell
sh install.sh docker-26.1.4.tgz
```

# 7. 修改Docker默认镜像和容器存储位置

[Docker](https://so.csdn.net/so/search?q=Docker&spm=1001.2101.3001.7020) 默认安装的情况下，会使用 `/var/lib/docker/` 目录作为存储目录，用以存放拉取的镜像和创建的容器等。不过由于此目录一般都位于系统盘，遇到系统盘比较小，而镜像和容器多了后就容易磁盘空间不足

- 查看当前[docker](https://so.csdn.net/so/search?q=docker&spm=1001.2101.3001.7020)的默认存储目录 `docker info` ,默认为`/var/lib/docker/` 

- 查看磁盘空间，可以看到/data 目录最大，所以可以将地址设置为/data/docker

  ```
  [root@test_jenkins_for_mes workspace]# df -h
  Filesystem               Size  Used Avail Use% Mounted on
  devtmpfs                 3.9G     0  3.9G   0% /dev
  tmpfs                    3.9G     0  3.9G   0% /dev/shm
  tmpfs                    3.9G  297M  3.6G   8% /run
  tmpfs                    3.9G     0  3.9G   0% /sys/fs/cgroup
  /dev/mapper/centos-root   44G   39G  5.5G  88% /
  /dev/sda1               1014M  151M  864M  15% /boot
  /dev/mapper/datavg-data  200G   19G  182G  10% /data
  overlay                  200G   19G  182G  10% /data/docker/overlay2/18b7d8f10d4f7f788ff391c04494b0bcc5dd6eb0deafd3f89a0caffd4e00f45f/merged
  tmpfs                    783M     0  783M   0% /run/user/1001
  ```

  

- 编辑 `/etc/docker/daemon.json` 文件

  `sudo vim /etc/docker/daemon.json `

  ```
  {
    "data-root": "/data/docker"
  }
  ```

- 重启docker, 并查看 `docker info`

  ```shell
  systemctl restart docker
  ```

  