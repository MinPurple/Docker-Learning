# 使用 Django

我们现在将使用 `Docker Compose` 配置并运行一个 `Django/PostgreSQL` 应用。

## 1 准备

在一切工作开始前，需要先编辑好三个必要的文件。

* `Dockerfile`

    第一步，因为应用将要运行在一个满足所有环境依赖的 Docker 容器里面，那么我们可以通过编辑 `Dockerfile` 文件来指定 Docker 容器要安装内容。内容如下：

    ```dockerfile
    FROM python:3
    ENV PYTHONUNBUFFERED 1
    RUN mkdir /code
    WORKDIR /code
    COPY requirements.txt /code/
    RUN pip install -r requirements.txt
    COPY . /code/
    ```

    以上内容指定应用将使用安装了 Python 以及必要依赖包的镜像。更多关于如何编写 `Dockerfile` 文件的信息可以查看 [Dockerfile](../Dockerfile指令详解/README.md) 使用。

* `requirements.txt`

    第二步，在 `requirements.txt` 文件里面写明需要安装的具体依赖包名。

    ```js
    Django>=2.0,<3.0
    psycopg2>=2.7,<3.0
    ```

* `docker-compose.yml`

    第三步，`docker-compose.yml` 文件将把所有的东西关联起来。它描述了应用的构成（一个 web 服务和一个数据库）、使用的 Docker 镜像、镜像之间的连接、挂载到容器的卷，以及服务开放的端口。

    ```yaml

    version: "3"
    services:

      db:
        image: postgres

      web:
        build: .
        command: python manage.py runserver 0.0.0.0:8000
        volumes:
          - .:/code
        ports:
          - "8000:8000"
        links:
          - db
    ```

    查看 [docker-compose](../三驾马车/Docker&#32;Compose.md) 章节 了解更多详细的工作机制。

## 2 启动

现在我们就可以使用 `docker-compose run` 命令启动一个 `Django` 应用了。

```sh
$ docker-compose run web django-admin startproject django_example .
```

由于 web 服务所使用的镜像并不存在，所以 `Compose` 会首先使用 `Dockerfile` 为 `web` 服务构建一个镜像，接着使用这个镜像在容器里运行 `django-admin startproject django_example` 指令。

这将在当前目录生成一个 Django 应用。

```sh
$ ls
Dockerfile       docker-compose.yml          django_example       manage.py       requirements.txt
```

如果你的系统是 Linux,记得更改文件权限。

```sh 
$ sudo chown -R $USER:$USER .
```

首先，我们要为应用设置好数据库的连接信息。用以下内容替换 `django_example/settings.py` 文件中 `DATABASES = ...` 定义的节点内容。

```py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'HOST': 'db',
        'PORT': 5432,
    }
}
```

这些信息是在 `postgres` 镜像固定设置好的。然后，运行 `docker-compose up` ：

```sh
$ docker-compose up

Recreating djangotest_db_1 ... done
Creating djangotest_web_1  ... done
Attaching to djangotest_db_1, djangotest_web_1
db_1   | 2019-10-31 08:03:11.839 UTC [1] LOG:  starting PostgreSQL 12.0 (Debian 12.0-2.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
db_1   | 2019-10-31 08:03:11.840 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
db_1   | 2019-10-31 08:03:11.840 UTC [1] LOG:  listening on IPv6 address "::", port 5432
db_1   | 2019-10-31 08:03:11.846 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
db_1   | 2019-10-31 08:03:11.871 UTC [22] LOG:  database system was shut down at 2019-10-31 08:03:10 UTC
db_1   | 2019-10-31 08:03:11.876 UTC [1] LOG:  database system is ready to accept connections
web_1  | Performing system checks...
web_1  | 
web_1  | System check identified no issues (0 silenced).
web_1  | 
web_1  | You have 13 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
web_1  | Run 'python manage.py migrate' to apply them.
web_1  | October 31, 2019 - 08:03:13
web_1  | Django version 1.11.25, using settings 'django_example.settings'
web_1  | Starting development server at http://0.0.0.0:8000/
web_1  | Quit the server with CONTROL-C.
web_1  | [31/Oct/2019 08:03:53] "GET / HTTP/1.1" 200 1716
```

这个 Django 应用已经开始在你的 Docker 守护进程里监听着 8000 端口了。打开 `127.0.0.1:8000` 即可看到 Django 欢迎页面。

你还可以在 Docker 上运行其它的管理命令，例如对于同步数据库结构这种事，在运行完 `docker-compose up `后，在另外一个终端**进入文件夹**运行以下命令即可：

```sh
$ docker-compose run web python manage.py migrate
```