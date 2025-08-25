# Docker_Summary

## Docker基础概念

- **Docker Registry**：存储镜像的仓库
  - 公有仓库：如 Docker Hub（默认官方仓库）

  - 私有仓库：如 Harbor、GitLab Registry（企业内部使用）

- **镜像（Image）**：容器的只读模板，含代码、依赖等，基于分层文件系统构建

  - **镜像命名格式**：`<registry>/<namespace>/<image>:<tag>`

    - `registry`：仓库地址

    - `namespace`：命名空间（区分项目 / 用户）

    - `image`：镜像名

    - `tag`：版本标签（默认latest）

- **容器（Container）**：镜像的可运行实例，独立运行环境，可创建、启停、删除

- **数据卷（Volume）**：独立存储区域，实现数据持久化，与容器生命周期解耦

## Docker常用操作

### 镜像操作

#### Dockerfile 创建镜像：

##### 常用指令

1. 基础指令：

   - **`FROM <image>[:<tag>]`：指定基础镜像，所有 Dockerfile 都必须以此开始**

     - **如果遇到基础镜像源超时问题，需要更换国内镜像源**，如：

       ```dockerfile
       #docker init 自动生成的dockerfile
       FROM alpine:latest AS final
       
       # alpine 为 Linux的轻量化版本，默认源下载慢会SSL连接超时；采用阿里云代替
       RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
       ```

   - `LABEL maintainer="your-name <your-email@example.com>"`： 指明镜像维护者信息

2. 构建时指令：

   - `ARG <name>[=<default value>]`： 定义**构建时变量**

   - `ENV <key>=<value>` 或 `ENV <key> <value>`：**设置环境变量，在构建和运行时都有效**

   - **`WORKDIR /path/to/workdir`：设置工作目录**，影响后续的 RUN、CMD、ENTRYPOINT、COPY、ADD

     - 示例：

       ```dockerfile
       WORKDIR /usr/src/app
       COPY . .  # 复制到当前工作目录
       ```

   - **`RUN <command>` 或 `RUN ["executable", "param1", "param2"]`：在构建镜像时执行命令，每个 RUN 创建一个新的镜像层**

     - 示例：

       ```dockerfile
       RUN apt-get update && apt-get install -y python3
       RUN ["npm", "install"]
       # 多命令合并减少镜像层
       RUN apt-get update \
           && apt-get install -y curl vim \
           && rm -rf /var/lib/apt/lists/*
       ```

   - **`COPY [--chown=<user>:<group>] <src>... <dest>`： 将文件/目录从构建上下文复制到镜像中**

     - **注意: 推荐使用 COPY，除非需要 ADD 的特殊功能**

     - 示例：

       ```dockerfile
       COPY app.py /usr/src/app/
       COPY --chown=1000:1000 config/ /etc/myapp/
       COPY ["hello world.txt", "/tmp/"]  # 文件名有空格时
       ```

   - `ADD [--chown=<user>:<group>] <src>... <dest>`：类似 COPY，但**`ADD`支持 URL 下载和自动解压缩**

     - 示例：

       ```
       WORKDIR /usr/src/app
       COPY . .  # 复制到当前工作目录
       ```

3. 运行时指令：

   - **`EXPOSE <port> [<port>/<protocol>...]`：声明容器监听的端口**（仅作文档说明）
     - **注意: 实际映射端口需要 `docker run -p`**
   - **`VOLUME ["/data"]` 或 `VOLUME /data`： 创建挂载点，指定可被外部挂载的目录**
   - `USER <user>[:<group>]` 或 `USER <UID>[:<GID>]`：指定运行容器时的用户

4. 启动指令：**`CMD`  与  `ENTRYPOINT` 的区别！！！**

   - `CMD command param1 param2` 或者 `CMD ["executable","param1","param2"]`：指定容器启动时的默认命令
     - **docker run 覆盖  `CMD` 执行的命令**
     - **注意**: 只有最后一个 CMD 指令生效
   - `ENTRYPOINT  command param1 param2`，语法同 `CMD`：指定容器启动时执行的命令（**不会被 docker run 参数覆盖**）
     - **docker run 追加在 `ENTRYPOINT` 执行的命令后**

