以下内容可直接保存为 `Docker-使用手册.md`，使用 Typora、VSCode、Obsidian 等工具打开，并可导出为 PDF/HTML。

---

# Docker 系统使用手册

| 文档信息 | 内容                  |
| ---- | ------------------- |
| 文档编号 | DOCKER-MAN-001      |
| 文档版本 | 1.0                 |
| 适用版本 | Docker Engine 27.x  |
| 更新日期 | 2026-09-24          |
| 文档状态 | 正式版                 |
| 密级   | 内部公开                |
| 适用读者 | 开发、测试、运维、DevOps 工程师 |

---

## 修订记录

| 版本  | 日期         | 修订人   | 修订说明                                                  |
| --- | ---------- | ----- | ----------------------------------------------------- |
| 1.0 | 2026-09-24 | 运维平台组 | 初版发布，覆盖环境、镜像、容器、网络、卷、Dockerfile、Compose、Swarm、资源限制与清理 |

---

## 目录

1. 环境信息与系统管理  
2. 镜像管理  
3. 容器生命周期管理  
4. 网络管理  
5. 数据卷管理  
6. Dockerfile 指令详解  
7. Docker Compose  
8. Swarm 集群管理  
9. 资源限制速查  
10. 常用组合与清理  
    附录 A 全局选项  
    附录 B 命令速查总表  
    附录 C 场景卡片模板  

---

## 文档约定

| 约定项     | 说明                                           |
| ------- | -------------------------------------------- |
| `<...>` | 表示占位符，需要替换为实际值，如 `<容器名>`、`<镜像名>`             |
| 场景卡片    | 每个核心场景统一包含：目的、前置条件、操作方法、输出示例、输出解释、验证方法、清理/注意 |
| 输出示例    | 仅作参考，实际输出可能因版本、系统、网络不同而略有差异                  |
| 警告标记    | ⚠️ 表示高危操作，执行前必须确认                            |
| 命令提示符   | 默认使用 `$` 表示宿主机终端，`#` 表示容器内 root 终端           |

---

# 1. 环境信息与系统管理

## 1.1 命令速查

| 命令                     | 作用              | 示例                                  |
| ---------------------- | --------------- | ----------------------------------- |
| `docker version`       | 查看客户端与服务端版本     | `docker version`                    |
| `docker info`          | 查看 Docker 系统信息  | `docker info`                       |
| `docker system info`   | 同 `docker info` | `docker system info`                |
| `docker system df`     | 查看磁盘占用          | `docker system df`                  |
| `docker system df -v`  | 查看磁盘占用明细        | `docker system df -v`               |
| `docker system events` | 实时查看 Docker 事件  | `docker system events`              |
| `docker system prune`  | 清理无用资源          | `docker system prune -af --volumes` |

---

## 1.2 场景：验证 Docker 环境是否正常

**目的**  
确认 Docker 客户端、服务端是否正常工作，API 版本是否兼容，排查“命令能执行但连不上 Docker daemon”的问题。

**前置条件**  
Docker 已安装，当前用户有权限访问 Docker daemon。

**操作方法**

```bash
docker version
```

**输出示例**

```text
Client: Docker Engine - Community
 Version:           27.0.3
 API version:       1.46
 Go version:        go1.21.11
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          27.0.3
  API version:      1.46 (minimum version 1.24)
  Go version:       go1.21.11
  OS/Arch:          linux/amd64
  Experimental:     false
```

**输出解释**

| 字段                                    | 含义                                |
| ------------------------------------- | --------------------------------- |
| `Client`                              | 当前使用的 Docker 命令行客户端               |
| `Server`                              | Docker daemon，真正执行容器操作的守护进程       |
| `API version`                         | 客户端与服务端通信的 API 版本                 |
| 只有 `Client` 没有 `Server`               | Docker daemon 未启动或当前用户无权限连接       |
| `Cannot connect to the Docker daemon` | Docker 服务未运行，或 `DOCKER_HOST` 配置错误 |

**验证是否达到目的**

```bash
docker version --format '{{.Server.Version}}'
```

输出示例：

```text
27.0.3
```

如果能看到服务端版本号，说明客户端已经能正常访问 Docker daemon。

**清理/注意**

普通用户权限不足时：

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 1.3 场景：查看 Docker 整体运行环境

**目的**  
查看存储驱动、CPU、内存、容器数量、镜像数量、Swarm 状态、Docker 根目录等核心信息。

**前置条件**  
Docker daemon 正在运行。

**操作方法**

```bash
docker system info
```

**输出示例**

```text
Containers: 8
 Running: 3
 Paused: 0
 Stopped: 5
Images: 12
Server Version: 27.0.3
Storage Driver: overlay2
 Logging Driver: json-file
 Cgroup Driver: systemd
 Swarm: inactive
 CPUs: 8
 Total Memory: 15.6GiB
 Name: prod-node-01
 Docker Root Dir: /var/lib/docker
```

