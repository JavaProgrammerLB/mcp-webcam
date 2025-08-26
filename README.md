# 局域网访问 React/Node 服务完整笔记

> 目标：让你在本机启动的 React/Node 服务能被局域网（手机/其他设备）通过 **HTTPS** 访问，并支持摄像头/麦克风 API。

---

## ✅ 快速 Checklist
- [ ] 用 `BIND_HOST=0.0.0.0` 启动服务，确保对外监听。
- [ ] 查出电脑当前局域网 IP（热点/Wi-Fi）。
- [ ] 用 `mkcert` 安装本地根证书并签出站点证书。
- [ ] 在 **iPhone** 安装并“完全信任”根证书。
- [ ] 启动 **HTTPS 反向代理**（local-ssl-proxy / Caddy / Nginx）。
- [ ] 用 `https://<电脑IP>:<HTTPS端口>/` 访问并验证 `navigator.mediaDevices` 可用。

---

## 1) 如何绑定 `0.0.0.0`
```bash
BIND_HOST=0.0.0.0 PORT=3333 node dist/server.js --streaming
```

验证：
```bash
lsof -iTCP:3333 -sTCP:LISTEN
# 看到 TCP *:3333 (LISTEN) 说明已绑定 0.0.0.0
```

---

## 2) 如何查询电脑的 IP
- **macOS**
  ```bash
  ipconfig getifaddr en0
  ```
- **Windows**
  ```powershell
  ipconfig
  ```
- **Linux**
  ```bash
  hostname -I
  ```

---

## 3) 生成证书（mkcert）
```bash
brew install mkcert
mkcert -install
mkdir -p certs
mkcert -key-file certs/lan.key -cert-file certs/lan.crt 172.20.10.2 localhost 127.0.0.1
```

根证书位置：
```bash
mkcert -CAROOT
# 目录下 rootCA.pem
```

---

## 4) 在 iPhone 上安装并信任证书
1. AirDrop `rootCA.pem` 到 iPhone  
2. 设置 → 已下载描述文件 → 安装  
3. 设置 → 通用 → 关于本机 → 证书信任设置 → 开启完全信任  

---

## 5) 开启 HTTPS 反向代理

### 方法 A：local-ssl-proxy
```bash
npx local-ssl-proxy --source 3443 --target 3333 \
  --cert certs/lan.crt --key certs/lan.key
```
访问：
```
https://<电脑IP>:3443/?user=default
```

### 方法 B：Caddy
```caddy
https://<电脑IP>:3443 {
  tls certs/lan.crt certs/lan.key
  reverse_proxy 127.0.0.1:3333
}
```
```bash
caddy run --config ./Caddyfile
```

### 方法 C：Nginx
```nginx
server {
    listen 3443 ssl;
    server_name <电脑IP>;
    ssl_certificate certs/lan.crt;
    ssl_certificate_key certs/lan.key;
    location / {
        proxy_pass http://127.0.0.1:3333;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## 6) 注意事项
- iPhone/电脑同一网络  
- 每次 IP 可能变：`ipconfig getifaddr en0`  
- 证书 SAN 中必须包含访问的 IP/域名  
- 摄像头/麦克风只能在 HTTPS 或 localhost 下使用  

---

## 7) 固定域名方案 (dev.local)
```bash
mkcert dev.local localhost 127.0.0.1 ::1
```
在 `/etc/hosts`：
```
<电脑IP> dev.local
```
Caddy 配置：
```caddy
https://dev.local:3443 {
  tls certs/dev.local.crt certs/dev.local.key
  reverse_proxy 127.0.0.1:3333
}
```
以后访问：
```
https://dev.local:3443/?user=default
```