##### 多阶段构建

```dockerfile
# 构建阶段
FROM golang:1.21 AS build
WORKDIR /src
COPY . .
RUN go build -o app .

# 运行阶段
FROM alpine:latest AS final
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories # linux记得更换国内源
RUN apk --no-cache add ca-certificates
COPY --from=builder /src/app /usr/local/bin/
CMD ["app"]
```



##### docker init 自动得到 Dockerfile 

注意： `alpine:latest` 切换为国内源

![image-20250824170847468](./assets/image-20250824170847468.png)



#### 管理镜像：

1.  **`docker images`：列出本地已下载的 Docker 镜像，默认隐藏中间镜像**
   - `-a`：显示所有镜像（包括中间镜像）  
   - `-q`：只显示镜像 ID
   - `--no-trunc`：显示完整的镜像 ID（不截断）  
2.  **`docker rmi <镜像>`：删除指定镜像；无法直接删除正在运行的容器所依赖的镜像，需先停止并删除容器**
   - `-f`：强制删除镜像（即使有容器依赖也删除，需谨慎使用）。  
3.  `docker inspect <镜像名/容器名/ID>`：查看镜像或容器的详细元数据（JSON 格式）
4.  **镜像的导入和导出（保存和加载）：**
   - `docker save -o <文件名.tar> <镜像>`：保存镜像为 tar 文件（保留元数据）
      - **区别于：容器的导出 `docker export`，不保留元数据，只保存容器的文件系统**

   - `docker load -i <文件名.tar>`：从 tar 文件加载镜像




### 容器操作

#### **创建并启动容器：**

1. 创建（仅创建不启动）：`docker create <镜像>`
2. **创建并启动：`docker run <镜像>`**；如果本地镜像没有，则自动从Docker Hub拉取
   - `-it`：交互模式，进入容器终端
   - `-d`：后台运行（守护模式）
     - `--restart <策略>`：设置启动策略
       - `no`（默认值）：不会在 Docker 启动时自动启动容器。你需要手动 `docker start`。

       - `always`：容器会在 Docker 启动时自动启动（哪怕你没有手动 start）。

       - `unless-stopped`：和 `always` 类似，但如果你上次是手动停止的，它就不会自动启动

       - `on-failure`：仅在容器异常退出（非 0 状态码）时自动重启

   - `-p <主机端口>:<容器端口>`：端口映射
     - `-P`：随机映射所有 EXPOSE 的端口
   - `-v <主机路径>:<容器路径>`：绑定挂载
     - `-v <卷名>:<容器路径>`：挂载数据卷
   - `--name <名称>`：指定容器名称
   - `--net= <网络名>`：指定所属网络
   - `--rm`：容器停止后自动删除
3. 启动容器：  `docker start <容器ID/名称>`
4. 停止容器（优雅退出）：  `docker stop <容器ID/名称>`
5. 强制终止（立即终止）：`docker kill <容器ID/名称>`
6. 重启容器：  `docker restart <容器ID/名称>`
7. 暂停 / 恢复：`docker pause/docker unpause <容器ID/名称>`
8. **删除容器：  `docker rm <容器ID/名称>` 仅删除停止的容器**
   - **强制删除运行中的容器：`docker rm -f <容器ID/名称>`** 
   - **删除挂载的数据卷卷 ：`docker rm -v <容器ID/名称>`**



#### **管理容器：**

1. **列出容器：**

   - **`docker ps`：仅列出正在运行的容器**
     - `-a`：显示所有容器（包括已停止的）
     - `-l`：显示最近创建的容器。  
     - `-q`：只显示容器 ID。  
     - `--no-trunc`：显示完整的容器信息  