**输出解释**

| 字段                      | 含义                  |
| ----------------------- | ------------------- |
| `Containers`            | 容器总数、运行数、停止数        |
| `Images`                | 本地镜像数量              |
| `Storage Driver`        | 存储驱动，常见为 `overlay2` |
| `Swarm`                 | 是否加入 Swarm 集群       |
| `CPUs` / `Total Memory` | 宿主机 CPU 与内存资源       |
| `Docker Root Dir`       | 镜像、容器、卷默认存储路径       |

**验证是否达到目的**

```bash
docker info --format 'ServerVersion={{.ServerVersion}} Driver={{.Driver}} Containers={{.Containers}} Images={{.Images}}'
```

输出示例：

```text
ServerVersion=27.0.3 Driver=overlay2 Containers=8 Images=12
```

能取到存储驱动、容器数、镜像数，说明环境信息读取正常。

---

## 1.4 场景：定位磁盘空间占用

**目的**  
磁盘空间不足时，快速判断是镜像、容器、数据卷还是构建缓存占用最多。

**前置条件**  
无。

**操作方法**

```bash
docker system df
```

**输出示例**

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          12        5         3.2GB     1.8GB (56%)
Containers      8         3         245MB     120MB (48%)
Local Volumes   5         2         1.1GB     500MB (45%)
Build Cache     0         0         0B        0B
```

**输出解释**

| 字段            | 含义       |
| ------------- | -------- |
| `TOTAL`       | 总数量      |
| `ACTIVE`      | 正在被使用的数量 |
| `SIZE`        | 总占用空间    |
| `RECLAIMABLE` | 可回收空间及占比 |

**验证是否达到目的**

清理后再次执行：

```bash
docker system df
```

如果 `RECLAIMABLE` 明显下降，说明清理生效。

**清理操作**

```bash
docker system prune -af --volumes
```

> ⚠️ **注意**  
> `--volumes` 会删除未被使用的数据卷，可能导致数据库数据丢失。生产环境执行前必须先执行 `docker system df -v` 确认。

---

# 2. 镜像管理

## 2.1 命令速查

| 命令                   | 作用      | 示例                                                    |
| -------------------- | ------- | ----------------------------------------------------- |
| `docker pull`        | 拉取镜像    | `docker pull nginx:1.25`                              |
| `docker images`      | 列出本地镜像  | `docker images`                                       |
| `docker build`       | 构建镜像    | `docker build -t myapp:1.0 .`                         |
| `docker tag`         | 给镜像打标签  | `docker tag myapp:1.0 registry.example.com/myapp:1.0` |
| `docker push`        | 推送镜像    | `docker push registry.example.com/myapp:1.0`          |
| `docker rmi`         | 删除镜像    | `docker rmi nginx:1.25`                               |
| `docker image prune` | 清理镜像    | `docker image prune -a`                               |
| `docker save`        | 导出镜像    | `docker save -o myapp.tar myapp:1.0`                  |
| `docker load`        | 导入镜像    | `docker load -i myapp.tar`                            |
| `docker history`     | 查看镜像层历史 | `docker history nginx:1.25`                           |
| `docker inspect`     | 查看镜像详情  | `docker inspect nginx:1.25`                           |

---

## 2.2 场景：拉取指定版本镜像

**目的**  
拉取固定版本镜像，避免使用 `latest` 导致版本漂移。

**前置条件**  
能访问 Docker Hub 或私有 Registry。

**操作方法**

```bash
docker pull nginx:1.25
```

**输出示例**

```text
1.25: Pulling from library/nginx
a2abf6c4d29d: Pull complete
...
Digest: sha256:...
Status: Downloaded newer image for nginx:1.25
docker.io/library/nginx:1.25
```

**输出解释**

| 输出                               | 含义                 |
| -------------------------------- | ------------------ |
| `Pulling from library/nginx`     | 从 Docker Hub 官方库拉取 |
| `Pull complete`                  | 每一层下载完成            |
| `Digest`                         | 镜像内容摘要，可用于固定版本     |
| `Status: Downloaded newer image` | 拉取成功               |

**验证是否达到目的**

```bash
docker images nginx:1.25
```

输出示例：

```text
REPOSITORY   TAG     IMAGE ID       CREATED       SIZE
nginx        1.25    a875bf1a1a1a   2 weeks ago   187MB
```

能查到镜像，说明拉取成功。

**参数场景**

```bash
docker pull --platform linux/arm64 nginx:1.25
```

目的：在 x86 机器上为 ARM 设备准备镜像。  
验证：

```bash
docker inspect nginx:1.25 -f '{{.Architecture}}'
```

输出 `arm64` 即成功。

---

## 2.3 场景：构建应用镜像

**目的**  
根据当前目录的 `Dockerfile` 构建应用镜像，用于测试或发布。

**前置条件**  
当前目录存在 `Dockerfile`，且构建上下文不要过大。

**操作方法**

```bash
docker build -t myapp:1.0 .
```

**输出示例**

```text
[+] Building 12.3s (10/10) FINISHED
 => [internal] load build definition from Dockerfile
 => [1/5] FROM node:20-alpine
 => [2/5] WORKDIR /app
 => [3/5] COPY package*.json ./
 => [4/5] RUN npm ci --only=production
 => [5/5] COPY . .
 => exporting to image
 => => naming to docker.io/library/myapp:1.0
