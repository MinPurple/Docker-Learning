## 1 Docker Compose

### 1.1 简介

Docker Compose是Docker官方编排（Orchestration）项目之一，负责快速的部署分布式应用。其代码目前在 https://github.com/docker/compose 上开源。Compose 定位是 「定义和运行多个 Docker 容器的应用（Defining and running multi-container Docker applications）」，其前身是开源项目`Fig`。

我们使用一个`Dockerfile`模板文件，可以很方便的定义一个单独的应用容器。然而，在日常工作中，经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现一个 `Web` 项目，除了 `Web` 服务容器本身，往往还需要再加上后端的**数据库服务容器**或者**缓存服务容器**，甚至还包括**负载均衡容器**等。

`Compose` 恰好满足了这样的需求。它允许用户通过一个单独的 `docker-compose.yml`模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（`project`）。

Compose 中有两个重要的概念：

* 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
* 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 `docker-compose.yml` 文件中定义。

Compose 的默认管理对象是项目，一个项目可以由多个服务（ 容器） 关联而成, 通过子命令对项目中的一组容器进行便捷地生命周期管理。

Compose 项目由 Python 编写，实现上调用了 Docker 服务提供的 API 来对容器进行管理。所以只要所操作的平台支持 Docker API，就可以在其上利用 Compose 来进行编排管理。

### 1.2 安装与卸载

`Compose` 支持 Linux、macOS、Windows 10 三大平台。`Compose` 可以通过 `Python` 的包管理工具`pip`进行安装，也可以直接下载编译好的二进制文件使用，甚至能够直接在 `Docker` 容器中运行。

前两种方式是传统方式，适合本地环境下安装使用；最后一种方式则不破坏系统环境，更适合云计算场景。Docker for Mac 、Docker for Windows 自带 `docker-compose` 二进制文件，安装 Docker 之后可以直接使用。

```sh
$ docker-compose --version
docker-compose version 1.17.1, build 6d101fb
```

#### 1.2.1 二进制安装