2. **进入容器：**

   - `docker exec -it <容器ID/名称> <命令>`：在运行容器内执行命令（一般用于，进入终端bash）。

   - `docker attach <容器ID/名称>`：连接容器输入输出（退出可能导致容器停止，建议用exec）

3. 查看容器日志等信息：

   - `docker logs -ft <容器ID/名称>`：查看日志（`-f`实时跟踪，）

     - `-f`：实时跟踪日志
     - `-t`：显时间戳

   - `docker inspect <镜像名/容器ID/名称>`：查看镜像/容器的详细信息

   - `docker stats <容器ID/名称>`：实时查看资源占用（CPU、内存等）

     `docker top <容器ID/名称>`：查看容器内进程

   - `docker rename <旧名称> <新名称>`：重命名容器

4. **容器的导出和导入：**

   -  `docker export <容器ID/名称> -o <文件名.tar>`：将容器当前状态（文件系统）导出为 tar 文件
     - **区别于镜像的保存，`docker save` 包含镜像的所有内容**
   - `docker import <文件名.tar> <新镜像名:标签>`：将容器快照导入为“新”镜像



### 仓库操作

- `docker login <仓库地址>`：登录仓库（默认 Docker Hub）

- `docker logout <仓库地址>`：退出登录

- **`docker push <仓库地址/命名空间/镜像名:标签>`：推送镜像**

- **`docker pull <仓库地址/命名空间/镜像名:标签>`：拉取镜像**

### 数据卷操作

- `docker volume create <卷名>`：创建

- `docker volume ls`：列出所有

- `docker volume inspect <卷名>`：查看详情

- `docker volume rm <卷名>`：删除

- `docker volume prune`：删除未使用的

> **绑定挂载：主机文件 / 目录直接挂载到容器，数据双向同步**
>
> 即 ：`docker run -v /host/path:/container/path nginx` 





## Docker Compose

**Dockerfile 与 Docker Compose 区别：**

- 一个 **Dockerfile** 是一个文本文件，**包含用户可以在命令行上调用的所有命令/指令，以组装一个镜像**；

  - 使用 **docker build** 命令，我们可以从 Dockerfile 构建一个镜像

- **Docker Compose** 是一个用于**定义和运行多容器 Docker 应用程序**的工具；

  - 使用 Compose，您可以**使用 YAML 文件来配置应用程序的服务**。然后，**通过一个命令，您可以从配置中创建并启动所有服务**

  

### docker-compose.yaml 编写

- **顶级关键字：**

  - `version`: 指定 Compose 文件格式版本

  - **`services`: 定义服务配置**
    - 包含一个或多个服务定义
    - 每个服务为独立的容器配置单元
    - 服务名称作为子键（如 web: , db:）

  - **`volumes`: 定义命名卷**
    - 包含一个或多个服务定义
    - 每个服务为独立的容器配置单元
    - 服务名称作为子键（如 web:、db:）

  - `networks`: 定义网络配置
    - 定义服务间通信的网络
    - 可指定网络驱动（如 bridge）
    - 支持 IP 地址管理配置（子网、网关）
    - 可引用外部已存在的网络

- **服务级关键字：**

  - 基础配置

    - **`image`: 指定容器使用的镜像（如 nginx:alpine） (与 build 关键字互斥)，**
      - 从本地镜像查找；本地没有，则从Compose拉取

    - **`build`: 从 Dockerfile 构建镜像（与 image 关键字互斥），**
      - **需指定 context（构建上下文路径）**
      - 可指定 dockerfile（自定义 Dockerfile 名称）
      - 支持 build args（构建参数）

    - `container_name`: 容器名称
      - 默认为：`项目名_服务名_序号`

    - `hostname`: 容器主机名

    - `restart`: 重启策略（`always|on-failure|unless-stopped|no`）

  - 网络和端口

    - **`ports`: 端口映射**

    - `expose`: 暴露端口

    - `networks`: 网络配置

    - `external_links`: 外部链接

  - 存储

    - **`volumes`: 卷挂载**

    - `tmpfs`: 临时文件系统

  - 环境配置

    - `environment`: 环境变量

    - `env_file`: 环境变量文件

    - `working_dir`: 工作目录
    - `user`: 运行用户

  - 命令和入口点

    - `command`: 覆盖默认命令

    - `entrypoint`: 入口点

    - `stdin_open`: 保持STDIN开启
    - `tty`: 分配伪TTY

  - 依赖关系

    - `depends_on`: 服务依赖

    - `links`: 服务链接（已弃用）

  - 资源限制

    - `mem_limit`: 内存限制
    - `cpus`: CPU限制

    - `deploy`: 部署配置（Swarm模式）

  - 健康检查	
    - `healthcheck`: 健康检查配置

  - 日志
    - `logging`: 日志配置

