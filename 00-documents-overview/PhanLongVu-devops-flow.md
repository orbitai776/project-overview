# **Document flow devops mcp node**

**Người viết:** Phan Long Vũ	

**Ngày cập nhật cuối:** 25/3/2026	

**Phiên bản tài liệu:** 0.1

- [**Document flow devops mcp node**](#document-flow-devops-mcp-node)
  - [**1. QUY TRÌNH AUTOMATION WORKFLOW (CI - TÍCH HỢP LIÊN TỤC)**](#1-quy-trình-automation-workflow-ci---tích-hợp-liên-tục)
    - [**1.1. Thiết lập Secret Keys và Environments**](#11-thiết-lập-secret-keys-và-environments)
    - [**1.2. Luồng thực thi CI (CI Pipeline Flow)**](#12-luồng-thực-thi-ci-ci-pipeline-flow)
  - [**2. QUY TRÌNH DEPLOY LÊN VPS (CD - TRIỂN KHAI LIÊN TỤC)**](#2-quy-trình-deploy-lên-vps-cd---triển-khai-liên-tục)
    - [**2.1. Xác thực và Kết nối (Authentication):**](#21-xác-thực-và-kết-nối-authentication)
    - [**2.2. Chuyển giao phiên bản mới :**](#22-chuyển-giao-phiên-bản-mới-)
    - [**2.3. Thực thi Triển khai (Execute \& Restart):**](#23-thực-thi-triển-khai-execute--restart)
    - [**2.4. Dọn dẹp (Cleanup):**](#24-dọn-dẹp-cleanup)
  - [**3. QUY TRÌNH SETUP DOMAIN, NGINX VÀ SSL**](#3-quy-trình-setup-domain-nginx-và-ssl)
    - [**3.1. Cài đặt SSL tự động với Certbot (Let's Encrypt)**](#31-cài-đặt-ssl-tự-động-với-certbot-lets-encrypt)
    - [**3.2. Cấu hình Nginx Server Block**](#32-cấu-hình-nginx-server-block)
  - [**4. QUY TRÌNH SETUP CLOUDFLARE RECORD A**](#4-quy-trình-setup-cloudflare-record-a)
    - [**4.1. Các bước cấu hình trên Dashboard Cloudflare**](#41-các-bước-cấu-hình-trên-dashboard-cloudflare)
    - [**4.2. Các dns record bao gồm**:](#42-các-dns-record-bao-gồm)

## **1\. QUY TRÌNH AUTOMATION WORKFLOW (CI \- TÍCH HỢP LIÊN TỤC)**

Phần này mô tả luồng logic tích hợp liên tục (CI) sử dụng công cụ Automation (như GitHub Actions, GitLab CI). Mục tiêu của phần này là tự động hóa các khâu kiểm tra và đóng gói mỗi khi có code mới, áp dụng linh hoạt cho mọi loại ngôn ngữ (Node.js, Python, PHP...).

### **1.1. Thiết lập Secret Keys và Environments**

Secret lưu các thông tin bảo mật của hệ thống, không được lộ ra bên ngoài.  
Danh sách Secret key của hệ thống gồm:

* SERVER\_IP: IP của vps  
* SERVER\_USER: user dùng để chạy app lên).  
* SSH\_PRIVATE\_KEY: nội dung ssh key để ssh vào vps.

ENV lưu các thông tin không nhạy cảm của hệ thống, cần trong quá trình build, test, deploy. Giá trị của ENV trong 3 môi trường dev, stg, production sẽ được config khác nhau  
Danh sách ENV:

* CONTAINER: tên container sẽ được chạy lên vps  
* ENV: tên môi trường.  
* PORT: tên port được chạy lên vps.  
* REGISTRY: tên của image registry

### **1.2. Luồng thực thi CI (CI Pipeline Flow)**

Dù dự án viết bằng công nghệ nào, luồng CI chuẩn mực luôn trải qua 4 giai đoạn cơ bản sau:

1. **Giai đoạn Kích hoạt (Trigger):** Pipeline được kích hoạt tự động khi có sự kiện  **PUSH(push code)** lên nhánh **`dev,stg,main`**..  
``` yml
on:  
  push:  
    branches: [ "main", "stg", "dev" ]  
  workflow_dispatch:
```  
2. **Giai đoạn Build:** Runner được kích hoạt sẽ chạy các lệnh để build project  
* Bước 1: setup môi trường build: 
``` yml 
runs-on: ubuntu-latest
  permissions:
    contents: read
    packages: write
```
* Bước 2: Login vào registry
``` yml
steps:
  - uses: actions/checkout@v4
  - name: Login to GHCR
    uses: docker/login-action@v3
    with:
      registry: ${{ env.REGISTRY }}
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB\_TOKEN }}
```
* Bước 3: build image bằng docker và push lên registry.
``` yml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ${{ env.REGISTRY }}/${{ env.IMAGE\_NAME }}:${{ github.ref\_name }}
      ${{ env.REGISTRY }}/${{ env.IMAGE\_NAME }}:${{ github.sha }}`
```
## **2\. QUY TRÌNH DEPLOY LÊN VPS (CD \- TRIỂN KHAI LIÊN TỤC)**

Luồng triển khai (CD) là bước nối tiếp ngay sau khi luồng CI (Phần 1\) đã chạy thành công. Nhiệm vụ của luồng này là đưa phiên bản mới nhất lên VPS đích và khởi chạy.

### **2.1. Xác thực và Kết nối (Authentication):**  
   Bước 1: get key ssh và tiến hành setup để runner ssh lên được vps.
``` bash
        mkdir -p ~/.ssh  
             echo "${{ secrets.SSH_PRIVATE_KEY }}" \> ~/.ssh/deploy_key  
             chmod 600 ~/.ssh/deploy_key  
             ssh-keyscan -H ${{ secrets.SERVER_IP }} >> ~/.ssh/known_hosts
```
  Bước 2: Deploy lên vps. Lệnh ssh sẽ như sau: 
``` bash
ssh -i ~/.ssh/deploy_key ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_IP }} << EOF

# … các lệnh ở bước tiếp theo được thực thi ở đây

EOF
```
### **2.2. Chuyển giao phiên bản mới :** 
Ta sẽ pull image được up ở giai đoạn ci về đây.  
Bước 1: Login vào ghcr bằng github_token
``` bash
echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
```
Bước 2: Pull image về ở giai đoạn build bước 3.
``` bash
docker pull ghcr.io/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
```
     
### **2.3. Thực thi Triển khai (Execute & Restart):** 
Bước này tiến hành clear các container cũ và chạy container mới lên  
   Bước 1:Xóa các container cũ đi (nếu có):

    docker stop $CONTAINER 2>/dev/null || true  
    docker rm $CONTAINER 2>/dev/null || true

	Bước 2: Chạy container mới lên. 

Minh họa với nestjs
``` bash
docker run -d 

              --name $CONTAINER \

              --restart unless-stopped \

              -p $PORT:3000 \

              -e NODE_ENV=$ENV \

              -e PORT=3000 \

              --log-opt max-size=10m \

              --log-opt max-file=3 \

              ghcr.io/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
```
Minh họa với golang:
``` bash
docker run -d \

              --name $CONTAINER \

              --restart unless-stopped \

              -p $PORT:8080 \

              -e GIN_MODE=release \

              -e PORT=8080 \

              -e ENV=$ENV \

              --log-opt max-size=10m \

              --log-opt max-file=3 \

              $REGISTRY/$IMAGE_NAME:$BRANCH_NAME
```
Minh họa với python:
``` bash
docker run -d \

              --name $CONTAINER \

              --restart unless-stopped \

              -p $PORT:8080 \

              -e SERVICE_PORT=8080\

	  -e DEBUG=false \

              --log-opt max-size=10m \

              --log-opt max-file=3 \

	    $REGISTRY/$IMAGE_NAME:$BRANCH_NAME
```
Minh họa với php:
``` bash
docker run -d \

              --name $CONTAINER \

              --restart unless-stopped \

              -p $PORT:8080 \

              -e  APP_NAME=$CONTAINER. \

  -e  APP_ENV=$ENV \

              -e PORT=$PORT \

              --log-opt max-size=10m \

              --log-opt max-file=3 \

	    $REGISTRY/$IMAGE_NAME:$BRANCH_NAME
```
### **2.4. Dọn dẹp (Cleanup):** 
Dọn dẹp các image cũ và không xài nữa đi:  
``` bash
docker image prune -f  
```  

## **3\. QUY TRÌNH SETUP DOMAIN, NGINX VÀ SSL**

Phần này mô tả cách cấu hình Web Server (Nginx) để nhận tên miền và cài đặt chứng chỉ bảo mật HTTPS (SSL) tự động.

### **3.1. Cài đặt SSL tự động với Certbot (Let's Encrypt)**

Chạy các lệnh sau trên VPS:
``` bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d <domain> -d www.<domain>
```
Nếu ufw đang bật thì nhớ cho phép port 80 bằng lệnh:
``` bash
sudo ufw allow 80
```
### **3.2. Cấu hình Nginx Server Block**

Tạo file cấu hình tại `/etc/nginx/sites-enables/<site-domain>.conf`:
``` nginx
server {
  listen 80;

  server_name <site-domain> // nếu traffic target site là <site-domain>

  location / {
    proxy_pass 127.0.0.1:<port> # thì traffic được forward qua port này.
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
  }
}
```
Kích hoạt cấu hình:
``` bash
nginx -t  # Kiểm tra cú pháp
sudo systemctl reload nginx # load cấu hình nginx`
```
## **4\. QUY TRÌNH SETUP CLOUDFLARE RECORD A** 

### **4.1. Các bước cấu hình trên Dashboard Cloudflare**

1. Đăng nhập vào Cloudflare, chọn tên miền cần cấu hình.  
2. Điều hướng đến menu **DNS** \-\> **Records**.  
3. Bấm **Add record** và điền các thông số:  
   * **Type:** `A`  
   * **Name:** tên subdomain   
   * **IPv4 address:** `[Nhập địa chỉ IP Public của VPS]`  
   * **Proxy status:** tắt  
4. Bấm **Save**.

### **4.2. Các dns record bao gồm**: 

| Environment |  | Endpoint |
| :---- | :---- | :---- |
| DEV | Web Platform | [https://platform-dev.orbitai.fun/](https://platform-dev.orbitai.fun/) |
|  | Gateway | [https://platform-gateway-dev.orbitai.fun/api/health/status](https://platform-gateway-dev.orbitai.fun/api/health/status) |
|  | Service Backend NestJS | [https://platform-backend-dev.orbitai.fun/](https://platform-backend-dev.orbitai.fun/) |
|  | Services AI Multi-turn Slot Filling |  |
|  | Service Backend PHP |  | [https://platform-backend-php-dev.orbitai.fun/]
|  | Service Backend Python |  |
|  | Service Backend NetCore |  |
|  | Service Backend Java |  |
|  |  |  |
| STAGING | Web Platform | [https://platform-stg.orbitai.fun/](https://platform-stg.orbitai.fun/) |
|  | Gateway | [https://platform-gateway-stg.orbitai.fun/api/health/status](https://platform-gateway-stg.orbitai.fun/api/health/status) |
|  | Service Backend NestJS | [https://platform-backend-stg.orbitai.fun/](https://platform-backend-stg.orbitai.fun/) |
|  | Services AI Multi-turn Slot Filling |  |
|  | Service Backend PHP |  | [https://platform-backend-php-stg.orbitai.fun/]
|  | Service Backend Python |  |
|  | Service Backend NetCore |  |
|  | Service Backend Java |  |
|  |  |  |
| PRODUCTION | Web Platform | [https://platform.orbitai.fun/](https://platform.orbitai.fun/) |
|  | Gateway | [https://platform-gateway.orbitai.fun/api/health/status](https://platform-gateway.orbitai.fun/api/health/status) |
|  | Service Backend NestJS | [https://platform-backend.orbitai.fun/](https://platform-backend.orbitai.fun/) |
|  | Services AI Multi-turn Slot Filling |  |
|  | Service Backend PHP |  |
|  | Service Backend Python |  |
|  | Service Backend NetCore |  |
|  | Service Backend Java |  |