在 Linux 上的也安装十分简单，从 官方 [GitHub Release](https://github.com/docker/compose/releases) 处直接下载编译好的二进制文件即可。例如，在 Linux 64 位系统上直接下载对应的二进制包。

```sh
$ curl -L https://github.com/docker/compose/releases/download/1.25.0-rc4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

$ sudo chmod +x /usr/local/bin/docker-compose
```

#### 1.2.2 PIP 安装

> 注： x86_64 架构的 Linux 建议按照上边的方法下载二进制包进行安装，如果您计算机的架构是 ARM (例如，树莓派)，再使用 pip 安装。

这种方式是将 Compose 当作一个 Python 应用来从 pip 源中安装。执行安装命令：

```sh
$ sudo pip install -U docker-compose
# 可以看到类似如下输出， 说明安装成功。
Collecting docker-compose
  Downloading docker-compose-1.17.1.tar.gz (149kB): 149kB downloaded
...
Successfully installed docker-compose cached-property requests texttable websocket-client docker-py dockerpty six enum34 backports.ssl-match-hostname ipaddress

# bash 补全命令：

$ curl -L https://raw.githubusercontent.com/docker/compose/1.8.0/contrib/completion/bash/docker-compose > /etc/bash_completion.d/docker-compose
```

安装成功后， 可以查看 docker-compose 命令的用法。

```sh
$ docker-compose -h
Define and run multi-container applications with Docker.
Usage:
docker-compose [-f=<arg>...] [options] [COMMAND] [ARGS...]
docker-compose -h|--help

Options:
-f, --file FILE Specify an alternate compose file (d
efault: docker-compose.yml)
-p, --project-name NAME Specify an alternate project name (d
efault: directory name)
--x-networking (EXPERIMENTAL) Use new Docker networ
king functionality.
Requires Docker 1.9 or later.
--x-network-driver DRIVER (EXPERIMENTAL) Specify a network dri
ver (default: "bridge").
Requires Docker 1.9 or later.
--verbose Show more output
-v, --version Print version and exit

Commands:
build Build or rebuild services
help Get help on a command
kill Kill containers
logs View output from containers
pause Pause services
port Print the public port for a port binding
ps List containers
pull Pulls service images
restart Restart services
rm Remove stopped containers
run Run a one-off command
scale Set number of containers for a service
start Start services
stop Stop services
unpause Unpause services
up Create and start containers
migrate-to-labels Recreate containers to add labels
version Show the Docker-Compose version information
```


#### 1.2.3 容器中执行

Compose 既然是一个 Python 应用，自然也可以直接用容器来执行它。

```sh
$ curl -L https://github.com/docker/compose/releases/download/1.8.0/run.sh > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

实际上，查看下载的run.sh脚本内容，如下:

```sh
set -e

VERSION="1.8.0"
IMAGE="docker/compose:$VERSION"

# Setup options for connecting to docker hostif [ -z "$DOCKER_HOST" ]; then
    DOCKER_HOST="/var/run/docker.sock"fiif [ -S "$DOCKER_HOST" ]; then
    DOCKER_ADDR="-v $DOCKER_HOST:$DOCKER_HOST -e DOCKER_HOST"else
    DOCKER_ADDR="-e DOCKER_HOST -e DOCKER_TLS_VERIFY -e DOCKER_CERT_PATH"fi

# Setup volume mounts for compose config and contextif [ "$(pwd)" != '/' ]; then
    VOLUMES="-v $(pwd):$(pwd)"fiif [ -n "$COMPOSE_FILE" ]; then
    compose_dir=$(dirname $COMPOSE_FILE)fi# TODO: also check --file argumentif [ -n "$compose_dir" ]; then
    VOLUMES="$VOLUMES -v $compose_dir:$compose_dir"fiif [ -n "$HOME" ]; then
    VOLUMES="$VOLUMES -v $HOME:$HOME -v $HOME:/root" # mount $HOME in /root to share docker.configfi
# Only allocate tty if we detect oneif [ -t 1 ]; then
    DOCKER_RUN_OPTIONS="-t"fiif [ -t 0 ]; then
    DOCKER_RUN_OPTIONS="$DOCKER_RUN_OPTIONS -i"fi
exec docker run --rm $DOCKER_RUN_OPTIONS $DOCKER_ADDR $COMPOSE_OPTIONS $VOLUMES -w "$(pwd)" $IMAGE "$@"
```

可以看到，它其实是下载了docker/compose镜像并运行。

#### 1.2.4 卸载

* 如果是二进制包方式安装的，删除二进制文件即可。

    ```sh
    $ sudo rm /usr/local/bin/docker-compose
    ```

* 如果是通过 pip 安装的，则执行如下命令即可删除。

    ```sh
    $ sudo pip uninstall docker-compose
    ```

### 1.3 使用

#### 1.3.1 术语

首先介绍几个术语。

* 服务 (service)：一个应用容器，实际上可以运行多个相同镜像的实例。

* 项目 (project)：由一组关联的应用容器组成的一个完整业务单元。

可见，一个项目可以由多个服务（容器）关联而成，Compose 面向项目进行管理。

下面我们用 Python 来建立一个能够记录页面访问次数的 web 网站。 新建文件夹，在该目录中编写app.py文件

#### 1.3.2 场景

最常见的项目是 web 网站，该项目应该包含 web 应用和缓存。

下面我们用 Python 来建立一个能够记录页面访问次数的 web 网站。

* web 应用

    新建文件夹，在该目录中编写 `app.py` 文件

    ```py
    from flask import Flask
    from redis import Redis

    app = Flask(__name__)
    redis = Redis(host='redis', port=6379)

    @app.route('/')
    def hello():
        count = redis.incr('hits')
        return 'Hello World! 该页面已被访问 {} 次。\n'.format(count)

    if __name__ == "__main__":
        app.run(host="0.0.0.0", debug=True)
    ```

* `Dockerfile`

    编写 Dockerfile 文件，内容为

    ```dockerfile
    FROM python:3.6-alpine
    ADD . /code
    WORKDIR /code
    RUN pip install redis flask
    CMD ["python", "app.py"]
    ```

* `docker-compose.yml`
    
    编写 `docker-compose.yml` 文件，这个是 Compose 使用的主模板文件。

    ```yaml
    version: '3'
    services:

    web:
      build: .
      ports:
        - "5000:5000"

      redis:
        image: "redis:alpine"
    ```

* 运行 compose 项目

    ```sh
    $ docker-compose up
    ```

此时访问本地 5000 端口，每次刷新页面，计数就会加 1。


### 1.4 Compose 命令说明

#### 1.4.1 命令对象与格式

对于 Compose 来说， 大部分命令的对象既可以是项目本身， 也可以指定为项目中的服务或者容器。 如果没有特别的说明， 命令对象将是项目， 这意味着项目中所有的服务都会受到命令影响。

执行 `docker-compose [COMMAND] --help` 或者 `docker-compose help [COMMAND]` 可以查看具体某个命令的使用格式。

Compose 命令的基本的使用格式是

```sh
docker-compose [-f=<arg>...] [options] [COMMAND] [ARGS...]
```

#### 1.4.2 命令选项

* `-f`, --file FILE 指定使用的 **Compose 模板文件**， 默认为 `dockercompose.yml` ,可以多次指定。
* `-p`, --project-name NAME 指定**项目名称**， 默认将使用所在目录名称作为
项目名。
* `--x-networking` 使用 Docker 的可拔插网络后端特性（ 需要 Docker 1.9 及以后版本） 。
* `--x-network-driver DRIVER` 指定网络后端的驱动， 默认为 `bridge` （ 需要 Docker 1.9 及以后版本） 。
* `--verbose` 输出更多调试信息。
* `-v`, --version 打印版本并退出。

#### 1.4.3 命令使用说明

* **build**

    格式为 ：

    ```sh
    $ docker-compose build [options] [SERVICE...] 
    ```

    构建（ 重新构建） 项目中的服务容器。

    服务容器一旦构建后， 将会带上一个标记名， 例如对于 web 项目中的一个 db 容器， 可能是 web_db。
    
    可以随时在项目目录下运行 docker-compose build 来重新构建服务。

    选项包括：
    * `--force-rm` 删除构建过程中的临时容器。
    * `--no-cache` 构建镜像过程中不使用 cache（ 这将加长构建过程） 。
    * `--pull` 始终尝试通过 pull 来获取更新版本的镜像。

* **help**
    
    获得一个命令的帮助

* **kill**

    格式为 
    
    ```sh
    $ docker-compose kill [options] [SERVICE...] 
    ```

    通过发送 SIGKILL 信号来强制停止服务容器。

    支持通过 `-s` 参数来指定发送的信号， 例如通过如下指令发送 SIGINT 信号。

    ```sh
    $ docker-compose kill -s SIGINT
    ```

* **logs**

    格式为 
    
    ```sh
    $ docker-compose logs [options] [SERVICE...]
    ```

    查看服务容器的输出。 默认情况下， `docker-compose` 将对不同的服务输出使用不同的颜色来区分。 可以通过 `--no-color` 来关闭颜色。
    
    该命令在调试问题的时候十分有用。

* **pause**

    格式为 
    
    ```sh
    $ docker-compose pause [SERVICE...]
    ```

    暂停一个服务容器。

* **port**

    格式为 
    
    ```sh
    $ docker-compose port [options] SERVICE PRIVATE_PORT
    ```

    打印某个容器端口所映射的公共端口。

    选项包括：
    * `--protocol=proto` 指定端口协议， tcp（ 默认值） 或者 udp。
    * `--index=index` 如果同一服务存在多个容器， 指定命令对象容器的序号（ 默认为 1)

* **ps**

    格式为 
    
    ```sh
    $ docker-compose ps [options] [SERVICE...] 
    ```

    列出项目中目前的所有容器。
    
    选项：
    * `-q` 只打印容器的 ID 信息

* **pull**

    格式为 
    
    ```sh
    $ docker-compose pull [options] [SERVICE...]
    ```

    拉取服务依赖的镜像。
    
    选项：
    * `--ignore-pull-failures` 忽略拉取镜像过程中的错误

* **restart**

    格式为 
    
    ```sh
    $ docker-compose restart [options] [SERVICE...]
    ```

    重启项目中的服务。
    
    选项：
    * `-t, --timeout TIMEOUT` 指定重启前停止容器的超时（ 默认为 10 秒）

* **rm**

    格式为 
    
    ```sh
    $ docker-compose rm [options] [SERVICE...]
    ```

    删除所有（ 停止状态的） 服务容器。 推荐先执行 docker-compose stop 命令来停止容器。
    
    选项：
    * `-f, --force` 强制直接删除， 包括非停止状态的容器。 一般尽量不要使用该选项。
    * `-v` 删除容器所挂载的数据卷。

* **run**

    格式为 
    
    ```sh
    $ docker-compose run [options] [-p PORT...] [-e KEY=VAL...] SERVICE [COMMAND] [ARGS...]
    ```

    在指定服务上执行一个命令。例如：
    
    ```sh
    $ docker-compose run ubuntu ping docker.com
    ```

    将会启动一个 ubuntu 服务容器， 并执行 `ping docker.com` 命令。

    默认情况下， 如果存在关联， 则所有关联的服务将会自动被启动， 除非这些服务已经在运行中。
    
    该命令类似启动容器后运行指定的命令， 相关卷、 链接等等都将会按照配置自动创建。

    两个不同点：
    * 给定命令将会覆盖原有的自动运行命令；
    * 不会自动创建端口， 以避免冲突。

    如果不希望自动启动关联的容器， 可以使用 `--no-deps` 选项， 例如

    ```sh
    $ docker-compose run --no-deps web python manage.py shell
    ```

    将不会启动 web 容器所关联的其它容器。

    选项：
    * `-d` 后台运行容器。
    * `--name` NAME 为容器指定一个名字。
    * `--entrypoint` CMD 覆盖默认的容器启动指令。
    * `-e KEY=VAL` 设置环境变量值， 可多次使用选项来设置多个环境变量。
    * `-u`, `--user=""` 指定运行容器的用户名或者 uid。
    * `--no-deps` 不自动启动关联的服务容器。
    * `--rm` 运行命令后自动删除容器， d 模式下将忽略。
    * `-p, --publish=[]` 映射容器端口到本地主机。
    * `--service-ports` 配置服务端口并映射到本地主机。
    * `-T` 不分配伪 tty， 意味着依赖 tty 的指令将无法运行。

* **scale**

    格式为 
    
    ```sh
    $ docker-compose scale [options] [SERVICE=NUM...]
    ```

    设置指定服务运行的容器个数。

    通过 service=num 的参数来设置数量。 例如：

    ```sh
    $ docker-compose scale web=3 db=2
    ```

    将启动 3 个容器运行 web 服务， 2 个容器运行 db 服务。

    一般的， 当指定数目多于该服务当前实际运行容器， 将新创建并启动容器； 反之，将停止容器。
    
    选项：
    * -t, --timeout TIMEOUT 停止容器时候的超时（ 默认为 10 秒） 。

* **start**

    格式为 
    
    ```sh
    $ docker-compose start [SERVICE...]
    ```

    启动已经存在的服务容器。

* **stop**

    格式为 
    
    ```sh
    $ docker-compose stop [options] [SERVICE...]
    ```

    停止已经处于运行状态的容器， 但不删除它。 通过 `docker-compose start` 可以再次启动这些容器。

    选项：
    * -t, --timeout TIMEOUT 停止容器时候的超时（ 默认为 10 秒） 。
    
* **unpause**

    格式为 
    
    ```sh
    $ docker-compose unpause [SERVICE...]
    ```

    恢复处于暂停状态中的服务。

* **up**

    格式为 
    
    ```sh
    $ docker-compose up [options] [SERVICE...]
    ```

    该命令十分强大， 它将尝试自动完成包括构建镜像， （ 重新） 创建服务， 启动服务，并关联服务相关容器的一系列操作。链接的服务都将会被自动启动， 除非已经处于运行状态。

    可以说， 大部分时候都可以直接通过该命令来启动一个项目。

    默认情况， `docker-compose up` 启动的容器都在前台， 控制台将会同时打印所有容器的输出信息， 可以很方便进行调试。
    
    当通过 Ctrl-C 停止命令时， 所有容器将会停止。
    
    如果使用 `docker-compose up -d` ， 将会在后台启动并运行所有的容器。 一般推荐生产环境下使用该选项。

    默认情况， 如果服务容器已经存在， `docker-compose up` 将会尝试停止容器，然后重新创建（ 保持使用 `volumes-from` 挂载的卷） ， 以保证新启动的服务匹配`docker-compose.yml` 文件的最新内容。 
    
    如果用户不希望容器被停止并重新创建， 可以使用 `docker-compose up --no-recreate` 。 这样将只会启动处于停止状态的容器， 而忽略已经运行的服务。
    
    如果用户只想重新部署某个服务， 可以使用`docker-compose up --no-deps -d <SERVICE_NAME>` 来重新创建服务并后台停止旧服务， 启动新服务， 并不会影响到其所依赖的服务。

    options:
    * `-d` 在后台运行服务容器。
    * `--no-color` 不使用颜色来区分不同的服务的控制台输出。
    * `--no-deps` 不启动服务所链接的容器。
    * `--force-recreate` 强制重新创建容器， 不能与 --no-recreate 同时使用。
    * `--no-recreate` 如果容器已经存在了， 则不重新创建， 不能与 `--forcerecreate` 同时使用。
    * `--no-build` 不自动构建缺失的服务镜像。
    * `-t`, --timeout TIMEOUT 停止容器时候的超时（ 默认为 10 秒） 。

* **migrate-to-labels**

    格式为 
    
    ```sh
    $ docker-compose migrate-to-labels 
    ```

    重新创建容器， 并添加 `label`。

    主要用于升级 1.2 及更早版本中创建的容器， 添加缺失的容器标签。实际上， 最彻底的办法当然是删除项目， 然后重新创建。

* **version**

    格式为 
    
    ```sh
    $ docker-compose version
    ```

    打印版本信息。

### 1.5 Compose 模板文件

模板文件是使用 `Compose` 的核心， 涉及到的指令关键字也比较多

默认的模板文件名称为 `docker-compose.yml` ， 格式为 `YAML` 格式。

在旧版本（ 版本 1） 中， 其中每个顶级元素为服务名称， 次级元素为服务容器的配置信息，例如:

```yaml
webapp:
  image: examples/web
  ports:
    - "80:80"
  volumes:
    - "/data"
```

版本 2 扩展了 Compose 的语法， 同时尽量保持跟版本 1 的兼容， 除了可以声明网络和存储信息外， 最大的不同一是添加了版本信息， 另一个是需要将所有的服务放到 services 根下面。例如， 上面例子改写为版本 2， 内容为：

```yaml
version: "2"
services:
  webapp:
    image: examples/web
    ports:
      - "80:80"
    volumes:
      - "/data"
```

注意每个服务都必须通过 `image` 指令指定镜像或 `build` 指令（ 需要 Dockerfile） 等来自动构建生成镜像。

如果使用 `build` 指令， 在 `Dockerfile` 中设置的选项(例如： `CMD` , `EXPOSE` ,`VOLUME` , `ENV` 等) 将会自动被获取， 无需在 `docker-compose.yml` 中再次设
置。

下面分别介绍各个指令的用法。

#### 1.5.1 build

**指定 Dockerfile 所在文件夹的路径**（ 可以是绝对路径， 或者相对 `dockercompose.yml` 文件的路径）。 Compose 将会利用它自动构建这个镜像， 然后使用这个镜像。

```yaml
build: /path/to/build/dir
```

#### 1.5.2 cap_add, cap_drop

**指定容器的内核能力（ capacity） 分配。**

例如， 让容器拥有所有能力可以指定为：

```yaml
cap_add:
  - ALL
```

去掉 NET_ADMIN 能力可以指定为：

```yaml
cap_drop:
  - NET_ADMIN
```

#### 1.5.2 command

覆盖容器启动后默认执行的命令。

```yaml
command: echo "hello world"
```

#### 1.5.3 cgroup_parent

指定父 cgroup 组， 意味着将继承该组的资源限制。例如， 创建了一个 cgroup 组名称为cgroups_1 。

```yaml
cgroup_parent: cgroups_1
```

#### 1.5.4 container_name

指定容器名称。 默认将会使用 **项目名称_服务名称_序号** 这样的格式。例如：

```yaml
container_name: docker-web-container
```

需要注意， 指定容器名称后， 该服务将**无法进行扩展（ scale）** ， 因为 Docker 不允
许多个容器具有相同的名称。

#### 1.5.5 devices

指定设备映射关系。例如：

```yaml
devices:
  - "/dev/ttyUSB1:/dev/ttyUSB0"
```

#### 1.5.6 dns

自定义 DNS 服务器。 可以是一个值， 也可以是一个列表。

```yaml
dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 9.9.9.9
```

#### 1.5.7 dns_search

配置 DNS 搜索域。 可以是一个值， 也可以是一个列表。

```yaml
dns_search: example.com
dns_search:
  - domain1.example.com
  - domain2.example.com
```

#### 1.5.8 dockerfile

如果需要指定额外的编译镜像的 Dockefile 文件， 可以通过该指令来指定。

```yaml
dockerfile: Dockerfile-alternate
```

注意， 该指令**不能跟 image 同时使用**， 否则 Compose 将不知道根据哪个指令来
生成最终的服务镜像。

#### 1.5.9 env_file

从文件中获取环境变量， 可以为单独的文件路径或列表。

如果通过 `docker-compose -f FILE` 方式来指定 `Compose` 模板文件， 则`env_file` 中变量的路径会基于模板文件路径。如果有变量名称与 environment 指令冲突， 则按照惯例， 以后者为准。

```yaml
env_file: .env
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

环境变量文件中每一行必须符合格式， 支持 # 开头的注释行。

```py
# common.env: Set development environment
PROG_ENV=development
```

#### 1.5.10 environment

设置环境变量。 你可以使用数组或字典两种格式。

只给定名称的变量会自动获取运行 Compose 主机上对应变量的值， 可以用来防止泄露不必要的数据。例如

```yaml
environment:
  RACK_ENV: development
  SESSION_SECRET:
```

或者:

```yaml
environment:
  - RACK_ENV=development
  - SESSION_SECRET
```

注意，如果变量名称或者值中用到 `true|false， yes|no`  等表达布尔含义的词汇，最好放到引号里，避免 YAML 自动解析某些内容为对应的布尔语义。

http://yaml.org/type/bool.html 中给出了这些特定词汇， 包括

```js
y|Y|yes|Yes|YES|n|N|no|No|NO
|true|True|TRUE|false|False|FALSE
|on|On|ON|off|Off|OFF
```

#### 1.5.11 expose

暴露端口， 但不映射到宿主机， 只被连接的服务访问。

仅可以指定内部端口为参数

```yaml
expose:
  - "3000"
  - "8000"
```

#### 1.5.12 extends

基于其它模板文件进行扩展。

例如我们已经有了一个 webapp 服务， 定义一个基础模板文件为 `common.yml` 。

```yaml
# common.yml
webapp:
  build: ./webapp
  environment:
    - DEBUG=false
    - SEND_EMAILS=false
```

再编写一个新的 `development.yml` 文件， 使用 `common.yml` 中的 webapp 服务进行扩展。

```yaml
# development.yml
web:
  extends:
    file: common.yml
    service: webapp
  ports:
    - "8000:8000"
  links:
    - db
  environment:
    - DEBUG=true
  db:
    image: postgres
```

后者会自动继承 common.yml 中的 webapp 服务及环境变量定义。

使用 extends 需要注意：

* 要避免出现循环依赖， 例如 A 依赖 B， B 依赖 C， C 反过来依赖 A 的情况。
* extends 不会继承 links 和 volumes_from 中定义的容器和数据卷资源。

一般的， 推荐在基础模板中只定义一些可以共享的镜像和环境变量， 在扩展模板中
具体指定应用变量、 链接、 数据卷等信息。

#### 1.5.13 external_links

链接到 `docker-compose.yml` 外部的容器， 甚至 并非 Compose 管理的外部容器。 参数格式跟 links 类似。

```yaml
external_links:
  - redis_1
  - project_db_1:mysql
  - project_db_1:postgresql
```

#### 1.5.14 extra_hosts

类似 Docker 中的 `--add-host` 参数， 指定额外的 `host` 名称映射信息。

```yaml
extra_hosts:
  - "googledns:8.8.8.8"
  - "dockerhub:52.1.157.61"
```

会在启动后的服务容器中 /etc/hosts 文件中添加如下两条条目。
```js
8.8.8.8 googledns
52.1.157.61 dockerhub
```

#### 1.5.15 image

指定为镜像名称或镜像 ID。 如果镜像在本地不存在， Compose 将会尝试拉去这个镜像。

```yaml
image: ubuntu
image: orchardup/postgresql
image: a4bc65fd
```

#### 1.5.16 labels

为容器添加 Docker 元数据（ metadata） 信息。 例如可以为容器添加辅助说明信息。

```yaml
labels:
  com.startupteam.description: "webapp for a startup team"
  com.startupteam.department: "devops department"
  com.startupteam.release: "rc3 for v1.0"
```

#### 1.5.17 links

链接到其它服务中的容器。 使用**服务名称**（同时作为别名） 或**服务名称： 服务别名**`（SERVICE:ALIAS）`格式都可以。

```yaml
links:
  - db
  - db:database
  - redis
```

使用的别名将会自动在服务容器中的 /etc/hosts 里创建。 例如：

```js
172.17.2.186 db
172.17.2.186 database
172.17.2.187 redis
```

#### 1.5.18 log_driver

类似 Docker 中的 `--log-driver` 参数， 指定日志驱动类型。目前支持三种日志驱动类型。

```yaml
log_driver: "json-file"
log_driver: "syslog"
log_driver: "none"
```
#### 1.5.19 log_opt

日志驱动的相关参数。例如

```yaml
log_driver: "syslog"
log_opt:
  syslog-address: "tcp://192.168.0.42:123"
```

#### 1.5.20 net

设置网络模式。 使用和 `docker client` 的 `--net` 参数一样的值。

```yaml
net: "bridge"
net: "none"
net: "container:[name or id]"
net: "host"
```

#### 1.5.21 pid

跟主机系统共享进程命名空间。 打开该选项的容器之间， 以及容器和宿主机系统之间可以通过进程 ID 来相互访问和操作。

```yaml
pid: "host"
```

#### 1.5.22 ports

暴露端口信息。

使用宿主： 容器 `（ HOST:CONTAINER）` 格式， 或者仅仅指定容器的端口（ 宿主将
会随机选择端口） 都可以。

```yaml
ports:
  - "3000"
  - "8000:8000"
  - "49100:22"
  - "127.0.0.1:8001:8001"
```

注意： 当使用 `HOST:CONTAINER` 格式来映射端口时， 如果你使用的容器端口小于 60 并且没放到引号里， 可能会得到错误结果， 因为 YAML 会自动解析 `xx:yy` 这种数字格式为 60 进制。 为避免出现这种问题， 建议数字串都采用引号包括起来的字符串格式。

#### 1.5.23 security_opt

指定容器模板标签（ label） 机制的默认属性（ 用户、 角色、 类型、 级别等） 。

例如配置标签的用户名和角色名。

```yaml
security_opt:
  - label:user:USER
  - label:role:ROLE
```

#### 1.5.24 ulimits

指定容器的 `ulimits` 限制值。

例如， 指定最大进程数为 65535， 指定文件句柄数为 20000（ 软限制， 应用可以随
时修改， 不能超过硬限制） 和 40000（ 系统硬限制， 只能 root 用户提高） 。

```yaml
ulimits:
  nproc: 65535
  nofile:
    soft: 20000
    hard: 40000
```

#### 1.5.25 volumes

数据卷所挂载路径设置。 可以设置宿主机路径 （ `HOST:CONTAINER` ） 或加上访问模式 （ `HOST:CONTAINER:ro` ） 。该指令中路径支持相对路径。 例如

```yaml
volumes:
  - /var/lib/mysql
  - cache/:/tmp/cache
  - ~/configs:/etc/configs/:ro
```

#### 1.5.26 volumes_driver

较新版本的 Docker 支持数据卷的插件驱动。用户可以先使用第三方驱动创建一个数据卷， 然后使用名称来访问它。此时， 可以通过 volumes_driver 来指定驱动。

```yaml
volume_driver: mydriver
```

#### 1.5.27 volumes_from

从另一个服务或容器挂载它的数据卷。

```yaml
volumes_from:
  - service_name
  - container_name
```

#### 1.5.28 其它指令

此外， 还有包括 `cpu_shares`, `cpuset`, `domainname`, `entrypoint`, `hostname`,
`ipc`, `mac_address`, `mem_limit`, `memswap_limit`, `privileged`, `read_only`,
`restart`, `stdin_open`, `tty`, `user`, `working_dir` 等指令， 基本跟 `docker-run` 中对应参数的功能一致。

例如， 指定使用 cpu 核 0 和 核 1， 只用 50% 的 CPU 资源：
```yaml
cpu_shares: 73
cpuset: 0,1
```

指定服务容器启动后执行的命令。
```yaml
entrypoint: /code/entrypoint.sh
```
指定容器中运行应用的用户名。
```yaml
user: nginx
```
指定容器中工作目录。
```yaml
working_dir: /code
```
指定容器中搜索域名、 主机名、 mac 地址等。
```yaml
domainname: your_website.com
hostname: test
mac_address: 08-00-27-00-0C-0A
```
指定容器中
```yaml
ipc: host
```
指定容器中内存和内存交换区限制都为 1G。
```yaml
mem_limit: 1g
memswap_limit: 1g
```
允许容器中运行一些特权命令。
```yaml
privileged: true
```

指定容器退出后的重启策略为始终重启。 该命令对保持服务始终运行十分有效， 在
生产环境中推荐配置为 always 或者 unless-stopped 。
```yaml
restart: always
```

以只读模式挂载容器的 root 文件系统， 意味着不能对容器内容进行修改。
```yaml
read_only: true
```
打开标准输入， 可以接受外部输入。
```yaml
stdin_open: true
```
模拟一个假的远程控制台。
```yaml
tty: true
```

### 1.6 读取环境变量

从 1.5.0 版本开始，Conpose 模板文件支持动态读取主机的系统环境变量。

例如，下面的Compose文件将从运行它的环境中读取变量 `${MONGO_VERSION}` 的值，并写入执行的指令中。

```yaml
db:
  image: "mongo:${MONGO_VERSION}"
```

如果执行 `MONGO_VERSION=3.0 docker-compose up` 则会启动一个 `mongo:3.2` 镜像的容器； 如果执行 `MONGO_VERSION=2.8 docker-compose up` 则会启动一个` mongo:2.8` 镜像的容器。