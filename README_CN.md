# 在 Ubuntu 中学习使用 Docker 搭建独立开发环境

这是一份面向 Linux 新手的 Docker 入门笔记。目标不是背命令，而是让你能在 Ubuntu 上正确安装 Docker，并用 Docker 为不同项目搭建互不污染、可重复创建的开发环境。

常用命令速查可以看：[Docker 常用指令笔记](./DOCKER_COMMANDS_CN.md)。

> 适用系统：Ubuntu 22.04 / 24.04 及较新的 Ubuntu 发行版。
>
> 推荐方式：安装 Docker Engine，而不是直接安装 Ubuntu 软件源里的旧版 `docker.io`。下面的安装步骤按 Docker 官方 apt 仓库方式整理。

## 0. 先理解 Docker 解决什么问题

在没有 Docker 时，不同项目可能要求不同版本的 Python、Node.js、CUDA、数据库或系统库。装在同一台机器上，很容易互相影响。

Docker 的核心思路是：

- `镜像 image`：一份打包好的环境模板，例如 `python:3.12-slim`。
- `容器 container`：镜像运行起来后的实例，可以理解成“正在运行的独立环境”。
- `Dockerfile`：描述如何制作镜像的文本文件。
- `volume`：把重要数据放到容器生命周期之外，避免删容器时数据丢失。
- `bind mount`：把宿主机目录挂进容器，适合开发时实时修改代码。
- `Docker Compose`：用一个 `compose.yaml` 管理多个容器，例如 Web 服务加数据库。

一句话记忆：

```text
Dockerfile 构建 image，image 运行成 container，volume 保存数据，Compose 管理一组 container。
```

## 1. 安装前检查系统

打开终端，先确认系统版本和 CPU 架构：

```bash
lsb_release -a
dpkg --print-architecture
```

常见架构：

- 普通 x86_64 电脑一般显示 `amd64`。
- 树莓派、部分 ARM 设备可能显示 `arm64`。

更新软件包索引：

```bash
sudo apt update
```

如果你之前安装过 Ubuntu 仓库里的 Docker、Podman 兼容包或旧版 containerd，建议先清理冲突包：

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

如果没有任何冲突包被安装，命令可能不会删除东西，这是正常的。

## 2. 使用官方 apt 仓库安装 Docker Engine

### 2.1 安装基础工具

```bash
sudo apt update
sudo apt install ca-certificates curl
```

### 2.2 添加 Docker 官方 GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 2.3 添加 Docker 官方 apt 源

下面这段命令会根据你的 Ubuntu 版本自动写入正确的发行版代号。

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

刷新软件包索引：

```bash
sudo apt update
```

### 2.4 安装 Docker

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

这些包分别负责：

- `docker-ce`：Docker Engine 主程序。
- `docker-ce-cli`：`docker` 命令行工具。
- `containerd.io`：容器运行时依赖。
- `docker-buildx-plugin`：新版镜像构建插件。
- `docker-compose-plugin`：新版 Compose 插件，命令是 `docker compose`，中间有空格。

## 3. 验证安装是否成功

检查 Docker 服务：

```bash
sudo systemctl status docker
```

如果没有启动：

```bash
sudo systemctl start docker
```

运行官方测试容器：

```bash
sudo docker run hello-world
```

看到类似 `Hello from Docker!` 的输出，就说明 Docker Engine 可以正常工作。

查看版本：

```bash
docker --version
docker compose version
docker buildx version
```

## 4. 让当前用户不用每次输入 sudo

默认情况下，Docker 的 Unix socket 由 root 管理，所以普通用户运行 `docker` 可能会遇到权限问题。可以把当前用户加入 `docker` 组：

```bash
getent group docker || sudo groupadd docker
sudo usermod -aG docker $USER
```

让当前终端立即尝试加载新组：

```bash
newgrp docker
```

再次验证：

```bash
docker run hello-world
```

如果仍然失败，注销并重新登录，或者重启系统后再试。

重要安全提醒：加入 `docker` 组后，当前用户基本拥有接近 root 的容器管理能力。个人开发机通常可以接受；多人服务器上要谨慎。

