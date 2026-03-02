# 在 AWS 上部署 OpenClaw 指南

> 说明：这里按 **Docker + EC2** 的方式给出一套稳妥、易排障的部署流程，适合个人与小团队。后续可平滑迁移到 ECS/Fargate。

## 1. 准备工作

- 一个 AWS 账号
- 一个可 SSH 登录的密钥对（`.pem`）
- 已创建的安全组（至少放行）：
  - `22`（仅你的公网 IP）
  - `80`（HTTP）
  - `443`（HTTPS，可选）
  - `3000/8080`（按 OpenClaw 实际端口）
- OpenClaw 的镜像地址（如 Docker Hub / GHCR）或源码仓库

## 2. 创建 EC2 实例

推荐：

- AMI：Ubuntu 22.04 LTS
- 规格：`t3.small` 起步（低流量可 `t3.micro`）
- 磁盘：20GB gp3
- 绑定弹性 IP（EIP），避免重启后公网 IP 变化

## 3. 安装 Docker 运行环境

SSH 登录后执行：

```bash
sudo apt update && sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

## 4. 准备 OpenClaw 配置

创建部署目录：

```bash
mkdir -p ~/openclaw && cd ~/openclaw
```

新建 `.env`（根据 OpenClaw 文档替换变量名）：

```bash
APP_ENV=production
PORT=3000
OPENAI_API_KEY=sk-xxxx
DATABASE_URL=postgres://user:pass@db-host:5432/openclaw
REDIS_URL=redis://redis-host:6379/0
```

## 5. 使用 Docker Compose 部署

创建 `docker-compose.yml`（示例模板，按实际镜像名与端口调整）：

```yaml
services:
  openclaw:
    image: your-registry/openclaw:latest
    container_name: openclaw
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "3000:3000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 5
```

启动：

```bash
docker compose pull
docker compose up -d
```

检查：

```bash
docker compose ps
docker compose logs -f --tail=200
curl http://127.0.0.1:3000/health
```

## 6. 配置反向代理（可选但推荐）

用 Nginx 暴露 80/443，并把请求转发到 `127.0.0.1:3000`。HTTPS 推荐用 Let’s Encrypt：

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

核心反代配置（`/etc/nginx/sites-available/openclaw`）：

```nginx
server {
  listen 80;
  server_name your-domain.com;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

启用并重载：

```bash
sudo ln -s /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

签发证书：

```bash
sudo certbot --nginx -d your-domain.com
```

## 7. 生产建议

- 日志：接入 CloudWatch Agent 或 Loki
- 数据库：优先使用 RDS（不要把数据库和应用绑死在同一台 EC2）
- 缓存：使用 ElastiCache Redis
- 密钥管理：使用 AWS Systems Manager Parameter Store 或 Secrets Manager
- 自动升级：可结合 GitHub Actions 在发版后 SSH 到 EC2 执行 `docker compose pull && docker compose up -d`

## 8. 常见故障排查

1. 页面无法访问：
   - 检查安全组端口是否放行
   - 检查进程监听：`ss -lntp | rg 3000`
2. 容器不断重启：
   - `docker compose logs --tail=200 openclaw`
   - 核对 `.env` 变量是否齐全
3. 域名可访问但 502：
   - 检查 Nginx upstream 端口
   - 检查容器健康检查是否失败

---

如果你愿意，我可以下一步按你实际的 OpenClaw 仓库结构，直接给你一份 **可复制即用** 的：

- `.env` 模板
- `docker-compose.yml`
- `nginx` 配置
- 以及一份 GitHub Actions 自动部署脚本