```

**输出解释**

| 输出                                      | 含义                  |
| --------------------------------------- | ------------------- |
| `[1/5] FROM ...`                        | 每一步对应 Dockerfile 指令 |
| `CACHED`                                | 命中缓存，未重新执行          |
| `naming to docker.io/library/myapp:1.0` | 镜像构建并打标签成功          |

**验证是否达到目的**

```bash
docker run --rm myapp:1.0 node -v
```

输出示例：

```text
v20.11.0
```

能输出版本号，说明镜像可以正常启动。

**常用参数**

| 参数             | 说明            | 示例                                                     | 场景               |
| -------------- | ------------- | ------------------------------------------------------ | ---------------- |
| `-t`           | 指定镜像名和标签      | `docker build -t myapp:1.0 .`                          | 所有构建场景           |
| `-f`           | 指定 Dockerfile | `docker build -f Dockerfile.prod -t myapp:prod .`      | 多个 Dockerfile 并存 |
| `--build-arg`  | 传递构建参数        | `docker build --build-arg VERSION=2.0 -t myapp:2.0 .`  | 动态注入版本号          |
| `--no-cache`   | 不使用缓存         | `docker build --no-cache -t myapp:fresh .`             | 确保完全重新构建         |
| `--target`     | 指定多阶段目标       | `docker build --target builder -t myapp:builder .`     | 只构建到中间阶段         |
| `--cache-from` | 指定缓存来源        | `docker build --cache-from myapp:cache -t myapp:1.0 .` | CI/CD 远程缓存       |

---

## 2.4 场景：推送镜像到私有仓库

**目的**  
将本地构建的镜像推送到私有 Registry，供其他机器拉取。

**前置条件**  
已登录私有仓库，镜像已打上仓库地址标签。

**操作方法**

```bash
docker login registry.example.com
docker tag myapp:1.0 registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0
```

**输出示例**

```text
The push refers to repository [registry.example.com/myapp]
5f70bf18a086: Pushed
...
1.0: digest: sha256:... size: 1234
```

**输出解释**

| 输出                              | 含义       |
| ------------------------------- | -------- |
| `The push refers to repository` | 推送目标仓库   |
| `Pushed`                        | 层推送成功    |
| `digest`                        | 推送后的镜像摘要 |

**验证是否达到目的**

在另一台机器执行：

```bash
docker pull registry.example.com/myapp:1.0
```

能拉取成功，说明推送有效。

---

# 3. 容器生命周期管理

## 3.1 命令速查

| 命令               | 作用       | 示例                                               |
| ---------------- | -------- | ------------------------------------------------ |
| `docker run`     | 创建并运行容器  | `docker run -d --name web -p 8080:80 nginx:1.25` |
| `docker ps`      | 查看运行容器   | `docker ps -a`                                   |
| `docker start`   | 启动已停止容器  | `docker start web`                               |
| `docker stop`    | 优雅停止容器   | `docker stop web`                                |
| `docker restart` | 重启容器     | `docker restart web`                             |
| `docker kill`    | 强制停止容器   | `docker kill web`                                |
| `docker pause`   | 暂停容器     | `docker pause web`                               |
| `docker unpause` | 恢复容器     | `docker unpause web`                             |
| `docker rm`      | 删除容器     | `docker rm -f web`                               |
| `docker exec`    | 进入运行中容器  | `docker exec -it web /bin/bash`                  |
| `docker logs`    | 查看容器日志   | `docker logs -f web`                             |
| `docker inspect` | 查看容器详情   | `docker inspect web`                             |
| `docker stats`   | 查看资源使用   | `docker stats web`                               |
| `docker top`     | 查看容器内进程  | `docker top web`                                 |
| `docker cp`      | 容器与宿主机互拷 | `docker cp web:/etc/nginx/nginx.conf ./`         |
| `docker commit`  | 提交容器为新镜像 | `docker commit web my-nginx:custom`              |
| `docker export`  | 导出容器文件系统 | `docker export web -o web.tar`                   |
| `docker import`  | 导入容器文件系统 | `docker import web.tar my-nginx:imported`        |

---

## 3.2 场景：后台运行 Web 服务

**目的**  
后台运行 Nginx 容器，将宿主机 8080 端口映射到容器 80 端口，提供 Web 服务。

**前置条件**  
本地已有 `nginx:1.25` 镜像，8080 端口未被占用。

**操作方法**

```bash
docker run -d --name my-nginx -p 8080:80 nginx:1.25
```

**输出示例**

```text
c3f2a1b9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1
```

**输出解释**

| 参数/输出             | 含义               |
| ----------------- | ---------------- |
| 输出字符串             | 容器 ID            |
| `-d`              | 后台运行             |
| `--name my-nginx` | 容器名为 `my-nginx`  |
| `-p 8080:80`      | 宿主机 8080 → 容器 80 |
| `nginx:1.25`      | 使用的镜像            |

**验证是否达到目的**

```bash
docker ps --filter name=my-nginx
```

输出示例：

```text
CONTAINER ID   IMAGE        STATUS         PORTS                  NAMES
c3f2a1b9d8e7   nginx:1.25   Up 10 seconds  0.0.0.0:8080->80/tcp   my-nginx
```

再验证 HTTP：

```bash
curl -I http://localhost:8080
```

输出示例：

```text
HTTP/1.1 200 OK
Server: nginx/1.25.5
```

看到 `200 OK`，说明服务已通过 8080 端口对外提供。

**清理**

```bash
docker rm -f my-nginx
```

---

## 3.3 场景：交互式调试容器

**目的**  
临时进入 Ubuntu 容器，调试命令或测试环境，退出后自动删除。

**操作方法**

```bash
docker run -it --rm ubuntu:24.04 /bin/bash
```

**输出示例**

```text
root@a1b2c3d4e5f6:/#
```

**输出解释**

| 参数             | 含义        |
| -------------- | --------- |
| `-i`           | 保持标准输入    |
| `-t`           | 分配伪终端     |
| `--rm`         | 容器退出后自动删除 |
| `a1b2c3d4e5f6` | 容器 ID     |

**验证是否达到目的**

容器内执行：

```bash
cat /etc/os-release
```

输出示例：

```text
NAME="Ubuntu"
VERSION="24.04 LTS"
```

退出：

```bash
exit
```

检查是否自动删除：

```bash
docker ps -a | grep ubuntu
```

无输出，说明 `--rm` 生效。

---

## 3.4 场景：环境变量注入

**目的**  
启动 MySQL 时设置 root 密码和初始数据库。

**操作方法**

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=mydb \
  mysql:8.0
```

