# Docker 常用指令笔记

这份笔记用于日常快速查命令。想理解每个概念的来龙去脉，可以看 `README_CN.md`；想马上操作，就看这份。

## 0. 安装和服务检查

查看 Docker 是否安装：

```bash
docker --version
docker compose version
docker buildx version
```

查看 Docker 服务状态：

```bash
sudo systemctl status docker
```

启动、停止、重启 Docker 服务：

```bash
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker
```

设置开机自启：

```bash
sudo systemctl enable docker
```

运行测试容器：

```bash
docker run hello-world
```

## 1. 容器 container

查看正在运行的容器：

```bash
docker ps
```

查看所有容器：

```bash
docker ps -a
```

运行一个容器：

```bash
docker run IMAGE
```

交互式进入 Ubuntu 容器：

```bash
docker run -it ubuntu:24.04 bash
```

给容器命名：

```bash
docker run -it --name my-ubuntu ubuntu:24.04 bash
```

后台运行容器：

```bash
docker run -d --name my-nginx nginx
```

进入正在运行的容器：

```bash
docker exec -it CONTAINER bash
```

如果容器里没有 `bash`，试试：

```bash
docker exec -it CONTAINER sh
```

查看容器日志：

```bash
docker logs CONTAINER
docker logs -f CONTAINER
```

停止、启动、重启容器：

```bash
docker stop CONTAINER
docker start CONTAINER
docker restart CONTAINER
```

删除已停止容器：

```bash
docker rm CONTAINER
```

强制删除运行中的容器：

```bash
docker rm -f CONTAINER
```

查看容器详细信息：

```bash
docker inspect CONTAINER
```

查看容器资源占用：

```bash
docker stats
```

## 2. 镜像 image

拉取镜像：

```bash
docker pull IMAGE:TAG
```

示例：

```bash
docker pull python:3.12-slim
docker pull ubuntu:24.04
docker pull nginx:latest
```

查看本地镜像：

```bash
docker images
```

查看镜像详细信息：

```bash
docker image inspect IMAGE:TAG
```

删除镜像：

```bash
docker rmi IMAGE:TAG
```

给镜像打标签：

```bash
docker tag OLD_IMAGE:TAG NEW_IMAGE:TAG
```

构建镜像：

```bash
docker build -t IMAGE_NAME:TAG .
```

示例：

```bash
docker build -t flask-demo:dev .
```

不使用缓存重新构建：

```bash
docker build --no-cache -t IMAGE_NAME:TAG .
```

## 3. 端口映射

把宿主机 8080 端口映射到容器 80 端口：

```bash
docker run -d --name web -p 8080:80 nginx
```

访问：

```bash
curl http://localhost:8080
```

记忆格式：

```text
-p 宿主机端口:容器端口
```

常见例子：

```bash
docker run --rm -p 8000:8000 flask-demo:dev
docker run --rm -p 3000:3000 node-demo:dev
docker run --rm -p 8080:80 nginx
```

## 4. 目录挂载和数据卷

把当前目录挂载到容器 `/app`：

```bash
docker run --rm -it -v "$PWD":/app -w /app python:3.12-slim bash
```

运行当前目录里的 Python 文件：

```bash
docker run --rm -it -v "$PWD":/app -w /app python:3.12-slim python app.py
```

创建 volume：

```bash
docker volume create my-data
```

查看 volume：

```bash
docker volume ls
```

把 volume 挂载到容器：

```bash
docker run -it -v my-data:/data ubuntu:24.04 bash
```

查看 volume 详细信息：

```bash
docker volume inspect my-data
```

删除 volume：

```bash
docker volume rm my-data
```

清理未使用的 volume：

```bash
docker volume prune
```

注意：删除 volume 可能删除数据库和项目数据，执行前先确认。

## 5. Dockerfile 构建常用命令

最小 Python 示例：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

构建：

```bash
docker build -t my-app:dev .
```

