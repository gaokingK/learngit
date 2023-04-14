path:Docker/learn_Docker_Compose.md
# docker compose 简单使用
用来定义和运行多容器应用程序的工具。使用compose，可以使用YML文件来配置应用程序所需要的所有服务，然后，使用一个命令就可以创建并启动所有服务。

### 环境安装
- docker 安装
略
- docker compose 安装

运行以下命令以下载 Docker Compose 的当前稳定版本的二进制包：
```shell
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# 要安装其他版本的 Compose，请替换 1.24.1。
# 将可执行权限应用于二进制文件
$ sudo chmod +x /usr/local/bin/docker-compose
# 创建软链：
$ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
# 测试是否安装成功：
$ docker-compose --version
docker-compose version 1.24.1, build 4667896b
```

### 使用
#### 应用程序文件
```python
# app.py
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)


def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)


@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)

# requirements.txt
flask
redis
```
#### 准备Dockerfile
```dockerfile
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP app.py
ENV FLASK_RUN_HOST 0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
COPY . .
CMD ["flask", "run"]
```
#### 准备docker-compose.yml
```yml
# yaml 配置
version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

### 构建并运行应用
`docker-compose up -d`
path:Docker/learn_Docker_resource.md
## Docker的底层原理
- link:
    - https://mp.weixin.qq.com/s?__biz=Mzg2NTAyNTc5NQ==&mid=2247484441&idx=1&sn=ae2421fd803b911eb8670891cf90a677&chksm=ce612e75f916a763a3c46053b36aa89c49702064fa568db50e26605cff2c4a60db36037d0c44&scene=27
- flink:
    - learn_Linux_Cgroup&namespace
- docker 的底层原理是利用了Linux的namespace和cgroup
- cgroup 是用来制造约束（对资源的约束）的主要手段，namespace是用来修改进程视图（产生进程隔离）的主要手段
### namespace
- 容器是一个特殊一点的进程，docker在创建一个容器（相当于创建一个进程）时，传入一组namespace参数，这样容器就只能看到当前namespace限定的资源、文件、设备、网络等等资源。而对于宿主机以及其他不相干的程序完全看不到。
### Cgroup
- Cgroup的全称是Control Group，它的作用是限制一个进程组能使用的资源上限，比如cpu、磁盘、网络带宽、内存等
- linux的实现方式是在一个特定的目录下有特定的配置文件。/sys/fs/cgroup/；这一个个的文件夹就是cgroup可以限制的资源类型。比如/sys/fs/cgroup/cpu
- 每个容器的每种资源都在相应的配置文件夹的docker下面
```
# docker run -it -d  --cpu-period=100000 --cpu-quota=20000 busybox
2cf367a47f6cd9766effe154e9d725a21657663a3d14f731077ea7e09153cc35a
# ls -l /sys/fs/cgroup/cpu/docker
 2总用量 0
 3drwxr-xr-x 2 root root 0 1月  21 16:41 2cf367a47f6cd9766effe154e9d725a21657663a3d14f731077ea7e09153cc35a