**输出示例**

```text
f1e2d3c4b5a697887766554433221100aabbccddeeff00112233445566778899
```

**输出解释**

| 参数                              | 含义               |
| ------------------------------- | ---------------- |
| `-e MYSQL_ROOT_PASSWORD=secret` | 设置 root 密码       |
| `-e MYSQL_DATABASE=mydb`        | 启动时创建 `mydb` 数据库 |
| 输出容器 ID                         | 容器启动成功           |

**验证是否达到目的**

等待几秒后执行：

```bash
docker exec -it mysql mysql -uroot -psecret -e "SHOW DATABASES;"
```

输出示例：

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mydb               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

看到 `mydb`，说明环境变量生效。

---

## 3.5 场景：数据卷持久化

**目的**  
把 MySQL 数据持久化到命名卷，防止删除容器后数据丢失。

**操作方法**

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
```

**输出示例**

```text
a1b2c3d4e5f6...
```

**输出解释**

| 参数                             | 含义                                |
| ------------------------------ | --------------------------------- |
| `-v mysql-data:/var/lib/mysql` | 命名卷 `mysql-data` 挂载到容器 MySQL 数据目录 |
| Docker 自动创建                    | 若卷不存在，会自动创建                       |

**验证是否达到目的**

```bash
docker volume inspect mysql-data
```

输出示例：

```json
[
    {
        "CreatedAt": "2026-09-24T10:00:00Z",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/mysql-data/_data",
        "Name": "mysql-data",
        "Scope": "local"
    }
]
```

进一步验证持久化：

```bash
docker exec mysql mysql -uroot -psecret -e "CREATE DATABASE persist_test;"
docker rm -f mysql
docker run -d --name mysql2 -e MYSQL_ROOT_PASSWORD=secret -v mysql-data:/var/lib/mysql mysql:8.0
docker exec mysql2 mysql -uroot -psecret -e "SHOW DATABASES;"
```

如果还能看到 `persist_test`，说明数据持久化成功。

---

## 3.6 场景：端口绑定指定 IP

**目的**  
只允许通过宿主机指定 IP `10.0.0.1` 的 8080 端口访问容器。

**操作方法**

```bash
docker run -d --name web -p 10.0.0.1:8080:80 nginx:1.25
```

**验证是否达到目的**

```bash
docker ps --filter name=web
```

输出示例：

```text
PORTS
10.0.0.1:8080->80/tcp
```

从本机访问：

```bash
curl -I http://10.0.0.1:8080
```

返回 `200 OK`，说明绑定指定 IP 成功。

---

## 3.7 场景：自动重启策略

**目的**  
让容器在异常退出或宿主机重启后自动恢复，但手动停止后不自动拉起。

**操作方法**

```bash
docker run -d --name web --restart unless-stopped -p 8080:80 nginx:1.25
```

**验证是否达到目的**

```bash
docker inspect web -f '{{.HostConfig.RestartPolicy.Name}}'
```

输出示例：

```text
unless-stopped
```

模拟异常退出：

```bash
docker kill web
sleep 3
docker ps --filter name=web
```

如果容器重新变为 `Up`，说明自动重启生效。

---

## 3.8 场景：限制容器资源

**目的**  
限制容器最多使用 2 个 CPU 核心和 512MB 内存，防止拖垮宿主机。

**操作方法**

```bash
docker run -d --name web --cpus=2 --memory=512m nginx:1.25
```

**验证是否达到目的**

```bash
docker inspect web -f 'CPU={{.HostConfig.NanoCpus}} MEM={{.HostConfig.Memory}}'
```

输出示例：

```text
CPU=2000000000 MEM=536870912
```

解释：

| 输出                    | 含义      |
| --------------------- | ------- |
| `NanoCpus=2000000000` | 2 个 CPU |
| `Memory=536870912`    | 512MB   |

再查看实时资源：

```bash
docker stats --no-stream web
```

输出示例：

```text
CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT
...            web    0.00%   3.5MiB / 512MiB
```

`LIMIT` 显示 `512MiB`，说明限制生效。

---

## 3.9 场景：查看容器日志

**目的**  
查看应用启动日志或错误日志，排查启动失败、请求异常。

**操作方法**

```bash
docker logs --tail 50 -f my-nginx
```

**输出示例**

```text
2026/09/24 10:00:00 [notice] 1#1: nginx/1.25.5
2026/09/24 10:00:00 [notice] 1#1: start worker processes
```

**输出解释**

| 参数          | 含义           |
| ----------- | ------------ |
| `--tail 50` | 只看最近 50 行    |
| `-f`        | 持续跟踪新日志      |
| `[notice]`  | Nginx 正常启动信息 |

**验证是否达到目的**

另开终端访问：

```bash
curl http://localhost:8080
```

回到日志终端，应看到新的访问日志：

```text
172.17.0.1 - - [24/Sep/2026:10:01:00 +0000] "GET / HTTP/1.1" 200 615
```

看到 `200`，说明日志实时输出正常。

---

## 3.10 场景：进入运行中容器

**目的**  
进入正在运行的容器，检查配置文件或执行诊断命令。

**操作方法**

```bash
docker exec -it my-nginx /bin/bash
```

若容器没有 bash：

```bash
docker exec -it my-nginx sh
```

**输出示例**

```text
root@c3f2a1b9d8e7:/#
```

**输出解释**

| 说明              | 含义                               |
| --------------- | -------------------------------- |
| `exec`          | 在容器内启动新进程                        |
| 退出后             | 容器继续运行                           |
| 不要用 `attach` 代替 | `attach` 连接主进程，`Ctrl+C` 可能导致容器停止 |

**验证是否达到目的**

容器内执行：

```bash
nginx -t
```

输出示例：

```text
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

