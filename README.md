
```markdown
# Docker项目部署

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

### 配置阿里云加速镜像
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

```dockerfile
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
```

### 后端 Dockerfile

```dockerfile
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
```

---

## 创建 Nginx 配置

```nginx
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
```

---

## 创建 `docker-compose.yml`

```yaml
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
      - SPRING

_MAIL_HOST=smtp.126.com
      - SPRING_MAIL_PORT=465
      - SPRING_MAIL_USERNAME=admin
      - SPRING_MAIL_PASSWORD=admin@123
      - ARCHIVE_MQ_EXCHANGE=archive.exchange
      - ARCHIVE_MQ_QUEUE=archive.queue
      - ARCHIVE_MQ_ROUTING_KEY=archive.routingkey
      - ARCHIVE_TIME_CONFIG= 60000
  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile
    ports:
      - "80:80"
    depends_on:
      - frontend
      - backend

    networks:
      - backend
      - frontend

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    networks:
      - backend

  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=archive
      - MYSQL_USER=test
      - MYSQL_PASSWORD=test@123
    volumes:
      - ./mysql:/var/lib/mysql
    networks:
      - backend

networks:
  backend:
  frontend:
```

---

## 启动 Docker Compose

```bash
docker-compose up -d
```

---

## 自动更新 Docker 容器

### 设置裸仓库和工作副本

```bash
mkdir -p ~/bare-repo/archive.git
cd ~/bare-repo/archive.git
git init --bare
```

### 配置 Git 钩子

```bash
nano hooks/post-receive
```

在 `post-receive` 文件中添加以下内容：

```bash
#!/bin/bash
echo "Deploying the latest changes..."
GIT_WORK_TREE=~/project-root/frontend git checkout -f
GIT_WORK_TREE=~/project-root/backend git checkout -f
cd ~/project-root
docker-compose down
docker-compose up -d --build
echo "Deployment successful."
```

保存并退出，然后设置文件可执行权限：

```bash
chmod +x hooks/post-receive
```

### 在本地推送代码

```bash
cd ~/project-root/frontend
git remote add deploy ~/bare-repo/archive.git
git push deploy master

cd ~/project-root/backend
git remote add deploy ~/bare-repo/archive.git
git push deploy master
```

### 配置 `docker-compose.yml`

确保 `docker-compose.yml` 中的镜像名称和服务名称相同，并包括版本。

```yaml
services:
  frontend:
    image: registry.gitlab.com/your-user/your-project/frontend:latest

  backend:
    image: registry.gitlab.com/your-user/your-project/backend:latest

  nginx:
    image: registry.gitlab.com/your-user/your-project/nginx:latest
```

---

## 将 Docker 应用部署到其他服务器

### 导出 Docker 镜像

```bash
docker save -o archive.tar frontend backend nginx
```

### 传输 Docker 镜像

```bash
scp archive.tar user@remote-server:/path/to/save
```

### 导入 Docker 镜像

```bash
ssh user@remote-server
docker load -i /path/to/archive.tar
```

### 启动 Docker 应用

```bash
docker-compose -f /path/to/docker-compose.yml up -d
```

---

