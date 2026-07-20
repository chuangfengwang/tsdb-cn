# 基础镜像：TimescaleDB HA PG17 (基于 Ubuntu)
FROM docker.io/timescale/timescaledb-ha:pg17

USER root

# 环境变量配置
ENV PATH="/usr/lib/postgresql/17/bin:${PATH}"
# SCWS 版本 (锁定版本以保证稳定性)
ENV SCWS_VERSION=1.2.3
# pg_textsearch 版本 (锁定版本以保证稳定性)
ENV PG_TEXTSEARCH_TAG=v1.3.1

# ----------------------------------------------------------
# 1. 安装系统依赖
# ----------------------------------------------------------
# build-essential: 编译 C/C++ 扩展必备
# cmake: 编译 pg_jieba 必备
# postgresql-server-dev-17: 编译 PG 扩展必备
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    wget \
    git \
    cmake \
    ca-certificates \
    postgresql-server-dev-17 \
    && rm -rf /var/lib/apt/lists/*

# ----------------------------------------------------------
# 2. 安装 SCWS 和 zhparser (方案 A: 基于词典的分词)
# ----------------------------------------------------------
COPY dep-src/scws-${SCWS_VERSION}.tar.bz2 /tmp/
RUN tar xjf /tmp/scws-${SCWS_VERSION}.tar.bz2 -C /tmp \
    && cd /tmp/scws-${SCWS_VERSION} \
    && ./configure --prefix=/usr/local \
    && make -j$(nproc) \
    && make install \
    && rm -rf /tmp/scws-${SCWS_VERSION} /tmp/scws-${SCWS_VERSION}.tar.bz2

# 下载并安装 SCWS (如果不使用本地文件, 可以直接从官网下载)
#RUN wget -q -O - https://www.xunsearch.com/scws/down/scws-${SCWS_VERSION}.tar.bz2 | tar xjf - \
#    && cd scws-${SCWS_VERSION} \
#    && ./configure --prefix=/usr/local \
#    && make -j$(nproc) \
#    && make install \
#    && cd .. \
#    && rm -rf scws-${SCWS_VERSION}

RUN git clone https://github.com/amutu/zhparser.git \
    && cd zhparser \
    && make \
    && make install \
    && cd .. \
    && rm -rf zhparser

# ----------------------------------------------------------
# 3. 安装 pg_jieba (方案 B: 基于 HMM/结巴算法的分词)
# ----------------------------------------------------------
# 注意：必须使用 --recursive 拉取子模块
RUN git clone https://github.com/jaiminpan/pg_jieba.git \
    && cd pg_jieba \
    && git submodule update --init --recursive \
    && mkdir build \
    && cd build \
    && cmake -DPostgreSQL_TYPE_INCLUDE_DIR=/usr/include/postgresql/17/server -DPostgreSQL_INCLUDE_DIR=/usr/include/postgresql -DPostgreSQL_BIN_DIR=/usr/lib/postgresql/17/bin .. \
    && make -j$(nproc) \
    && make install \
    && cd ../.. \
    && rm -rf pg_jieba

# ----------------------------------------------------------
# 4. 安装 pg_textsearch (BM25 相关性算法)
# ----------------------------------------------------------
RUN git clone https://github.com/timescale/pg_textsearch.git \
    && cd pg_textsearch \
    && git checkout ${PG_TEXTSEARCH_TAG} \
    && make \
    && make install \
    && cd .. \
    && rm -rf pg_textsearch

# ----------------------------------------------------------
# 5. 覆盖配置文件, 启用 pg_textsearch,zhparser,pg_jieba 库
# ----------------------------------------------------------
COPY postgresql.conf /home/postgres/pgdata/data/postgresql.conf

# ----------------------------------------------------------
# 6. 清理构建环境
# ----------------------------------------------------------
# 卸载仅编译时需要的包，减小镜像体积
RUN apt-get purge -y --auto-remove \
    build-essential \
    postgresql-server-dev-17 \
    cmake

# 切换回 postgres 用户 (HA 镜像运行要求)
USER postgres
