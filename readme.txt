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