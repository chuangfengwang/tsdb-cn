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

export tsdb_cn_version='v1.1.2'

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
podman manifest create tsdb-bm25cn-multi-arch-${tsdb_cn_version}

# 添加各架构镜像
podman manifest add tsdb-bm25cn-multi-arch-${tsdb_cn_version} ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64
podman manifest add tsdb-bm25cn-multi-arch-${tsdb_cn_version} ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64

# 推送最终 manifest list
podman manifest push tsdb-bm25cn-multi-arch-${tsdb_cn_version} ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}

# 清理 manifest 避免影响后续构建
podman manifest rm tsdb-bm25cn-multi-arch-${tsdb_cn_version}

# 清理临时标签（可选）
podman rmi ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-amd64 ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-${tsdb_cn_version}-arm64
```

# local test

```bash
# 启动本地镜像
export tsdb_password="xxxxxx"

mkdir -p tsdb_data

podman run \
--name tsdb-local \
-e POSTGRES_PASSWORD=${tsdb_password} \
-p 15435:5432 \
-v $(pwd)/tsdb_data:/home/postgres/pgdata/data:Z,U \
localhost/timescale-bm25-cn:pg17-v1.1.2-arm64

# 进入容器检查文件配置
podman exec -it tsdb-local /bin/bash

# 在宿主机连接数据库
psql -h podman-box.orb.local -p 15435 -U postgres -d postgres

```

在容器中, 检查 /home/postgres/pgdata/data/postgresql.auto.conf 中是否启用了插件,且插件参数是否有配置. 
具体来说, 配置文件应包含下面的配置行
```text
zhparser.dict_in_memory = 'on'
shared_preload_libraries = 'pg_stat_statements, timescaledb, pg_textsearch, zhparser, pg_jieba'
```

检查 sql 效果
```postgresql

-- 确认插件生效
SHOW shared_preload_libraries;

create database pg_search_demo;
\c pg_search_demo

-- 检查 pg_textsearch + pg_jieba 效果
CREATE EXTENSION IF NOT EXISTS pg_textsearch;
CREATE EXTENSION IF NOT EXISTS pg_jieba;

-- 创建自定义分词器配置(log_jiebacfg 是自定义的分词器名称)
DROP TEXT SEARCH CONFIGURATION IF EXISTS public.log_jiebacfg;

CREATE TEXT SEARCH CONFIGURATION public.log_jiebacfg (PARSER = jieba);

-- 我们排除了 x(未知) 的词性
ALTER TEXT SEARCH CONFIGURATION public.log_jiebacfg 
ADD MAPPING FOR 
 a   , -- adjective
 ad  , -- ad
 ag  , -- ag
 an  , -- an
 b   , -- difference
 c   , -- conjunction
 d   , -- adverb
 df  , -- df
 dg  , -- dg
 e   , -- exclamation
 eng , -- letter
 f   , -- direction noun
 g   , -- morpheme
 h   , -- h
 i   , -- idiom
 j   , -- abbreviate
 k   , -- k
 l   , -- temporary idiom
 m   , -- numeral
 mg  , -- mg
 mq  , -- numeral-classifier compound
 n   , -- noun
 ng  , -- ng
 nr  , -- person's name
 nrfg, -- nrfg
 nrt , -- nrt
 ns  , -- location
 nt  , -- organization
 nz  , -- other proper noun
 o   , -- onomatopoeia
 p   , -- prepositional
 q   , -- quantity
 r   , -- pronoun
 rg  , -- rg
 rr  , -- rr
 rz  , -- rz
 s   , -- space
 t   , -- time
 tg  , -- tg
 u   , -- auxiliary
 ud  , -- ud
 ug  , -- ug
 uj  , -- uj
 ul  , -- ul
 uv  , -- uv
 uz  , -- uz
 v   , -- verb
 vd  , -- vd
 vg  , -- vg
 vi  , -- vi
 vn  , -- vn
 vq  , -- vq
-- x   , -- unknown
 y   , -- modal verbs
 z   , -- z
 zg   -- zg
WITH simple;

-- 验证分词效果
SELECT * FROM ts_debug('public.log_jiebacfg', 'PostgreSQL结合结巴分词非常强大	database system');

 alias | description |   token    | dictionaries | dictionary |   lexemes
-------+-------------+------------+--------------+------------+--------------
 eng   | letter      | PostgreSQL | {simple}     | simple     | {postgresql}
 v     | verb        | 结合       | {simple}     | simple     | {结合}
 n     | noun        | 结巴       | {simple}     | simple     | {结巴}
 n     | noun        | 分词       | {simple}     | simple     | {分词}
 d     | adverb      | 非常       | {simple}     | simple     | {非常}
 a     | adjective   | 强大       | {simple}     | simple     | {强大}
 x     | unknown     |            | {}           |            |
 eng   | letter      | database   | {simple}     | simple     | {database}
 x     | unknown     |            | {}           |            |
 eng   | letter      | system     | {simple}     | simple     | {system}
(10 行记录)

CREATE INDEX idx_documents_bm25 
ON documents 
USING bm25(content) WITH (text_config = 'public.log_jiebacfg');

-- 使用 pg textsearch 索引(bm25): 示例1
SELECT id, content, content <@> '自然语言处理' AS bm25_score
FROM documents
ORDER BY content <@> '自然语言处理' 
LIMIT 10;

 id |                 content                  |     bm25_score
----+------------------------------------------+---------------------
  5 | 自然语言处理是人工智能的一个重要领域     | -2.7817881107330322
  8 | 机器学习和深度学习在自然语言处理中的应用 |  -2.383758068084717
(2 行记录)


-- 使用 pg textsearch 索引(bm25): 示例2
SELECT id, content, content <@> 'postgresql 系统' AS bm25_score
FROM documents
where content <@> 'postgresql 系统' < 0
ORDER BY content <@> 'postgresql 系统' 
LIMIT 10;

 id |                 content                  |     bm25_score
----+------------------------------------------+---------------------
  1 | PostgreSQL is a powerful database system | -1.6696925163269043
  4 | PostgreSQL结合结巴分词非常强大           | -1.5651187896728516
(2 行记录)

```