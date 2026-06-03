# 自包含镜像（把工具打进镜像，不依赖挂载）。
# 构建需在仓库根目录执行（build context = 仓库根）：
#   docker build -f deploy/Dockerfile -t logistics-survey .
#   docker run -d --name logistics-survey --restart unless-stopped -p 8327:80 logistics-survey
FROM nginx:alpine
COPY deploy/nginx.conf /etc/nginx/conf.d/default.conf
COPY tools/ /usr/share/nginx/html/