如果你之前用 `sudo docker` 生成过 `~/.docker` 配置，之后改用普通用户运行时可能出现权限错误。可以修复目录权限：

```bash
sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
sudo chmod g+rwx "$HOME/.docker" -R
```

## 5. 第一次使用：运行、查看、停止、删除容器

### 5.1 运行一个临时 Ubuntu 容器

```bash
docker run -it --name my-ubuntu ubuntu:24.04 bash
```

参数解释：

- `docker run`：创建并启动容器。
- `-it`：进入交互式终端。
- `--name my-ubuntu`：给容器起名，方便后续操作。
- `ubuntu:24.04`：使用的镜像名和标签。
- `bash`：容器启动后执行的命令。

进入容器后，你会看到一个新的 shell。试试：

```bash
cat /etc/os-release
pwd
ls
exit
```

退出后，容器会停止。

### 5.2 查看容器

查看正在运行的容器：

```bash
docker ps
```

查看所有容器，包括已经停止的：

```bash
docker ps -a
```

### 5.3 再次启动并进入容器

```bash
docker start my-ubuntu
docker exec -it my-ubuntu bash
```

### 5.4 停止和删除容器

```bash
docker stop my-ubuntu
docker rm my-ubuntu
```

如果容器还在运行，也可以强制删除：

```bash
docker rm -f my-ubuntu
```

## 6. 镜像常用命令

拉取镜像：

```bash
docker pull python:3.12-slim
```

查看本地镜像：

```bash
docker images
```

删除镜像：

```bash
docker rmi python:3.12-slim
```

注意：如果某个镜像正在被容器使用，需要先删除相关容器。

## 7. 端口映射：让浏览器访问容器里的服务

容器默认和宿主机隔离。容器里启动了 Web 服务，不代表宿主机浏览器能直接访问。需要用 `-p` 发布端口：

```bash
docker run -d --name web -p 8080:80 nginx
```

含义：

```text
宿主机端口 8080 -> 容器端口 80
```

访问：

```bash
curl http://localhost:8080
```

或在浏览器打开：

```text
http://localhost:8080
```

清理：

```bash
docker rm -f web
```

## 8. 数据保存：volume 和 bind mount

### 8.1 容器文件默认不可靠

容器删除后，容器内部后续写入的文件通常也会消失。数据库、上传文件、训练结果等需要持久化的数据，不应该只放在容器内部。

### 8.2 使用 volume 保存数据

创建 volume：

```bash
docker volume create demo-data
```

运行容器并把 volume 挂到 `/data`：

```bash
docker run -it --name volume-demo -v demo-data:/data ubuntu:24.04 bash
```

在容器内写入文件：

```bash
echo "hello docker volume" > /data/hello.txt
exit
```

删除容器后重新挂载同一个 volume：

```bash
docker rm volume-demo
docker run --rm -it -v demo-data:/data ubuntu:24.04 cat /data/hello.txt
```

你仍然能看到之前写入的内容。

常用 volume 命令：

```bash
docker volume ls
docker volume inspect demo-data
docker volume rm demo-data
```

### 8.3 使用 bind mount 做开发

bind mount 是把宿主机目录直接挂进容器。开发时推荐使用，因为你可以在 VS Code 或其他编辑器中改代码，容器里立刻看到变化。

先创建一个练习目录：

```bash
mkdir -p ~/docker-labs/python-demo
cd ~/docker-labs/python-demo
printf 'print("hello from docker dev env")\n' > app.py
```

运行 Python 容器，并把当前目录挂到容器的 `/app`：

```bash
docker run --rm -it -v "$PWD":/app -w /app python:3.12-slim python app.py
```

参数解释：

- `--rm`：容器退出后自动删除，适合一次性任务。
- `-v "$PWD":/app`：把当前目录挂载进容器。
- `-w /app`：设置容器内工作目录。
- `python app.py`：在容器里运行代码。

## 9. 用 Dockerfile 固化开发环境

上面直接使用 `python:3.12-slim` 可以跑简单脚本，但真实项目通常要安装依赖。此时应该写 Dockerfile。

进入练习目录：

```bash
mkdir -p ~/docker-labs/flask-demo
cd ~/docker-labs/flask-demo
```

创建 `app.py`：