说明配置正常，也说明已成功进入容器。

---

# 4. 网络管理

## 4.1 命令速查

| 命令                          | 作用      | 示例                                     |
| --------------------------- | ------- | -------------------------------------- |
| `docker network ls`         | 列出网络    | `docker network ls`                    |
| `docker network create`     | 创建网络    | `docker network create my-net`         |
| `docker network connect`    | 连接容器到网络 | `docker network connect my-net web`    |
| `docker network disconnect` | 断开容器    | `docker network disconnect my-net web` |
| `docker network inspect`    | 查看网络详情  | `docker network inspect my-net`        |
| `docker network rm`         | 删除网络    | `docker network rm my-net`             |
| `docker network prune`      | 清理未使用网络 | `docker network prune`                 |

---

## 4.2 场景：自定义网络与容器名互访

**目的**  
创建自定义 bridge 网络，让容器可以通过容器名互相访问，而不是依赖 IP。

**操作方法**

```bash
docker network create --driver bridge --subnet 172.19.0.0/16 --gateway 172.19.0.1 my-net
```

**输出示例**

```text
f0e1d2c3b4a5968778695a4b3c2d1e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d
```

**输出解释**

| 参数                       | 含义     |
| ------------------------ | ------ |
| `--driver bridge`        | 默认桥接网络 |
| `--subnet 172.19.0.0/16` | 自定义子网  |
| `--gateway 172.19.0.1`   | 网关地址   |
| 输出网络 ID                  | 创建成功   |

**验证是否达到目的**

```bash
docker network inspect my-net
```

输出中应包含：

```json
"Subnet": "172.19.0.0/16",
"Gateway": "172.19.0.1"
```

再验证容器名互访：