```yaml
# Docker Compose 教学版配置模板
# 包含常用关键字的完整示例，适合学习使用

version: '3.8'  # 指定Compose文件格式版本

services:
  # 服务1：Web应用服务 - 演示完整配置选项
  webapp:
    # === 基础镜像配置 ===
    image: nginx:alpine  # 使用现有镜像
    # build:  # 或者构建自定义镜像（二选一）
    #   context: ./webapp
    #   dockerfile: Dockerfile
    #   args:
    #     - NODE_ENV=production
    
    container_name: my-webapp  # 自定义容器名
    hostname: webapp-server    # 容器内主机名
    restart: unless-stopped    # 重启策略: no|always|on-failure|unless-stopped
    
    # === 端口和网络 ===
    ports:
      - "80:80"     # 主机端口:容器端口
      - "443:443"   # HTTPS端口
    expose:         # 仅容器间可见的端口
      - "9000"
    networks:       # 加入的网络
      - frontend
      - backend
    
    # === 存储挂载 ===
    volumes:
      - ./html:/usr/share/nginx/html:ro  # 绑定挂载(只读)
      - app-data:/var/lib/app           # 命名卷
      - /tmp                            # 匿名卷
    
    # === 环境配置 ===
    environment:    # 环境变量
      DB_HOST: database
      DB_PORT: 5432
      LOG_LEVEL: info
    env_file:       # 环境变量文件
      - .env
    working_dir: /app        # 工作目录
    user: "1000:1000"        # 运行用户
    
    # === 启动命令 ===
    command: ["nginx", "-g", "daemon off;"]  # 覆盖默认命令
    # entrypoint: ["/custom-entrypoint.sh"]  # 自定义入口点
    
    # === 依赖关系 ===
    depends_on:     # 启动顺序依赖
      - database
      - cache
    
    # === 资源限制 ===
    deploy:         # Swarm模式配置
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 256M
    
    # === 健康检查 ===
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    
    # === 日志配置 ===
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 服务2：数据库服务 - 演示常用数据库配置
  database:
    image: postgres:14-alpine
    container_name: my-postgres
    restart: unless-stopped
    
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: myapp
    
    volumes:
      - postgres_data:/var/lib/postgresql/data  # 数据持久化
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # 初始化脚本
    
    ports:
      - "5432:5432"  # 开发时可映射端口，生产环境建议移除
    
    networks:
      - backend
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # 服务3：缓存服务 - 演示简单服务配置
  cache:
    image: redis:7-alpine
    container_name: my-redis
    restart: on-failure
    
    volumes:
      - redis_data:/data
    
    networks:
      - backend
    
    command: redis-server --appendonly yes --requirepass myredispass

  # 服务4：后台任务服务 - 演示构建配置
  worker:
    build:
      context: ./worker
      dockerfile: Dockerfile
      args:
        - PYTHON_VERSION=3.9
    
    container_name: my-worker
    restart: unless-stopped
    
    environment:
      - CELERY_BROKER=redis://cache:6379
      - DATABASE_URL=postgresql://myuser:mypassword@database:5432/myapp
    
    depends_on:
      - database
      - cache
    
    networks:
      - backend
    
    volumes:
      - ./logs:/app/logs  # 日志目录
    
    # 不映射端口，因为是后台服务

# === 卷定义 ===
volumes:
  postgres_data:    # 数据库数据卷
    driver: local
  redis_data:       # Redis数据卷
    driver: local
  app_data:         # 应用数据卷
    driver: local
    # driver_opts:    # 高级选项（可选）
    #   type: none
    #   o: bind
    #   device: /host/path

# === 网络定义 ===
networks:
  frontend:         # 前端网络
    driver: bridge
  backend:          # 后端网络
    driver: bridge
    internal: true  # 内部网络，无法访问外部
  # external_net:   # 外部网络引用
  #   external: true

# === 常用配置说明 ===
# 1. restart策略:
#    - no: 不自动重启(默认)
#    - always: 总是重启
#    - on-failure: 非正常退出时重启
#    - unless-stopped: 除非手动停止，否则总是重启

# 2. 卷挂载语法:
#    - 主机路径:容器路径[:选项]
#    - 卷名:容器路径[:选项]
#    - 容器路径 (匿名卷)
#    选项: ro(只读), rw(读写), z(SELinux标签)

# 3. 端口映射:
#    - "主机端口:容器端口"
#    - "主机IP:主机端口:容器端口"
#    - 范围映射: "8000-8010:8000-8010"

# 4. depends_on注意事项:
#    - 只保证启动顺序，不等待服务就绪
#    - 建议配合healthcheck使用
#    - 或在应用中实现重试机制

# 5. 网络最佳实践:
#    - 前端服务放在frontend网络
#    - 后端服务放在backend网络
#    - 数据库等敏感服务使用内部网络

# 6. 安全建议:
#    - 生产环境不要映射数据库端口到主机
#    - 使用.env文件管理敏感信息
#    - 为服务设置非root用户
#    - 使用只读挂载当不需要写入时
```

