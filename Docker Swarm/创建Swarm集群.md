## 1 创建Swarm集群

### 1.1 初始化集群

在 [Docker Machine](../三驾马车/Docker&#32;Machine.md) 中我们可以了解到 Docker Machine 可以在数秒内创建一个虚拟的 Docker 主机，下面我们使用它来创建三个 Docker 主机，并加入到集群中。

首先创建一个Docker主机作为管理节点

```sh
$ docker-machine create -d virtualbox manager
```

使用 `docker swarm init` 在管理节点初始化一个 `Swarm` 集群。

```sh
$ docker-machine ssh manager

docker@manager:~$ docker swarm init --advertise-addr 192.168.56.103
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.56.103:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

如果你的 Docker 主机有多个网卡，拥有多个 IP，必须使用 `--advertise-addr` 指定 IP。

> 执行 `docker swarm init` 命令的节点自动成为管理节点。

### 1.2 增加工作节点

拥有了一个管理节点，下面继续创建两个 Docker 主机作为工作节点，并加入到集群中。

```sh
$ docker-machine create -d virtualbox worker1

$ docker-machine ssh worker1

docker@worker1:~$ docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.56.103:2377

This node joined a swarm as a worker.
```

```sh 
$ docker-machine create -d virtualbox worker2

$ docker-machine ssh worker2

docker@worker1:~$ docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.56.103:2377

This node joined a swarm as a worker.
```

### 1.3 查看集群

经过上边的两步，我们已经拥有了一个最小的 `Swarm` 集群，包含一个管理节点和两个工作节点。

在管理节点使用 `docker node ls` 查看集群。

```sh
$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
03g1y59jwfg7cf99w4lt0f662    worker2   Ready   Active
9j68exjopxe7wfl6yuxml7a7j    worker1   Ready   Active
dxn1zf6l61qsb1josjja83ngz *  manager   Ready   Active        Leader
```