```bash
docker run -d --name web --network my-net nginx:1.25
docker run -it --rm --network my-net alpine ping -c 1 web
```

输出示例：

```text
PING web (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.085 ms
```

能 ping 通 `web`，说明自定义网络和 DNS 解析生效。

---

# 5. 数据卷管理

## 5.1 命令速查

| 命令                      | 作用      | 示例                              |
| ----------------------- | ------- | ------------------------------- |
| `docker volume create`  | 创建数据卷   | `docker volume create my-data`  |
| `docker volume ls`      | 列出数据卷   | `docker volume ls`              |
| `docker volume inspect` | 查看数据卷详情 | `docker volume inspect my-data` |
| `docker volume rm`      | 删除数据卷   | `docker volume rm my-data`      |
| `docker volume prune`   | 清理未使用卷  | `docker volume prune`           |

---

## 5.2 场景：创建并验证数据卷

**目的**  
创建一个命名卷，用于持久化数据库、日志等数据。

**操作方法**

```bash
docker volume create my-data
```

**输出示例**

```text
my-data
```

**输出解释**

输出卷名，表示创建成功。

**验证是否达到目的**

```bash
docker volume ls
```

输出示例：

```text
DRIVER    VOLUME NAME
local     my-data
```

查看详情：

```bash
docker volume inspect my-data
```

输出包含：

```json
"Mountpoint": "/var/lib/docker/volumes/my-data/_data"
```

说明卷已创建，并知道宿主机实际路径。

---

# 6. Dockerfile 指令详解

## 6.1 指令速查

| 指令            | 作用                | 示例                                                      |
| ------------- | ----------------- | ------------------------------------------------------- |
| `FROM`        | 指定基础镜像            | `FROM ubuntu:24.04`                                     |
| `RUN`         | 构建时执行命令           | `RUN apt-get update && apt-get install -y curl`         |
| `COPY`        | 复制文件到镜像           | `COPY app.py /app/`                                     |
| `ADD`         | 复制文件，支持 URL 和自动解压 | `ADD https://example.com/file.tar.gz /tmp/`             |
| `WORKDIR`     | 设置工作目录            | `WORKDIR /app`                                          |
| `ENV`         | 设置环境变量            | `ENV NODE_ENV=production`                               |
| `ARG`         | 构建时变量             | `ARG VERSION=1.0`                                       |
| `EXPOSE`      | 声明暴露端口            | `EXPOSE 8080`                                           |
| `CMD`         | 容器启动默认命令          | `CMD ["python", "app.py"]`                              |
| `ENTRYPOINT`  | 容器启动入口            | `ENTRYPOINT ["python"]`                                 |
| `VOLUME`      | 声明数据卷挂载点          | `VOLUME /data`                                          |
| `USER`        | 指定运行用户            | `USER appuser`                                          |
| `HEALTHCHECK` | 健康检查              | `HEALTHCHECK CMD curl -f http://localhost/ \|\| exit 1` |
| `LABEL`       | 添加元数据             | `LABEL maintainer="dev@example.com"`                    |

---

## 6.2 场景：CMD 与 ENTRYPOINT 组合

**目的**  
让容器有固定入口，同时允许运行时覆盖默认参数。

**Dockerfile 示例**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
ENTRYPOINT ["python"]
CMD ["app.py"]
```

**构建与运行**

```bash
docker build -t mypy:1.0 .
docker run --rm mypy:1.0
```

**输出示例**

```text
default app running
```

覆盖默认参数：

```bash
docker run --rm mypy:1.0 other.py
```

**输出解释**

| 指令                             | 含义                     |
| ------------------------------ | ---------------------- |
| `ENTRYPOINT ["python"]`        | 固定入口为 `python`         |
| `CMD ["app.py"]`               | 默认参数为 `app.py`         |
| `docker run mypy:1.0 other.py` | 实际执行 `python other.py` |

**验证是否达到目的**

如果运行 `other.py` 能执行，说明 ENTRYPOINT + CMD 组合生效。

---

## 6.3 场景：多阶段构建减小镜像

**目的**  
构建阶段包含编译器和依赖，最终镜像只包含运行所需文件。

**Dockerfile 示例**

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp .

FROM alpine:3.19
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

**构建**

```bash
docker build -t myapp:multi .
```

**验证是否达到目的**

```bash
docker images myapp:multi
```

对比单阶段构建，最终镜像体积应显著减小。

```bash
docker run --rm myapp:multi
```

能正常运行，说明多阶段构建成功。

---

# 7. Docker Compose

## 7.1 命令速查

| 命令                                                | 作用         | 示例                                                |
| ------------------------------------------------- | ---------- | ------------------------------------------------- |
| `docker compose up`                               | 前台启动       | `docker compose up`                               |
| `docker compose up -d`                            | 后台启动       | `docker compose up -d`                            |
| `docker compose up --build`                       | 重新构建并启动    | `docker compose up -d --build`                    |
| `docker compose down`                             | 停止并删除容器、网络 | `docker compose down`                             |
| `docker compose down -v`                          | 同时删除数据卷    | `docker compose down -v`                          |
| `docker compose ps`                               | 查看服务状态     | `docker compose ps`                               |
| `docker compose logs -f web`                      | 查看服务日志     | `docker compose logs -f web`                      |
| `docker compose exec web /bin/bash`               | 进入服务容器     | `docker compose exec web /bin/bash`               |
| `docker compose run web python manage.py migrate` | 执行一次性命令    | `docker compose run web python manage.py migrate` |
| `docker compose config`                           | 验证配置       | `docker compose config`                           |

---

## 7.2 场景：一键启动 Web + 数据库

**目的**  
根据 `compose.yaml` 一键启动多容器应用。

**前置条件**  
当前目录有 `compose.yaml`。

**compose.yaml 示例**

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=1
    depends_on:
      - db
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

volumes:
  db-data:
```

