# podman 安装

```bash
sudo apt update
# 安装并检查安装
sudo apt install podman
podman --version


# 配置监听 socket
sudo systemctl enable podman.socket
sudo systemctl start podman.socket
sudo systemctl status podman.socket


systemctl --user enable podman.socket --now
systemctl --user start podman.socket
systemctl status podman.socket
```


# podman 构建多架构镜像

podman farm 还存在一堆问题, 不太可用

```bash
# github 登录
export ghcr_user="xxx"
export ghcr_token="yyyyyy"

echo "$ghcr_token" | podman login ghcr.io -u "$ghcr_user" --password-stdin
podman login --get-login ghcr.io

echo "$ghcr_token" | podman --connection kubuntu26 login ghcr.io -u "$ghcr_user" --password-stdin
podman --connection kubuntu26 login --get-login ghcr.io

export tsdb_cn_version='v1.1.1'

# 在不同架构主机上构建并推送 ghcr
podman --connection kubuntu26 build -t localhost/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64 .
podman --connection orb-box build -t localhost/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64 .

podman --connection kubuntu26 push localhost/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64 ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64
podman --connection orb-box push localhost/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64 ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64

# 在 kubuntu26 上
podman push localhost/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64 ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64

# 把不同架构镜像拉到同一台机器上
podman pull ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64
podman pull ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64


# 创建 manifest list
podman manifest create tsdb-bm25cn-multi-arch

# 添加各架构镜像
podman manifest add tsdb-bm25cn-multi-arch ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64
podman manifest add tsdb-bm25cn-multi-arch ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64

# 推送最终 manifest list
podman manifest push tsdb-bm25cn-multi-arch ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}

# 清理临时标签（可选）
podman rmi ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64 ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64
```