```
## docker的资源限制与分配
- link：
    - https://juejin.cn/post/7011314884390420493#heading-1
- 默认情况下，docker时没有资源限制的，会尽可能多的使用宿主机分配的资源，但如果不对容器的资源进行限制，对性能需求高的容器就会抢占资源，对其他容器产生影响
- Docker提供了限制内存，CPU或磁盘IO的方法， 可以对容器所占用的硬件资源大小以及多少进行限制，我们在使用docker create创建一个容器或者docker run运行一个容器的时候就可以来对此容器的硬件资源做限制。
### 限制cpu
- CPU份额控制：-c或--cpu-shares
    - -c 或 --cpu-shares： 在有多个容器竞争 CPU 时我们可以设置每个容器能使用的 CPU 时间比例。
        - 这个比例叫作共享权值。共享式CPU资源，是按比例切分CPU资源；
- CPU核控制：--cpuset-cpus、--cpus
    - --cpus： 限制容器运行的核数；
    - --cpuset-cpus： 限制容器运行在指定的CPU核心
- CPU周期控制：--cpu-period、--cpu-quota
    - Linux 通过 CFS（Completely Fair Scheduler，完全公平调度器）来调度各个进程对 CPU 的使用。CFS 默认的调度周期是 100ms。
    - --cpu-period 设置每个容器进程的CFStime
    - -cpu-quota 设置在CFStime内容器能使用的 CPU 时间
    - 单位时us
### 限制内存
-m 或 --memory：设置内存的使用限额，例如：100MB，2GB。
--memory-swap：设置内存+swap的使用限额。
    - 如果在启动容器时，只指定-m而不指定--memory-swap， 那么--memory-swap默认为-m的两倍
### 对磁盘IO进行限制
- docker 可通过设置权重、限制 bps 和 iops 的方式控制容器读写磁盘的带宽
    - bps 是 byte per second，表示每秒读写的数据量。
    - iops 是 io per second，表示每秒的输入输出量(或读写次数)。
- --blkio-weight 与 --cpu-shares 类似，设置的是相对权重值，默认为 500。在下面的例子中，container_A 读写磁盘的带宽是 container_B 的两倍。
```
docker run -it --name container_A --blkio-weight 600 ubuntu
docker run -it --name container_B --blkio-weight 300 ubuntu
```
### 在Docker中使用GPU
- 如果Docker要使用GPU，需要docker支持GPU，在docker19以前都需要单独下载nvidia-docker1或nvidia-docker2来启动容器，但是docker19中后需要GPU的Docker只需要加个参数-–gpus即可(-–gpus all表示使用所有的gpu；要使用2个gpu：–-gpus 2即可；也可直接指定使用哪几个卡：--gpus '"device=1,2"')
path:Docker/learn_Docker.md
# docker 解决的最核心的问题就是应用的打包，容器本身没有价值，有价值的是应用的打包
# 容器只是一个特殊一点的进程而已。
    - 容器是一个单进程模型。一个容器的本质是一个进程。用户的应用进程实际上就是容器里 PID=1 的进程，也是其他后续创建的所有进程的父进程。这就意味着，在一个容器中，你没办法同时运行两个不同的应 用。这是因为容器本身的设计，就是希望容器和应用能够同生命周期，这个概念对后续的容器编排非常重要。否则，一旦出现类似于“容器是正常运行的，但是里面的应用早已经挂了”的情况，编排系统处理起来就非常麻烦了 
# docker和虚拟机的区别
- docker容器不是一个虚拟机，并没有一个所谓的docker容器运行在宿主机上，用户进程还是那个用户进程，只不过docker帮我们加上了各种namespace参数。Docker 项目在这里扮演的角 色，更多的是旁路式的辅助和管理工作。
- 也可以看到docker和虚拟机相比的区别，虚拟化需要一个Hypervisor 来负责创建虚拟机，这个虚拟机是真实存在的，并且他里面真的要运行一个操作系统。而容器本质上仍然仅仅是宿主机操作系统上的一个进程而已。
# 准备和使用
### 产品和版本、存储引擎
- docker版本、docker-ce/docker-ee和docker-io的区别
    - link：https://blog.csdn.net/zsy_1991/article/details/90261419
    - docker-io/docker-engine是docker的早期版本、默认centos7 安装的是docker-io、版本号是1.x，docker-io的最新版本是1.13; docker-ce是新的版本分为社区版docker-ce(Docker Community Edition)和企业版docker-ee(Docker Enterprise Edition)，版本号是17.x，最新的版本是17.12
    - docker-ce 和docker-ee 的可用版本是根据year-month 来的
    - 检查是哪个版本？
        - [docker 版本号说明](https://www.cnblogs.com/lcword/p/14478791.html)
        - `docker -v` 看版本号，Docker-ce 在 17.03 版本之前叫 Docker-engine/docker-io
### 安装docker-ce
    - 参考https://mirrors.huaweicloud.com/home
### 简单命令
- 有的image用`docker run -d image_name /bin/sh` 会立马Existed，而使用`docker run -it image_name [/bin/sh]` 能进入docker，但是推出后容器也会跟着推出，这时可以使用`docker run -itd image_nume [/bin/sh] `选项
- 查看本机所有容器的Docker的服务 `docker ps -a # 可以看到我们刚刚起来的hello-world`
- 拉取指定版本的镜像`docker pull ubuntu:17.10 # 不指定版本就拉取最新的`
- 查看本地的所有镜像`docker images`
- 以指定版本的镜像启动一个容器`docker run image[:version] # 如果目标镜像没有，就会去dockerhub拉取，就像最初运行hello-world一样`
- 下午搬服务器，后续更新
#### [docker19.03限制容器使用的内存资源](https://www.cnblogs.com/architectforest/p/12586336.html)
### 镜像
- Simple Tags and Shared Tags [link](https://github.com/docker-library/faq#whats-the-difference-between-shared-and-simple-tags)
- #### 镜像管理
    - 删除镜像 `docker rmi hello-world`
    - 搜索镜像 `docker search httpd`
### 脚本
- 批量删除命令 `docker ps -a|tail -n +2|head -n 1|awk '{print $1}'|xargs -i docker rm {}`
- 删除所有的exited的container `docker ps -a|grep -w Exited|awk '{print $1}'|xargs -i docker rm {}`

### 问题
- docker也可虚拟化？
- 这里的 交付
    > Docker 是一个用于开发，交付和运行应用程序的开放平台
- 使用alpine 基础镜像build新镜像的时候出错
    ```bash
    # Dockerfile
    FROM python:3.7-alpine
    ...
    RUN pip install --no-cache-dir -r requirements.txt

    # requirements.txt
    django==2.2.13
    beautifulsoup4==4.8.2
    lxml
    pillow==6.2.2
    pyppeteer==0.0.25
    loguru==0.4.1
    djangorestframework==3.9.1

    # 错误日志
    [root@localhost MrDoc]# docker build -t mrdoc-python-3.7-alpine:test .
    Sending build context to Docker daemon  49.43MB
    Step 1/6 : FROM python:3.7-alpine
    ---> e449f233097e
    Step 2/6 : WORKDIR /usr/src/app
    ---> Running in 0f5653a61c4b
    Removing intermediate container 0f5653a61c4b
    ---> c4a37bec703b
    Step 3/6 : RUN mkdir -p ~/.pip
    ---> Running in cfa91edb7415
    Removing intermediate container cfa91edb7415
    ---> c404b5f81e44
    Step 4/6 : RUN echo -e '[global]\nindex-url = https://repo.huaweicloud.com/repository/pypi/simple\ntrusted-host = repo.huaweicloud.com\nproxy = http://ptaishanpublic2:Huawei123@90.90.64.10:8080\ntimeout = 120' > ~/.pip/pip.conf
    ---> Running in 169d4a621d62
    Removing intermediate container 169d4a621d62
    ---> 8a4fbb86c6d6
    Step 5/6 : COPY requirements.txt ./
    ---> 6cb507d584cf
    Step 6/6 : RUN pip install --no-cache-dir -r requirements.txt
    ---> Running in 6241095809e0
    Looking in indexes: https://repo.huaweicloud.com/repository/pypi/simple
    Collecting django==2.2.13
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/fb/e1/c5520a00ae75060b0c03eea0115b272d6dc5dbd2fd3b75d0c0fbc9d262bc/Django-2.2.13-py3-none-any.whl (7.5 MB)
    Collecting beautifulsoup4==4.8.2
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/cb/a1/c698cf319e9cfed6b17376281bd0efc6bfc8465698f54170ef60a485ab5d/beautifulsoup4-4.8.2-py3-none-any.whl (106 kB)
    Collecting lxml
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/c5/2f/a0d8aa3eee6d53d5723d89e1fc32eee11e76801b424e30b55c7aa6302b01/lxml-4.6.1.tar.gz (3.2 MB)
        ERROR: Command errored out with exit status 1:
        command: /usr/local/bin/python -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-0qbseg16/lxml/setup.py'"'"'; __file__='"'"'/tmp/pip-install-0qbseg16/lxml/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-pip-egg-info-pvlkcnot
            cwd: /tmp/pip-install-0qbseg16/lxml/
        Complete output (3 lines):
        Building lxml version 4.6.1.
        Building without Cython.
        Error: Please make sure the libxml2 and libxslt development packages are installed.
        ----------------------------------------
    ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.
    WARNING: You are using pip version 20.2.3; however, version 20.2.4 is available.
    You should consider upgrading via the '/usr/local/bin/python -m pip install --upgrade pip' command.
    The command '/bin/sh -c pip install --no-cache-dir -r requirements.txt' returned a non-zero code: 1

    ```
    而如果使用python:3.7 作为基础镜像就可以build 成功
    ```bash
    # 日志的关键区别
    Step 6/6 : RUN pip install --no-cache-dir -r requirements.txt
    ---> Running in 6fed28fe1833
    Looking in indexes: https://repo.huaweicloud.com/repository/pypi/simple
    Collecting django==2.2.13
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/fb/e1/c5520a00ae75060b0c03eea0115b272d6dc5dbd2fd3b75d0c0fbc9d262bc/Django-2.2.13-py3-none-any.whl (7.5 MB)
    Collecting beautifulsoup4==4.8.2
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/cb/a1/c698cf319e9cfed6b17376281bd0efc6bfc8465698f54170ef60a485ab5d/beautifulsoup4-4.8.2-py3-none-any.whl (106 kB)
    Collecting lxml
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/9f/b1/b42206e8ad5e2180988fa97691f4d0db761775c5ce89e48d7b70e6f90c3a/lxml-4.6.1-cp37-cp37m-manylinux2014_aarch64.whl (6.7 MB)
    Collecting pillow==6.2.2
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/b3/d0/a20d8440b71adfbf133452d4f6e0fe80de2df7c2578c9b498fb812083383/Pillow-6.2.2.tar.gz (37.8 MB)
    Collecting pyppeteer==0.0.25
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/b0/16/a5e8d617994cac605f972523bb25f12e3ff9c30baee29b4a9c50467229d9/pyppeteer-0.0.25.tar.gz (1.2 MB)
    Collecting loguru==0.4.1
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/b2/f4/2c8b94025c6e30bdb331c7ee628dc152771845aedff35f0365c2a4dacd42/loguru-0.4.1-py3-none-any.whl (54 kB)
    Collecting djangorestframework==3.9.1
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/ef/13/0f394111124e0242bf3052c5578974e88e62e3715f0daf76b7c987fc6705/djangorestframework-3.9.1-py2.py3-none-any.whl (950 kB)
    Collecting pytz
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/4f/a4/879454d49688e2fad93e59d7d4efda580b783c745fd2ec2a3adf87b0808d/pytz-2020.1-py2.py3-none-any.whl (510 kB)
    Collecting sqlparse
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/14/05/6e8eb62ca685b10e34051a80d7ea94b7137369d8c0be5c3b9d9b6e3f5dae/sqlparse-0.4.1-py3-none-any.whl (42 kB)
    Collecting soupsieve>=1.2
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/6f/8f/457f4a5390eeae1cc3aeab89deb7724c965be841ffca6cfca9197482e470/soupsieve-2.0.1-py3-none-any.whl (32 kB)
    Collecting pyee
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/0d/0a/933b3931107e1da186963fd9bb9bceb9a613cff034cb0fb3b0c61003f357/pyee-8.1.0-py2.py3-none-any.whl (12 kB)
    Collecting websockets
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/e9/2b/cf738670bb96eb25cb2caf5294e38a9dc3891a6bcd8e3a51770dbc517c65/websockets-8.1.tar.gz (58 kB)
    Collecting appdirs
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/3b/00/2344469e2084fb287c2e0b57b72910309874c3245463acd6cf5e3db69324/appdirs-1.4.4-py2.py3-none-any.whl (9.6 kB)
    Collecting urllib3
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/56/aa/4ef5aa67a9a62505db124a5cb5262332d1d4153462eb8fd89c9fa41e5d92/urllib3-1.25.11-py2.py3-none-any.whl (127 kB)
    Collecting tqdm
    Downloading https://repo.huaweicloud.com/repository/pypi/packages/bd/cf/f91813073e4135c1183cadf968256764a6fe4e35c351d596d527c0540461/tqdm-4.50.2-py2.py3-none-any.whl (70 kB)
    Building wheels for collected packages: pillow, pyppeteer, websockets
    Building wheel for pillow (setup.py): started
    Building wheel for pillow (setup.py): finished with status 'done'
    Created wheel for pillow: filename=Pillow-6.2.2-cp37-cp37m-linux_aarch64.whl size=1355447 sha256=44a63449355c4506dcdd75190a25f80a055448d654fe901f07aa8390659947ce
    Stored in directory: /tmp/pip-ephem-wheel-cache-lkjnksvr/wheels/01/aa/41/7147d3a49f839eb42d1eb303b3166ff03d003878bd09eec850
    Building wheel for pyppeteer (setup.py): started
    Building wheel for pyppeteer (setup.py): finished with status 'done'
    Created wheel for pyppeteer: filename=pyppeteer-0.0.25-py3-none-any.whl size=78362 sha256=cbf0c8ec2bbc0319f5d4611b953da182e0810e88bc6c613a4a987f99e6d50dd3
    Stored in directory: /tmp/pip-ephem-wheel-cache-lkjnksvr/wheels/49/22/73/2a03f5494665ad60988869b13538bf438c715f126b96e2ff9d
    Building wheel for websockets (setup.py): started
    Building wheel for websockets (setup.py): finished with status 'done'
    Created wheel for websockets: filename=websockets-8.1-cp37-cp37m-linux_aarch64.whl size=78914 sha256=60af95dca10ca9810c085c526f8011cdb9885c2f9430fd2f68cbe794693e6558
    Stored in directory: /tmp/pip-ephem-wheel-cache-lkjnksvr/wheels/d6/82/11/da033e89cc4669750b0ca4f6817724439b80193a8b76d68e01
    Successfully built pillow pyppeteer websockets
    Installing collected packages: pytz, sqlparse, django, soupsieve, beautifulsoup4, lxml, pillow, pyee, websockets, appdirs, urllib3, tqdm, pyppeteer, loguru, djangorestframework
    ```
    关键的原因是alpine在安装某些三方包的时候下载的是源码的压缩包(.tar.gz)，而编译这些包需要实现编译好他们的依赖
    详细信息[link](https://www.jianshu.com/p/ec789a088f1e)
### 错误
1.
    ```shell
    ➜  MrDoc-master  docker run -d --name mrdoc -p 10086:10086 jonnyan404/mrdoc-alpine
    7b09ea58fbffd0e3e8964fd7571c9d5dc1fcf9b349d2018be85050e37dc2bfed
    docker: Error response from daemon: OCI runtime create failed: container_linux.go:345: starting container process caused "process_linux.go:430: container init caused \"write /proc/self/attr/keycreate: permission denied\"": unknown.
    # 解决办法
    # 关闭 SELinux 及防火墙
    setenforce 0
    systemctl stop firewalld
    systemctl disable firewalld
    ```
2. [resolve](https://blog.csdn.net/sin_geek/article/details/86736417)
    ```shell
    [root@localhost ~]# docker rmi f19b575
    Error response from daemon: conflict: unable to delete f19b575222f7 (cannot be forced) - image has dependent child images
    # resolve
    ```
### 简单上手
- 参照 [link](https://support.huaweicloud.com/instg-kunpengcpfs/kunpengcpfs_03_0001.html)来安装docker
- 需要配置代理
    ```shell
    export http_proxy='http://ptaishanpublic2:Huawei123@90.90.64.10:8080'
    export https_proxy='http://ptaishanpublic2:Huawei123@90.90.64.10:8080'
    wget https://download.docker.com/linux/static/stable/aarch64/docker-18.09.8.tgz --no-check-certificate
    ```
- 一直都很顺利，直到<code>docker run hello-world </code> 会提示错误<code>connection refused</code>
- 因为环境的差异性，需要额外配置一些东西去设置docker代理
    ```shell
    mkdir –p /etc/systemd/system/docker.service.d
    touch /etc/systemd/system/docker.service.d/https-proxy.conf
    # 文件内容如下 （分行）
    [Service]
    Environment="HTTP_PROXY=http://ptaishanpublic2:Huawei123@90.90.64.10:8080"
    "HTTPS_PROXY=http://ptaishanpublic2:Huawei123@90.90.64.10:8080"
    "NO_PROXY=localhost,127.0.0.1,docker-registry.example.com,"
    # 重启docker服务
    systemctl daemon-reload
    systemctl restart docker
    ```


### 90.90.67.14已运行的容器
```
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
00df1c32014d        vmware/nginx-photon:1.11.13        "nginx -g 'daemon of…"   2 weeks ago         Up 2 weeks          0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
4d1c5817bc4e        vmware/harbor-jobservice:v1.2.2    "/harbor/harbor_jobs…"   2 weeks ago         Up 2 weeks                                                                             harbor-jobservice
1964fd917292        vmware/harbor-ui:v1.2.2            "/harbor/harbor_ui"      2 weeks ago         Up 2 weeks                                                                             harbor-ui
e30fe12489ce        vmware/registry:2.6.2-photon       "/entrypoint.sh serv…"   2 weeks ago         Up 2 weeks          5000/tcp                                                           registry
c507f4dd0bfb        vmware/harbor-db:v1.2.2            "docker-entrypoint.s…"   2 weeks ago         Up 2 weeks          3306/tcp                                                           harbor-db
c5d3e428e0e6        vmware/harbor-adminserver:v1.2.2   "/harbor/harbor_admi…"   2 weeks ago         Up 2 weeks                                                                             harbor-adminserver
101cbd32c45b        vmware/harbor-log:v1.2.2           "/bin/sh -c 'crond &…"   2 weeks ago         Up 2 weeks          127.0.0.1:1514->514/tcp                                            harbor-log
[root@localhost ~]#
```
path:Docker/learn_Dockerfile.md
# link:
    - https://www.runoob.com/docker/docker-dockerfile.html
# 问题
- 怎么from 本地镜像
# 简单构建一个镜像
```
# vim Dockerfile
From 镜像名 # 
```
# 设置镜像源
path:other/learn_Linux_Cgroup&namespace.md
# link:
- https://mp.weixin.qq.com/s?__biz=Mzg2NTAyNTc5NQ==&mid=2247484441&idx=1&sn=ae2421fd803b911eb8670891cf90a677&chksm=ce612e75f916a763a3c46053b36aa89c49702064fa568db50e26605cff2c4a60db36037d0c44&scene=27
## namespace
- Linux 使用namespace技术来实现进程隔离，这样进程就只能看到分配给自己的资源
- #### 使用：
    - linux中船舰一个新的进程 `int pid = clone(main_function, stack_size, SIGCHLD, NULL);`
    - 如果在创建时可以传一个参数CLONE_NEWPID `int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL);` 这样这个新创建的进程就会看到一个隔离的命名空间，这个命名空间里，该进程的pid是1，Docker也使用这个技术来实现不同容器间的资源隔离.

- 为了隔离不同类型的资源，Linux 内核里面实现了以下几种不同类型的 namespace。
    - UTS，对应的宏为 CLONE_NEWUTS，表示不同的 namespace 可以配置不同的 hostname。
    - User，对应的宏为 CLONE_NEWUSER，表示不同的 namespace 可以配置不同的用户 和组。
    - Mount，对应的宏为 CLONE_NEWNS，表示不同的 namespace 的文件系统挂载点是 隔离的
    - PID，对应的宏为 CLONE_NEWPID，表示不同的 namespace 有完全独立的 pid，也即 一个 namespace 的进程和另一个 namespace 的进程里可以有一样的pid，但是代表不同的进程。
    - Network，对应的宏为 CLONE_NEWNET，表示不同的 namespace 有独立的网络协议 栈
## cgroup
- linux的实现方式是在一个特定的目录下有特定的配置文件。/sys/fs/cgroup/
path:other/doc/learn_vps.md
- nameslio设置域名解析：https://www.zhudc.com/website/2413
path:Readme.md M 42
- flink: 文件的引用
path:Flask/learn_Flask.md 28
## 钩子函数 又叫请求扩展