运行：

```bash
docker run --rm my-app:dev
```

带端口运行：

```bash
docker run --rm -p 8000:8000 my-app:dev
```

带目录挂载运行：

```bash
docker run --rm -it -v "$PWD":/app -w /app my-app:dev bash
```

## 6. Docker Compose

启动当前目录 `compose.yaml` 里的服务：

```bash
docker compose up
```

后台启动：

```bash
docker compose up -d
```

重新构建并启动：

```bash
docker compose up --build
```

查看 Compose 服务状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs
docker compose logs -f
```

进入某个服务容器：

```bash
docker compose exec SERVICE bash
```

如果没有 `bash`：

```bash
docker compose exec SERVICE sh
```

停止并删除 Compose 创建的容器和网络：

```bash
docker compose down
```

停止并连 volume 一起删除：

```bash
docker compose down -v
```

注意：`docker compose down -v` 可能删除数据库数据，新手不要随手用。

指定 compose 文件：

```bash
docker compose -f compose.yaml up
docker compose -f compose.dev.yaml up
```

## 7. 网络 network

查看网络：

```bash
docker network ls
```

创建网络：

```bash
docker network create my-net
```

让容器加入网络：

```bash
docker run -d --name web --network my-net nginx
```

查看网络详细信息：

```bash
docker network inspect my-net
```

删除网络：

```bash
docker network rm my-net
```

清理未使用网络：

```bash
docker network prune
```

## 8. 清理命令

查看 Docker 磁盘占用：

```bash
docker system df
```

清理停止的容器、无用网络、悬空镜像和构建缓存：

```bash
docker system prune
```

清理更多未使用镜像：

```bash
docker system prune -a
```

清理构建缓存：

```bash
docker builder prune
```

清理未使用镜像：

```bash
docker image prune
```

清理所有未被容器使用的镜像：

```bash
docker image prune -a
```

清理未使用 volume：

```bash
docker volume prune
```

谨慎使用清理命令，尤其是带 `-a` 和 `-v` 的命令。

## 9. 常见排错命令

权限错误：

```bash
getent group docker || sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

端口占用：

```bash
sudo ss -ltnp | grep ':8000'
```

查看容器为什么退出：

```bash
docker ps -a
docker logs CONTAINER
docker inspect CONTAINER
```

检查容器内文件：

```bash
docker exec -it CONTAINER ls /app
docker exec -it CONTAINER pwd
docker exec -it CONTAINER env
```

检查镜像是否存在：

```bash
docker images | grep IMAGE_NAME
```

检查 Compose 配置是否能解析：

```bash
docker compose config
```

## 10. 一眼记住的常用组合

临时跑一个命令，用完自动删容器：

```bash
docker run --rm IMAGE COMMAND
```

临时进入一个 Linux 环境：

```bash
docker run --rm -it ubuntu:24.04 bash
```

用容器运行当前目录代码：

```bash
docker run --rm -it -v "$PWD":/app -w /app IMAGE COMMAND
```

启动 Web 服务并映射端口：

```bash
docker run --rm -p 8000:8000 IMAGE
```

后台启动服务：

```bash
docker run -d --name NAME -p HOST_PORT:CONTAINER_PORT IMAGE
```

看日志并跟随输出：

```bash
docker logs -f CONTAINER
```

Compose 开发启动：

```bash
docker compose up --build
```

Compose 后台启动：

```bash
docker compose up -d
```

Compose 停止：

```bash
docker compose down
```

## 11. 危险命令提醒

下面这些命令会删除东西，执行前先看清楚：

```bash
docker rm -f CONTAINER
docker rmi IMAGE
docker volume rm VOLUME
docker volume prune
docker system prune
docker system prune -a
docker compose down -v
```

最容易误删数据的是：

```bash
docker volume prune
docker compose down -v
```

如果容器里跑的是数据库，先确认数据是否已经备份。
