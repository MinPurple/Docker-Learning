# 1 Docker Machine

`Docker Machine` 是 Docker 官方编排（`Orchestration`）项目之一，负责在多种平台上快速安装 `Docker` 环境。

`Docker Machine` 项目基于 `Go` 语言实现，目前在 [Github](https://github.com/docker/machine) 上进行维护。

这里将介绍 `Docker Machine` 的安装及使用。

## 1.1 安装

Docker Machine 可以在多种操作系统平台上安装，包括 Linux、macOS，以及 Windows。

* macOS、Windows
    
    `Docker Desktop for Mac/Windows` 自带 `docker-machine` 二进制包，安装之后即可使用。

    查看版本信息。

    ```sh
    $ docker-machine -v
    docker-machine version 0.16.2, build bd45ab13
    ```

* Linux
    
    在 `Linux` 上的也安装十分简单，从 官方 [GitHub Release](https://github.com/docker/machine/releases) 处直接下载编译好的二进制文件即可。

    例如，在 Linux 64 位系统上直接下载对应的二进制包。

    ```sh
    $ curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
    chmod +x /tmp/docker-machine &&
    sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
    ```

    完成后，查看版本信息。

    ```sh
    $ docker-machine -v
    docker-machine version 0.16.2, build bd45ab13
    ```

## 1.2 使用

Docker Machine 支持多种后端驱动，包括虚拟机、本地主机和云平台等。

### 1.2.1 创建本地主机实例

#### 1.2.1.1 Virtualbox 驱动

安装 `virtualbox`

https://www.if-not-true-then-false.com/2010/install-virtualbox-with-yum-on-fedora-centos-red-hat-rhel/

```sh
# 1 下载相应的repo包
cd /etc/yum.repos.d
wget http://download.virtualbox.org/virtualbox/rpm/rhel/virtualbox.repo

# 2 更新并搜索yum里的版本
yum update
yum clean all
yum makecache #更新缓存
yum search VirtualBox # 找到最新的文件名：VirtualBox-6.0.x86_64

# 3 安装
yum install VirtualBox-6.0

# 4 安装内核
rpm -qa |grep kernel
yum install kernel-devel 
yum install kernel-headers
rpm -qa gcc
rpm -qa make
rpm -qa perl
yum install gcc
/sbin/vboxconfig
```

使用 virtualbox 类型的驱动，创建一台 Docker 主机，命名为 test。

```sh
$ docker-machine create -d virtualbox test
```

你也可以在创建时加上如下参数，来配置主机或者主机上的 `Docker`。

* `--engine-opt dns=114.114.114.114` 配置 Docker 的默认 DNS

* `--engine-registry-mirror https://dockerhub.azk8s.cn` 配置 Docker 的仓库镜像

* `--virtualbox-memory 2048` 配置主机内存

* `--virtualbox-cpu-count 2` 配置主机 CPU

更多参数请使用 `docker-machine create --driver virtualbox --help` 命令查看。

#### 1.2.1.2 macOS xhyve 驱动

`xhyve` 驱动 GitHub: https://github.com/zchee/docker-machine-driver-xhyve

`xhyve` 是 `macOS` 上轻量化的虚拟引擎，使用其创建的 `Docker Machine` 较 `VirtualBox` 驱动创建的运行效率要高。

```sh
$ brew install docker-machine-driver-xhyve

$ docker-machine create \
      -d xhyve \
      # --xhyve-boot2docker-url ~/.docker/machine/cache/boot2docker.iso \
      --engine-opt dns=114.114.114.114 \
      --engine-registry-mirror https://dockerhub.azk8s.cn \
      --xhyve-memory-size 2048 \
      --xhyve-rawdisk \
      --xhyve-cpu-count 2 \
      xhyve
```

> 注意：非首次创建时建议加上 `--xhyve-boot2docker-url ~/.docker/machine/cache/boot2docker.iso` 参数，避免每次创建时都从 `GitHub` 下载 `ISO` 镜像。

更多参数请使用 `docker-machine create --driver xhyve --help` 命令查看。

#### 1.2.1.3 Windows 10

`Windows 10` 安装 `Docker Desktop for Windows` 之后不能再安装 `VirtualBox`，也就不能使用 `virtualbox` 驱动来创建 `Docker Machine`，我们可以选择使用 `hyperv` 驱动。

> 注意，必须事先在 `Hyper-V` 管理器中新建一个 外部虚拟交换机 执行下面的命令时，使用 `--hyperv-virtual-switch=MY_SWITCH` 指定虚拟交换机名称

```sh
$ docker-machine create --driver hyperv --hyperv-virtual-switch=MY_SWITCH vm
```

更多参数请使用 `docker-machine create --driver hyperv --help` 命令查看。

### 1.2.2 使用介绍

创建好主机之后，查看主机

```sh
$ docker-machine ls

NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER       ERRORS
test      -        virtualbox   Running   tcp://192.168.99.187:2376           v17.10.0-ce
```

创建主机成功后，可以通过 `env` 命令来让后续操作对象都是目标主机。

```sh
$ docker-machine env test
```

后续根据提示在命令行输入命令之后就可以操作 test 主机。

也可以通过 SSH 登录到主机。

```sh
$ docker-machine ssh test

docker@test:~$ docker --version
Docker version 17.10.0-ce, build f4ffd25
```

连接到主机之后你就可以在其上使用 Docker 了。

### 1.2.3 官方支持驱动

通过 `-d` 选项可以选择支持的驱动类型。

* amazonec2
* azure
* digitalocean
* exoscale
* generic
* google
* hyperv
* none
* openstack
* rackspace
* softlayer
* virtualbox
* vmwarevcloudair
* vmwarefusion
* vmwarevsphere

### 1.2.4 第三方驱动

请到 [第三方驱动列表](https://github.com/docker/docker.github.io/blob/master/machine/AVAILABLE_DRIVER_PLUGINS.md) 查看

### 1.2.5 操作命令

* active 查看活跃的 Docker 主机
* config 输出连接的配置信息
* create 创建一个 Docker 主机
* env 显示连接到某个主机需要的环境变量
* inspect 输出主机更多信息
* ip 获取主机地址
* kill 停止某个主机
* ls 列出所有管理的主机
* provision 重新设置一个已存在的主机
* regenerate-certs 为某个主机重新生成 TLS 认证信息
* restart 重启主机
* rm 删除某台主机
* ssh SSH 到主机上执行命令
* scp 在主机之间复制文件
* mount 挂载主机目录到本地
* start 启动一个主机
* status 查看主机状态
* stop 停止一个主机
* upgrade 更新主机 Docker 版本为最新
* url 获取主机的 URL
* version 输出 docker-machine 版本信息
* help 输出帮助信息
* 每个命令，又带有不同的参数，可以通过 `docker-machine COMMAND --help` 来查看具体的用法。