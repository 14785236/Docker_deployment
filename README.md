## 目录

1. [安装 Docker 和 Docker Compose](#安装-docker-和-docker-compose)
   - [更新软件包列表并安装依赖](#更新软件包列表并安装依赖)
   - [添加阿里云的 Docker GPG 密钥](#添加阿里云的-docker-gpg-密钥)
   - [添加阿里云的 Docker 仓库源](#添加阿里云的-docker-仓库源)
   - [更新 APT 并安装 Docker 和 Docker Compose](#更新-apt-并安装-docker-和-docker-compose)
   - [验证 Docker 是否安装成功](#验证-docker-是否安装成功)
   - [可选：添加当前用户到 Docker 组](#可选添加当前用户到-docker-组)
2. [创建项目目录结构](#创建项目目录结构)
3. [拉取代码](#拉取代码)
4. [创建 Dockerfile](#创建-dockerfile)
   - [前端 Dockerfile](#前端-dockerfile)
   - [后端 Dockerfile](#后端-dockerfile)
5. [创建 Nginx 配置](#创建-nginx-配置)
6. [创建 `docker-compose.yml`](#创建-docker-composeyml)
7. [启动 Docker Compose](#启动-docker-compose)
8. [自动更新 Docker 容器](#自动更新-docker-容器)
   - [设置裸仓库和工作副本](#设置裸仓库和工作副本)
   - [配置 Git 钩子](#配置-git-钩子)
   - [在本地推送代码](#在本地推送代码)
   - [配置 `docker-compose.yml`](#配置-docker-composeyml-1)
9. [将 Docker 应用部署到其他服务器](#将-docker-应用部署到其他服务器)

---

## 安装 Docker 和 Docker Compose

### 更新软件包列表并安装依赖

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release
```

### 添加阿里云的 Docker GPG 密钥

```bash
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```

### 添加阿里云的 Docker 仓库源

```bash
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```

### 更新 APT 并安装 Docker 和 Docker Compose

```bash
sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# 下载 Docker Compose 2.23.3 版本的二进制文件
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-$(uname -m)" -o /usr/local/bin/docker-compose

# 赋予执行权限
sudo chmod +x /usr/local/bin/docker-compose

# 检查安装
docker-compose --version
```
#### 配置阿里云加速镜像
阿里云镜像获取地址：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
登陆后，左侧菜单选中镜像加速器就可以看到你的专属地址了
```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://<你的加速id>.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
#验证配置是否成功
cat /etc/docker/daemon.json
#输出{
#  "registry-mirrors": ["https://<你的加速id>.mirror.aliyuncs.com"]
#}

```
### 验证 Docker 是否安装成功

```bash
sudo docker run hello-world
```

### 可选：添加当前用户到 Docker 组

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 创建项目目录结构

```bash
mkdir -p ~/project-root/{frontend,backend,nginx}
cd ~/project-root
```

---

## 拉取代码

```bash
# 拉取前端代码
cd ~/project-root/frontend
git clone <frontend-git-repo> .

# 拉取后端代码
cd ../backend
git clone <backend-git-repo> .
```

> 替换 `<frontend-git-repo>` 和 `<backend-git-repo>` 为你的前端和后端代码仓库的 URL。

---

## 创建 Dockerfile

### 前端 Dockerfile

```bash
cat << 'EOF' > ~/project-root/frontend/Dockerfile
# frontend/Dockerfile
# 使用 node:18-alpine 作为基础镜像
FROM node:18-alpine AS build

# 设置工作目录
WORKDIR /app

# 复制 package.json 和 package-lock.json
COPY archives-manage/package*.json ./

# 安装依赖
RUN npm install

# 复制所有文件到工作目录
COPY archives-manage/ .

# 构建项目
RUN npm run build

# 使用 Nginx 作为静态文件服务器
FROM nginx:alpine

# 将构建好的静态文件复制到 Nginx 默认的静态文件目录
COPY --from=build /app/dist /usr/share/nginx/html

# 暴露端口 80
EXPOSE 80

# 默认启动 Nginx
CMD ["nginx", "-g", "daemon off;"]
EOF
```

### 后端 Dockerfile

```bash
cat << 'EOF' > ~/project-root/backend/Dockerfile
# backend/Dockerfile
# 使用 Maven 作为构建环境
FROM maven:3.6.2-jdk-8 AS build
WORKDIR /app

# 复制 pom.xml 和 src 目录
COPY archive/pom.xml ./pom.xml
COPY archive/src ./src

# 构建应用
RUN mvn clean package -DskipTests

# 使用 OpenJDK 运行环境
FROM openjdk:8-alpine
WORKDIR /app

# 复制构建的 jar 文件
COPY --from=build /app/target/*.jar app.jar

# 设置启动命令
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
EXPOSE 10001
EOF
```

---

## 创建 Nginx 配置

```bash
cat << 'EOF' > ~/project-root/nginx/default.conf
# nginx/default.conf
server {
    listen 80;

    # 配置第一个前端服务
    location /frontend1/ {
        proxy_pass http://127.0.0.1/frontend:20001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 配置第二个前端服务
    location /frontend2/ {
        proxy_pass http://127.0.0.1/frontend:20007/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 配置后端 API 的路由
    location /api/ {
        proxy_pass http://127.0.0.1/backend:10001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 配置上传文件的路由
    location /upload/ {
        alias /usr/share/nginx/html/upload/;
        autoindex on;
    }
}
EOF
```

---

## 创建 `docker-compose.yml`

```bash
cat << 'EOF' > ~/project-root/docker-compose.yml
version: '3.9'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "20001:80"
      - "20007:80"
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "10001:10001"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://127.0.0.1:3306/archive?characterEncoding=utf8&useSSL=false&zeroDateTimeBehavior=convertToNull
      - SPRING_DATASOURCE_USERNAME=test
      - SPRING_DATASOURCE_PASSWORD=test@123
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
      - SPRING_REDIS_PASSWORD=123456
      - SPRING_REDIS_DATABASE=13
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: example
      MYSQL_DATABASE: archive
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:5
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend

volumes:
  mysql-data:
  redis-data:
EOF
```

---

## 启动 Docker Compose

```bash
cd ~/project-root
docker-compose up -d
```

---

## 测试环境自动更新 Docker 容器

### 设置裸仓库和工作副本

```bash
# 前端裸仓库
mkdir -p /home/user/frontend-repo.git
cd /home/user/frontend-repo.git
git init --bare

# 后端裸仓库
mkdir -p

 /home/user/backend-repo.git
cd /home/user/backend-repo.git
git init --bare

# 前端工作副本
mkdir -p /home/user/frontend-workdir
cd /home/user/frontend-workdir
git clone /home/user/frontend-repo.git .

# 后端工作副本
mkdir -p /home/user/backend-workdir
cd /home/user/backend-workdir
git clone /home/user/backend-repo.git .
```

### 配置 Git 钩子

```bash
# 配置前端的 post-receive 钩子
cat << 'EOF' > /home/user/frontend-repo.git/hooks/post-receive
#!/bin/bash
WORKDIR="/home/user/frontend-workdir"
cd $WORKDIR
unset GIT_DIR
git pull origin master
cd $WORKDIR
docker-compose -f $WORKDIR/docker-compose.yml up -d --build frontend
EOF

chmod +x /home/user/frontend-repo.git/hooks/post-receive

# 配置后端的 post-receive 钩子
cat << 'EOF' > /home/user/backend-repo.git/hooks/post-receive
#!/bin/bash
WORKDIR="/home/user/backend-workdir"
cd $WORKDIR
unset GIT_DIR
git pull origin master
cd $WORKDIR
docker-compose -f $WORKDIR/docker-compose.yml up -d --build backend
EOF

chmod +x /home/user/backend-repo.git/hooks/post-receive
```

### 在本地推送代码

```bash
# 推送前端代码
cd ~/project-root/frontend
git remote add production ssh://<username>@<server-ip>/home/user/frontend-repo.git
git push production master

# 推送后端代码
cd ../backend
git remote add production ssh://<username>@<server-ip>/home/user/backend-repo.git
git push production master
```

> 替换 `<username>` 和 `<server-ip>` 为你的服务器的用户名和 IP 地址。

### 配置 `docker-compose.yml`

```bash
cat << 'EOF' > /home/user/frontend-workdir/docker-compose.yml
version: '3.9'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "20001:80"
      - "20007:80"
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "10001:10001"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:mysql://127.0.0.1:3306/archive?characterEncoding=utf8&useSSL=false&zeroDateTimeBehavior=convertToNull
      - SPRING_DATASOURCE_USERNAME=test
      - SPRING_DATASOURCE_PASSWORD=test@123
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
      - SPRING_REDIS_PASSWORD=123456
      - SPRING_REDIS_DATABASE=13
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: example
      MYSQL_DATABASE: archive
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:5
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend

volumes:
  mysql-data:
  redis-data:
EOF
```

---

## 上线生成环境将 Docker 应用部署到其他服务器

1. 在目标服务器上安装 Docker 和 Docker Compose。
2. 拉取代码仓库并创建相应的目录结构。
3. 使用 `docker-compose up -d` 命令启动应用。
```

### 说明

- 文档中 `<frontend-git-repo>` 和 `<backend-git-repo>` 需要替换为你的前端和后端代码仓库的实际 URL。
- 文档中的 `<username>` 和 `<server-ip>` 需要替换为你的服务器的用户名和 IP 地址。

### 如何执行

你可以按照 `README.md` 中的步骤执行所有命令。确保你的服务器上具备必要的权限和工具，如 `git`、`docker`、`docker-compose`。
我们继续完善文档，涵盖如何将 Docker 应用部署到其他服务器以及如何进行测试。

### 部署和测试部分更新后的内容：

---

## 将 Docker 应用部署到其他服务器

要将 Docker 应用部署到其他服务器，请按以下步骤操作：

### 1. 在目标服务器上安装 Docker 和 Docker Compose

在目标服务器上安装 Docker 和 Docker Compose。如果目标服务器上已经安装了这些工具，可以跳过此步骤。

```bash
# 更新软件包列表并安装依赖
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release

# 添加阿里云的 Docker GPG 密钥
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# 添加阿里云的 Docker 仓库源
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# 更新 APT 并安装 Docker 和 Docker Compose
sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# 下载 Docker Compose 2.23.3 版本的二进制文件
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-linux-$(uname -m)" -o /usr/local/bin/docker-compose

# 赋予执行权限
sudo chmod +x /usr/local/bin/docker-compose

# 检查安装
docker-compose --version

# 验证 Docker 是否安装成功
sudo docker run hello-world
```

### 2. 拉取代码仓库并创建相应的目录结构

在目标服务器上，创建目录结构并拉取代码。

```bash
mkdir -p ~/project-root/{frontend,backend,nginx}
cd ~/project-root

# 拉取前端代码
cd frontend
git clone <frontend-git-repo> .

# 拉取后端代码
cd ../backend
git clone <backend-git-repo> .

# 创建 Nginx 配置
cd ../nginx
# 复制 nginx 配置文件
echo 'server {
    listen 80;
    location /frontend1/ {
        proxy_pass http://127.0.0.1/frontend:20001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /frontend2/ {
        proxy_pass http://http://127.0.0.1frontend:20007/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /api/ {
        proxy_pass http://http://127.0.0.1backend:10001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location /upload/ {
        alias /usr/share/nginx/html/upload/;
        autoindex on;
    }
}' > default.conf
```

> 确保替换 `<frontend-git-repo>` 和 `<backend-git-repo>` 为你的实际代码仓库 URL。

### 3. 使用 `docker-compose up -d` 命令启动应用

在项目根目录下，使用 Docker Compose 启动应用。

```bash
cd ~/project-root
docker-compose up -d
```

---

## 测试部署

### 测试前端

1. 打开浏览器访问服务器的 IP 地址或域名，如 `http://<服务地址ip>/frontend1/` 或 `http://<server-ip>/frontend2/`。
2. 确保前端应用正常加载，没有报错。

### 测试后端

1. 使用 API 工具（如 Postman）或浏览器访问后端接口，如 `http://<服务地址ip>/api/health`。
2. 确保后端接口正常响应。

### 测试文件上传

1. 在浏览器访问 `http://<服务地址ip>/upload/`。
2. 确保可以访问上传的文件，并且文件列表正常显示。

### 故障排查

如果在测试中遇到问题：

1. **查看容器日志：**

   ```bash
   docker-compose logs <服务名>
   ```

   例如，查看前端服务的日志：

   ```bash
   docker-compose logs frontend
   ```

2. **检查容器状态：**

   ```bash
   docker ps
   ```

   确保所有容器都处于运行状态。

3. **重启容器：**

   如果某个容器出现问题，可以尝试重启：

   ```bash
   docker-compose restart <服务名>
   ```

4. **验证端口映射：**

   确保服务器的防火墙没有阻止必要的端口（如 80、10001、20001、20007）。

---

## 其他注意事项

- **数据持久化**：确保 MySQL 和 Redis 数据目录被持久化，以防止数据丢失。在 `docker-compose.yml` 中使用 `volumes`。
- **安全性**：在生产环境中，不要在 Docker Compose 文件中存储明文密码。使用环境变量或加密服务。
- **监控和日志**：考虑添加监控和日志管理解决方案，如 Prometheus 和 Grafana 或 ELK 堆栈。
- **自动化部署**：可以集成 CI/CD 管道以实现代码推送后的自动化部署。

---

### 总结

通过本指南，你应该能够成功在服务器上设置并部署 Docker 化的全栈应用。如果遇到任何问题，请检查日志和容器状态，确保所有服务都已正确配置和启动。根据需求，可以进一步优化 Docker 配置和部署流程以适应生产环境的要求。