**操作方法**

```bash
docker compose up -d
```

**输出示例**

```text
[+] Running 4/4
 ✔ Network myapp_default  Created
 ✔ Container myapp-db-1   Started
 ✔ Container myapp-web-1  Started
```

**输出解释**

| 输出                              | 含义      |
| ------------------------------- | ------- |
| `Network myapp_default Created` | 创建默认网络  |
| `Container ... Started`         | 服务容器已启动 |
| `-d`                            | 后台运行    |

**验证是否达到目的**

```bash
docker compose ps
```

输出示例：

```text
NAME            IMAGE          STATUS          PORTS
myapp-db-1      postgres:16    Up 9 seconds    5432/tcp
myapp-web-1     myapp:1.0      Up 9 seconds    0.0.0.0:8000->8000/tcp
```

再验证应用：

```bash
curl -I http://localhost:8000
```

返回 `200 OK`，说明 Compose 启动成功。

**清理**

```bash
docker compose down
```

若需删除数据卷：

```bash
docker compose down -v
```

---

# 8. Swarm 集群管理

## 8.1 命令速查

| 命令                      | 作用     | 示例                                                               |
| ----------------------- | ------ | ---------------------------------------------------------------- |
| `docker swarm init`     | 初始化集群  | `docker swarm init --advertise-addr 192.168.1.100`               |
| `docker swarm join`     | 加入集群   | `docker swarm join --token <TOKEN> 192.168.1.100:2377`           |
| `docker service create` | 创建服务   | `docker service create --name web --replicas 3 -p 8080:80 nginx` |
| `docker service ls`     | 列出服务   | `docker service ls`                                              |
| `docker service ps`     | 查看任务分布 | `docker service ps web`                                          |
| `docker service scale`  | 扩缩容    | `docker service scale web=5`                                     |
| `docker service update` | 更新服务   | `docker service update --image nginx:1.26 web`                   |
| `docker service rm`     | 删除服务   | `docker service rm web`                                          |

---

## 8.2 场景：初始化集群并部署服务

**目的**  
在多台服务器上搭建 Docker Swarm 集群，并部署可扩展服务。

**操作方法**

管理节点：

```bash
docker swarm init --advertise-addr 192.168.1.100
```

输出示例：

```text
Swarm initialized: current node (xxx) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-xxx 192.168.1.100:2377
```

工作节点：

```bash
docker swarm join --token SWMTKN-1-xxx 192.168.1.100:2377
```

创建服务：

```bash
docker service create --name web --replicas 3 --publish 8080:80 nginx:1.25
```

**验证是否达到目的**

```bash
docker service ls
```

输出示例：

```text
ID             NAME   MODE         REPLICAS   IMAGE        PORTS
xxx            web    replicated   3/3        nginx:1.25   *:8080->80/tcp
```

`REPLICAS 3/3` 表示三个副本全部运行。

扩缩容：

```bash
docker service scale web=5
```

滚动更新：

```bash
docker service update --image nginx:1.26 web
```

再次查看：

```bash
docker service ps web
```

所有任务更新为新镜像，说明滚动更新成功。

---

# 9. 资源限制速查

| 参数              | 说明         | 示例                          | 场景               |
| --------------- | ---------- | --------------------------- | ---------------- |
| `--cpus`        | 限制 CPU 核心数 | `--cpus=2`                  | 防止 CPU 密集型容器抢占资源 |
| `--cpuset-cpus` | 绑定指定核心     | `--cpuset-cpus="0,1"`       | 避免跨 NUMA 节点性能损耗  |
| `-m, --memory`  | 限制最大内存     | `-m 512m`                   | 防止内存泄漏容器拖垮宿主机    |
| `--memory-swap` | 内存+交换区总量   | `--memory-swap 1g`          | 控制交换使用上限         |
| `--cpu-shares`  | CPU 权重     | `--cpu-shares 512`          | 多容器竞争时分配优先级      |
| `--ulimit`      | 文件描述符等限制   | `--ulimit nofile=1024:2048` | 高并发连接场景          |