```bash
cat > app.py <<'EOF'
from flask import Flask

app = Flask(__name__)

@app.get("/")
def index():
    return "hello from flask in docker\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
EOF
```

创建 `requirements.txt`：

```bash
cat > requirements.txt <<'EOF'
flask==3.1.1
EOF
```

创建 `Dockerfile`：

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["python", "app.py"]
```

常见 Dockerfile 指令：

- `FROM`：选择基础镜像。
- `WORKDIR`：设置工作目录。
- `COPY`：复制文件到镜像。
- `RUN`：构建镜像时执行命令，常用于安装依赖。
- `EXPOSE`：声明容器内服务端口，主要是文档作用。
- `CMD`：容器启动时默认执行的命令。

构建镜像：

```bash
docker build -t flask-demo:dev .
```

运行容器：

```bash
docker run --rm -p 8000:8000 flask-demo:dev
```

另开一个终端验证：

```bash
curl http://localhost:8000
```

## 10. 添加 .dockerignore

构建镜像时，Docker 会把当前目录作为 build context 发送给构建器。项目越大，发送越慢。应该用 `.dockerignore` 排除不需要进入镜像的文件。

在项目目录创建 `.dockerignore`：

```text
.git
__pycache__
*.pyc
.venv
venv
.env
node_modules
dist
build
```

原则：

- 不把本地虚拟环境放进镜像。
- 不把密钥、token、`.env` 放进镜像。
- 不把构建产物和缓存放进镜像。

## 11. 用 Docker Compose 管理开发环境

当项目只有一个容器时，`docker run` 还能接受。项目需要 Web 服务、数据库、缓存时，手写一长串命令会很难维护。Compose 用一个 YAML 文件保存这些配置。

继续使用 Flask 示例，创建 `compose.yaml`：

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    environment:
      FLASK_ENV: development
```

启动：

```bash
docker compose up
```

后台启动：

```bash
docker compose up -d
```

查看日志：

```bash
docker compose logs -f
```

停止并删除容器、网络：

```bash
docker compose down
```

重新构建并启动：

```bash
docker compose up --build
```

Compose 的好处：

- 配置可保存到项目仓库。
- 团队成员可以用同一份环境启动项目。
- 多容器项目更容易管理。

## 12. 推荐的项目目录结构

一个简单 Python 项目可以这样组织：

```text
my-project/
├── app.py
├── requirements.txt
├── Dockerfile
├── compose.yaml
├── .dockerignore
└── README.md
```

更复杂的项目可以这样：

```text
my-project/
├── src/
├── tests/
├── requirements.txt
├── Dockerfile
├── compose.yaml
├── .dockerignore
└── README.md
```

## 13. 日常开发工作流

推荐你形成这个固定流程：

1. 新项目先写 `Dockerfile`，明确基础镜像和依赖。
2. 写 `.dockerignore`，避免把无关文件打进镜像。
3. 用 `docker build -t 项目名:dev .` 构建镜像。
4. 用 `docker run --rm ...` 做单容器测试。
5. 如果需要数据库、缓存或多个服务，改用 `compose.yaml`。
6. 每次改依赖后重新构建镜像。
7. 每次改业务代码时优先用 bind mount，提高迭代速度。

## 14. 常用命令速查

### 容器

```bash
docker ps
docker ps -a
docker run IMAGE
docker run -it IMAGE bash
docker run -d --name NAME IMAGE
docker exec -it NAME bash
docker logs NAME
docker logs -f NAME
docker stop NAME
docker start NAME
docker restart NAME
docker rm NAME
docker rm -f NAME
```

### 镜像

```bash
docker images
docker pull IMAGE
docker build -t NAME:TAG .
docker rmi IMAGE
docker image prune
```

### 数据卷

```bash
docker volume ls
docker volume create NAME
docker volume inspect NAME
docker volume rm NAME
docker volume prune
```

### 网络

```bash
docker network ls
docker network inspect NAME
docker network prune
```

### Compose

```bash
docker compose up
docker compose up -d
docker compose up --build
docker compose ps
docker compose logs -f
docker compose exec SERVICE bash
docker compose down
docker compose down -v
```

