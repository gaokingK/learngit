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