​	

### Docker Compose 常用命令

1. **构建和拉取**

   ```bash
   # 构建服务
   docker-compose build
   # 构建时不使用缓存
   docker-compose build --no-cache
   # 拉取镜像
   docker-compose pull
   
   # 强制重新构建并启动
   docker-compose up --build
   ```

2. 基本命令

   ```bash
   # 启动所有服务
   docker-compose up
   # 后台启动
   docker-compose up -d
   # 启动指定服务
   docker-compose up web db
   
   # 停止所有服务
   docker-compose down
   # 停止并删除卷
   docker-compose down -v
   # 停止并删除镜像
   docker-compose down --rmi all
   ```

3. 服务管理

   ```bash
   # 查看服务状态
   docker-compose ps
   # 查看服务日志
   docker-compose logs
   # 查看指定服务日志
   docker-compose logs web
   # 实时查看日志
   docker-compose logs -f
   
   # 重启服务
   docker-compose restart
   # 重启指定服务
   docker-compose restart web
   
   # 停止服务
   docker-compose stop
   # 启动服务
   docker-compose start
   ```

4. 扩展和执行

   ```bash
   # 扩展服务实例
   docker-compose up --scale web=3
   
   # 在服务中执行命令
   docker-compose exec web bash
   # 运行一次性命令
   docker-compose run web python manage.py migrate
   # 运行命令但不启动依赖服务
   docker-compose run --no-deps web npm test
   ```

5. 配置和验证

   ```bash
   # 验证 compose 文件
   docker-compose config
   
   # 查看配置（解析后的完整配置）
   docker-compose config --services
   
   # 列出所有服务
   docker-compose ps --services
   # 查看镜像
   docker-compose images
   # 查看卷
   docker-compose ps -v
   ```

6. 清理命令

   ```bash
   # 删除停止的容器
   docker-compose rm
   # 强制删除
   docker-compose rm -f
   
   # 删除所有未使用的网络
   docker network prune
   
   # 删除所有未使用的卷
   docker volume prune
   
   # 完整清理（容器、网络、卷、镜像）
   docker system prune -a --volumes
   ```