注意：`docker compose down -v` 会删除 Compose 创建的 volume，数据库数据可能一起被删。新手不要随手加 `-v`。

## 15. 常见问题排查

### 15.1 permission denied while trying to connect to Docker daemon socket

原因：当前用户没有权限访问 Docker socket。

处理：

```bash
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

如果仍失败，注销重新登录或重启系统。

### 15.2 端口被占用

错误里如果出现 `port is already allocated`，说明宿主机端口已经被占用。

查看谁占用了 8000：

```bash
sudo ss -ltnp | grep ':8000'
```

换一个宿主机端口，例如：

```bash
docker run --rm -p 8001:8000 flask-demo:dev
```

### 15.3 镜像下载很慢或失败

可能原因：

- 网络不稳定。
- Docker Hub 访问受限。
- 公司或学校网络需要代理。

先确认网络：

```bash
docker pull hello-world
```

如果你使用代理，需要分别配置 Docker CLI 或 Docker daemon 的代理。配置前先确认你所在网络的代理地址，不要随便复制网上未知代理。

### 15.4 磁盘空间越来越少

查看 Docker 占用：

```bash
docker system df
```

清理停止的容器、无用网络、悬空镜像和构建缓存：

```bash
docker system prune
```

更激进的清理：

```bash
docker system prune -a
```

注意：`-a` 会删除所有当前没被容器使用的镜像，之后可能需要重新下载。

### 15.5 修改代码后容器里没有变化

检查你是否使用了 bind mount：

```bash
docker run --rm -it -v "$PWD":/app -w /app python:3.12-slim ls
```

如果你是把代码 `COPY` 进镜像，则修改代码后需要重新构建镜像：

```bash
docker build -t flask-demo:dev .
```

## 16. 新手容易混淆的点

### 16.1 Dockerfile 里的 RUN 和 docker run 不是一回事

- `RUN`：构建镜像时执行，结果写入镜像层。
- `docker run`：启动容器时执行。

### 16.2 EXPOSE 不等于发布端口

`EXPOSE 8000` 只是声明容器内部服务端口。真正让宿主机访问容器，需要：

```bash
docker run -p 8000:8000 IMAGE
```

### 16.3 镜像不是容器

- 删除容器：`docker rm`
- 删除镜像：`docker rmi`

容器是运行实例，镜像是模板。

### 16.4 容器不是虚拟机

容器通常只运行一个主要进程。不要把它当成完整桌面系统或长期手工维护的机器。正确做法是把环境写进 Dockerfile，需要时重新构建。

## 17. 一个完整练习任务

请你按顺序完成下面练习：

1. 安装 Docker Engine。
2. 运行 `docker run hello-world`。
3. 运行 `docker run -it ubuntu:24.04 bash` 并查看 `/etc/os-release`。
4. 运行 `nginx` 并通过浏览器访问 `http://localhost:8080`。
5. 使用 volume 保存一个文本文件，删除容器后确认文件还在。
6. 创建 Flask 示例项目。
7. 编写 `Dockerfile`。
8. 构建 `flask-demo:dev` 镜像。
9. 运行容器并通过 `curl http://localhost:8000` 验证。
10. 编写 `compose.yaml`，用 `docker compose up` 启动项目。

完成后，你就已经掌握了 Docker 独立开发环境的最小闭环。

## 18. 继续学习路线

建议按这个顺序继续：

1. Dockerfile 最佳实践：缓存、非 root 用户、多阶段构建。
2. Docker Compose：服务依赖、环境变量、数据库 volume。
3. 镜像仓库：给镜像打 tag，推送到 Docker Hub 或私有镜像仓库。
4. 开发环境模板化：为 Python、Node.js、ROS、CUDA 分别准备基础模板。
5. 安全习惯：不要把密钥写进镜像，不要随便挂载宿主机敏感目录。

## 19. 官方资料

- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Linux post-installation steps: https://docs.docker.com/engine/install/linux-postinstall/
- Publishing and exposing ports: https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/
- Persisting container data: https://docs.docker.com/get-started/docker-concepts/running-containers/persisting-container-data/
- Writing a Dockerfile: https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/
- Compose file reference: https://docs.docker.com/reference/compose-file/
