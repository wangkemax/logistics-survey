# 在 NAS 上用 Docker 跑（极空间 + Lucky 反代）

本工具是一组静态文件（`tools/` 下的 `index.html` + `manifest.webmanifest` + `sw.js` + 图标），
用 nginx 容器以**纯 http** 提供服务，前面交给你现成的 **Lucky** 反代成 HTTPS 域名即可。

> 手机经 Lucky 访问看到的是 `https://你的域名`，属于安全上下文，所以
> **Service Worker / 添加到主屏 / 离线缓存** 全部可用；http 只存在于 Lucky→容器的内网一段。

---

## 一、放文件

把仓库里的 `deploy/` 和 `tools/` 两个目录上传到 NAS（保持相对位置：`deploy/` 与 `tools/` 同级）。
例如放到 `/vol1/docker/logistics-survey/` 下，得到：

```
logistics-survey/
├── deploy/   (docker-compose.yml / nginx.conf / Dockerfile)
└── tools/    (index.html 等)
```

## 二、起容器（二选一）

**方式 A · docker compose（推荐，改 HTML 不用重建）**

```bash
cd /vol1/docker/logistics-survey/deploy
docker compose up -d
```

容器把 `../tools` 只读挂载进 nginx。以后更新工具：替换 `tools/index.html`，
`docker compose restart`（或客户端直接刷新，nginx 对 index.html / sw.js 已设 no-cache）。

> 极空间「Docker / 容器管理」里也可以：上传本项目 → 用 compose 文件创建项目；
> 或在图形界面手动建一个 nginx:alpine 容器，映射端口 `8087:80`，
> 挂载 `tools → /usr/share/nginx/html`、`deploy/nginx.conf → /etc/nginx/conf.d/default.conf`。

**方式 B · 自包含镜像（把工具打进镜像）**

```bash
cd /vol1/docker/logistics-survey      # 注意在仓库根目录
docker build -f deploy/Dockerfile -t logistics-survey .
docker run -d --name logistics-survey --restart unless-stopped -p 8087:80 logistics-survey
```

## 三、本机自测

```bash
curl -I http://<NAS局域网IP>:8087/index.html              # 200
curl -I http://<NAS局域网IP>:8087/manifest.webmanifest    # Content-Type: application/manifest+json
```

## 四、Lucky 反代

在 Lucky 里加一条 Web 服务/反向代理规则：

- 域名/前缀：`survey.你的域名`（按你习惯）
- 后端目标：`http://<NAS局域网IP>:8087`
- 证书：用你 Lucky 已配置的那张（Let's Encrypt / DDNS 证书）

保存后，手机浏览器打开 `https://survey.你的域名` → 菜单「添加到主屏幕」，即得离线可用的 App 图标。

## 五、注意

- 端口 `8087` 可自改，改 `docker-compose.yml` 的 `ports` 与 Lucky 后端目标即可。
- `tools/` 是只读挂载，容器内不会改你的文件。
- AI 图注仍走你在工具「设置」里填的 MiniMax 接口（key 只存手机本地浏览器，不经此容器）。
- 数据存在手机浏览器本地（localStorage / IndexedDB），不在 NAS。务必现场调研后用工具内「备份」导出归档。
