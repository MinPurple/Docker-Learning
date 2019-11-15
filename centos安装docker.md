## 1 使用 Docker 仓库进行安装

* 设置仓库

```sh
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

* 安装 Docker Engine-Community

```sh
$ yum update && yum install docker-ce-18.06.2.ce
```

* 运行`docker --version`,可以看到安装的版本：

```sh
$ docker --version
Docker version 18.03.1-ce, build 9ee9f40
```

* 配置

```sh
$ mkdir /etc/docker
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

$ mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
$ systemctl daemon-reload
$ systemctl restart docker
```

* 启动Docker

```sh
# 启动Docker服务并激活开机启动：
$ systemctl start docker & systemctl enable docker
```

运行一条命令验证一下：

```sh
$ docker run hello-world
```