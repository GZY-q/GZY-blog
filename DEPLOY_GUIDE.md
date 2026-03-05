# 贡正阳个人博客网站部署指南

本文档将指导你如何将基于纯静态技术（HTML/CSS/JS）开发的个人博客部署到你的云服务器上，并完成域名解析、Nginx 配置及 HTTPS 证书配置。

---

## 概览

- **服务器环境限制：** Linux 操作系统（推荐 Ubuntu 20.04/22.04 LTS 或 CentOS/AlmaLinux）
- **Web 服务器：** Nginx
- **部署类型：** 纯静态文件部署
- **前置准备：**
  - 一台具有公网 IP 的云服务器
  - 已备案的域名（若服务器在中国大陆）
  - 获取到的云服务器 SSH 登录凭证

---

## 阶段一：服务器环境初始化

### 1. 更新系统包并安装 Nginx
通过 SSH 连接到云服务器后，执行以下命令安装 Nginx。

**对于 Ubuntu/Debian 系统：**
```bash
sudo apt update
sudo apt install nginx -y
```

**对于 CentOS/RHEL 系统：**
```bash
sudo yum install epel-release -y
sudo yum install nginx -y
```

安装完成后，启动并设置 Nginx 为开机自启：
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

> **注意：** 确保云服务器的安全组（防火墙）已经开放了 `80`（HTTP）和 `443`（HTTPS）端口。

---

## 阶段二：上传网站静态文件

网站源代码包含了直接可用的 HTML、CSS 和 JS 文件，无需复杂的构建步骤。我们需要将这些文件上传到服务器的 Web 目录中。

1. **在服务器上创建存放网站的目录：**
   ```bash
   sudo mkdir -p /var/www/gzy-blog
   sudo chown -R $USER:$USER /var/www/gzy-blog
   sudo chmod -R 755 /var/www/gzy-blog
   ```

2. **将本地代码上传到服务器：**
   （假设您在本地终端，将其中的 `root@your_server_ip` 替换为实际的用户和 IP）
   ```bash
   # 使用 scp 上传目录中的所有文件到服务器
   scp -r ./* root@your_server_ip:/var/www/gzy-blog/
   ```

---

## 阶段三：配置 Nginx 虚拟主机

1. **创建一个新的 Nginx 配置文件：**
   ```bash
   sudo nano /etc/nginx/conf.d/gzy-blog.conf
   # 注意：如果你使用的是 Ubuntu，可能配置路径是在 /etc/nginx/sites-available/ 并需要软链接到 sites-enabled。本指南以通用兼容性 conf.d 为主。
   ```

2. **写入基础的 HTTP 监听配置（暂不包含 HTTPS）：**
   将以下内容复制进去（将 `your_domain.com` 替换为你的真实域名，例如 `gede.asia` 或子域名 `blog.gede.asia`）：
   ```nginx
   server {
       listen 80;
       server_name your_domain.com; # 替换为你的实际域名，例如 blog.gede.asia
       root /var/www/gzy-blog;
       index index.html;

       access_log /var/log/nginx/gzy-blog.access.log;
       error_log /var/log/nginx/gzy-blog.error.log;

       # 路由配置 - 支持静态文件的默认加载
       location / {
           try_files $uri $uri/ /index.html;
       }

       # 开启 Gzip 压缩，提高加载速度
       gzip on;
       gzip_types text/plain text/css application/javascript application/json image/svg+xml;
       gzip_min_length 1000;

       # 设置静态文件缓存策略
       location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf)$ {
           expires 30d;
           add_header Cache-Control "public, no-transform";
       }
   }
   ```

3. **测试并重载 Nginx 配置：**
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```
   > 此时，只要你的域名在云服务商处解析到了这台服务器的公网 IP，就可以通过 `http://your_domain.com` 访问网站了。

---

## 阶段四：配置 HTTPS 证书

配置 HTTPS 能让你的网站显示“安全”锁，并且由于我们的 WebRTC 等应用可能强制要求 HTTPS 协议，对于个人博客同样建议启用。

推荐使用 `Certbot` 服务自动获取并配置 Let's Encrypt 免费证书。

### 1. 安装 Certbot
**对于 Ubuntu：**
```bash
sudo apt install certbot python3-certbot-nginx -y
```
**对于 CentOS：**
```bash
sudo yum install certbot python3-certbot-nginx -y
```

### 2. 自动生成并配置证书
执行以下命令，Certbot 将自动读取 Nginx 配置、申请证书并修改你的 `gzy-blog.conf` 以将 HTTP 强制跳转为 HTTPS。
```bash
sudo certbot --nginx -d your_domain.com
```
按照终端提示进行：
- 输入你的邮箱地址（如：gede-gzy@qq.com，用于续期提醒）
- 同意服务条款 (A)
- 是否分享邮箱 (N)
- **当询问是否重定向 HTTP 到 HTTPS 时，选择 2 (Redirect) 以开启强制跳转。**

### 3. 测试自动续期
Let's Encrypt 证书的有效期是 90 天，Certbot 安装后会自动添加一个定时任务进行续期。你可以靠以下命令测试它：
```bash
sudo certbot renew --dry-run
```

如果未报错，则代表整个 HTTPS 部署彻底大功告成。

---

## 维护与更新指南

本项目没有使用数据库与后端复杂的依赖，更新过程非常极简：

由于我们在本地完成代码变更后，只需再次覆盖式上传最新的静态文件即可。
```bash
# 在你本地终端执行
scp -r ./index.html ./styles.css ./script.js ./articles.html ./article.html ./about.html root@your_server_ip:/var/www/gzy-blog/
```
上传后，如果浏览器没有立刻体现更改，可强制刷新网页（`Ctrl + F5` 或 清除浏览器缓存），也可以在 Nginx 配置中清理静态缓存。

**至此，你的个人专业博客已完全部署上线。**