---

# 10. 常用组合与清理

## 10.1 常用组合

**运行 Nginx 并挂载网页目录**

```bash
docker run -d --name my-nginx -p 80:80 -v /host/www:/usr/share/nginx/html nginx:1.25
```

**运行 MySQL 并持久化数据**

```bash
docker run -d --name mysql \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql-data:/var/lib/mysql \
  --restart unless-stopped \
  mysql:8.0
```

**构建并推送镜像**

```bash
docker build -t registry.example.com/myapp:1.0 .
docker push registry.example.com/myapp:1.0
```

**批量删除所有容器**

```bash
docker rm -f $(docker ps -aq)
```

---

## 10.2 安全清理

**目的**  
磁盘告急时，清理停止的容器、未使用镜像、未使用网络、未使用卷和构建缓存。

**操作方法**

```bash
docker system prune -af --volumes
```

**输出示例**

```text
Deleted Containers:
c3f2a1b9d8e7...
Deleted Images:
untagged: nginx:1.24
deleted: sha256:...
Deleted Volumes:
my-old-data
Deleted Networks:
old-net
Total reclaimed space: 2.3GB
```

**输出解释**

| 输出                      | 含义      |
| ----------------------- | ------- |
| `Deleted Containers`    | 删除停止的容器 |
| `Deleted Images`        | 删除未使用镜像 |
| `Deleted Volumes`       | 删除未使用卷  |
| `Total reclaimed space` | 总共释放空间  |

**验证是否达到目的**

```bash
docker system df
```

清理前 `RECLAIMABLE` 可能为 `1.8GB`，清理后应接近 `0B` 或明显下降。

> ⚠️ **高危操作警告**  
> `--volumes` 可能删除数据库数据卷。生产环境执行前必须先执行 `docker system df -v` 确认要删除的内容。

---

# 附录 A 全局选项

| 选项                | 说明        | 示例                               |
| ----------------- | --------- | -------------------------------- |
| `-D, --debug`     | 启用调试模式    | `docker -D ps`                   |
| `-H, --host`      | 指定守护进程地址  | `docker -H tcp://remote:2375 ps` |
| `-l, --log-level` | 设置日志级别    | `docker -l warn ps`              |
| `--config`        | 客户端配置文件位置 | `docker --config ~/.docker ps`   |
| `-c, --context`   | 指定上下文     | `docker -c mycontext ps`         |

---

# 附录 B 命令速查总表

| 分类      | 命令                       | 常用示例                                                  |
| ------- | ------------------------ | ----------------------------------------------------- |
| 环境      | `docker version`         | `docker version`                                      |
| 环境      | `docker info`            | `docker info`                                         |
| 系统      | `docker system df`       | `docker system df -v`                                 |
| 系统      | `docker system prune`    | `docker system prune -af --volumes`                   |
| 镜像      | `docker pull`            | `docker pull nginx:1.25`                              |
| 镜像      | `docker images`          | `docker images -a`                                    |
| 镜像      | `docker build`           | `docker build -t myapp:1.0 .`                         |
| 镜像      | `docker push`            | `docker push registry.example.com/myapp:1.0`          |
| 容器      | `docker run`             | `docker run -d --name web -p 8080:80 nginx:1.25`      |
| 容器      | `docker ps`              | `docker ps -a`                                        |
| 容器      | `docker exec`            | `docker exec -it web /bin/bash`                       |
| 容器      | `docker logs`            | `docker logs -f --tail 100 web`                       |
| 容器      | `docker stats`           | `docker stats --no-stream web`                        |
| 网络      | `docker network create`  | `docker network create my-net`                        |
| 网络      | `docker network inspect` | `docker network inspect my-net`                       |
| 卷       | `docker volume create`   | `docker volume create my-data`                        |
| 卷       | `docker volume inspect`  | `docker volume inspect my-data`                       |
| Compose | `docker compose up`      | `docker compose up -d`                                |
| Compose | `docker compose down`    | `docker compose down -v`                              |
| Swarm   | `docker service create`  | `docker service create --name web --replicas 3 nginx` |

---

# 附录 C 场景卡片模板

后续新增命令或参数时，统一按以下模板编写：

```markdown
## 场景：<场景名称>

**目的**  
<说明为什么要执行该操作，要解决什么问题。>

**前置条件**  
<说明需要哪些环境、权限、文件、端口、镜像等。>

**操作方法**

```bash
<命令>
```

**输出示例**

```text
<输出内容>
```

**输出解释**

| 输出/字段 | 含义  |
| ----- | --- |
| ...   | ... |

**验证是否达到目的**

```bash
<验证命令>
```

输出示例：

```text
<验证输出>
```

**清理/注意**

> ⚠️ <注意事项、清理命令、高危提醒>

```

---

**文档结束**
