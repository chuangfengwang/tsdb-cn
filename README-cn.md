[English](README.md) | 中文

# 介绍

tsdb-cn 是一个基于 TimescaleDB 的中文全文检索数据库镜像, 集成了 pg_textsearch pg_jieba zhparser 插件, 支持 BM25 排序算法, 适合用于中文文本的搜索和分析。

# 启动并连接实例

启动镜像, 这里使用 podman 作为示例, 你也可以使用 docker

```bash
# 拉取镜像
podman pull ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-v1.1.2

# 启动镜像
# 环境变量里配置数据库默认密码
export tsdb_password=your_password_here
# 创建数据目录
mkdir -p tsdb_data

# 启动容器
podman run \
--name tsdb \
-e POSTGRES_PASSWORD=${tsdb_password} \
-p 15435:5432 \
-v $(pwd)/tsdb_data:/home/postgres/pgdata/data:Z,U \
ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-v1.1.2

```

连接数据库

```bash
# 连接数据库
psql -h localhost -p 15435 -U postgres 
```

# 使用 pg_jieba 插件

```pgsql
-- 查看已加载的插件
postgres=# SHOW shared_preload_libraries;
          shared_preload_libraries
---------------------------------------------
 timescaledb,pg_textsearch,zhparser,pg_jieba

-- 启用插件
CREATE EXTENSION IF NOT EXISTS pg_textsearch;
CREATE EXTENSION IF NOT EXISTS pg_jieba;

-- 使用默认分词器配置(public.jiebacfg, public.jiebaqry)
postgres=# SELECT * FROM ts_debug('jiebacfg', 'PostgreSQL结合结巴分词非常强大	database system');
 alias | description |   token    | dictionaries | dictionary |   lexemes
-------+-------------+------------+--------------+------------+--------------
 eng   | letter      | PostgreSQL | {jieba_stem} | jieba_stem | {postgresql}
 v     | verb        | 结合       | {jieba_stem} | jieba_stem | {结合}
 n     | noun        | 结巴       | {jieba_stem} | jieba_stem | {结巴}
 n     | noun        | 分词       | {jieba_stem} | jieba_stem | {分词}
 d     | adverb      | 非常       | {jieba_stem} | jieba_stem | {非常}
 a     | adjective   | 强大       | {jieba_stem} | jieba_stem | {强大}
 x     | unknown     |            | {jieba_stem} | jieba_stem | {"      "}
 eng   | letter      | database   | {jieba_stem} | jieba_stem | {database}
 x     | unknown     |            | {jieba_stem} | jieba_stem | {" "}
 eng   | letter      | system     | {jieba_stem} | jieba_stem | {}
(10 行记录)

postgres=# SELECT * FROM ts_debug('jiebaqry', 'PostgreSQL结合结巴分词非常强大	database system');
 alias | description |   token    | dictionaries | dictionary |   lexemes
-------+-------------+------------+--------------+------------+--------------
 eng   | letter      | PostgreSQL | {jieba_stem} | jieba_stem | {postgresql}
 v     | verb        | 结合       | {jieba_stem} | jieba_stem | {结合}
 n     | noun        | 结巴       | {jieba_stem} | jieba_stem | {结巴}
 n     | noun        | 分词       | {jieba_stem} | jieba_stem | {分词}
 d     | adverb      | 非常       | {jieba_stem} | jieba_stem | {非常}
 a     | adjective   | 强大       | {jieba_stem} | jieba_stem | {强大}
 x     | unknown     |            | {jieba_stem} | jieba_stem | {"      "}
 eng   | letter      | database   | {jieba_stem} | jieba_stem | {database}
 x     | unknown     |            | {jieba_stem} | jieba_stem | {" "}
 eng   | letter      | system     | {jieba_stem} | jieba_stem | {}
(10 行记录)

```

创建自定义分词器配置
```pgsql
-- 创建自定义分词器配置(log_jiebacfg 是自定义搜索配置的名称)
DROP TEXT SEARCH CONFIGURATION IF EXISTS public.log_jiebacfg;
CREATE TEXT SEARCH CONFIGURATION public.log_jiebacfg (PARSER = jieba);

-- 配置词性映射, 排除了 x(未知) 的词性
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
postgres=# SELECT * FROM ts_debug('public.log_jiebacfg', 'PostgreSQL结合结巴分词非常强大	database system');
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

```

# 联合使用 pg_jieba 和 pg_textsearch 插件

```pgsql
create database pg_search_demo;
\c pg_search_demo

CREATE TABLE documents (id bigserial PRIMARY KEY, content text);
INSERT INTO documents (content) VALUES
    ('PostgreSQL is a powerful database system'),
    ('BM25 is an effective ranking function'),
    ('Full text search with custom scoring'),
    ('PostgreSQL结合结巴分词非常强大'),
    ('自然语言处理是人工智能的一个重要领域'),
    ('数据库系统的性能优化是关键'),
    ('文本搜索和信息检索技术的发展'),
    ('机器学习和深度学习在自然语言处理中的应用'),
    ('BM25算法在搜索引擎中的使用'),
    ('自定义分词器配置可以提高搜索效果');
 
CREATE INDEX idx_documents_bm25 
ON documents 
USING bm25(content) WITH (text_config = 'public.log_jiebacfg');

-- 使用 pg textsearch 索引(bm25): 示例1
SELECT id, content, content <@> '自然语言处理' AS bm25_score
FROM documents
ORDER BY content <@> '自然语言处理' 
LIMIT 10;

 id |                 content                  |     bm25_score
----+------------------------------------------+--------------------
  5 | 自然语言处理是人工智能的一个重要领域     | -2.834373950958252
  8 | 机器学习和深度学习在自然语言处理中的应用 | -2.521183490753174
(2 行记录)

-- 使用 pg textsearch 索引(bm25): 示例2
SELECT id, content, content <@> 'postgresql 系统' AS bm25_score
FROM documents
where content <@> 'postgresql 系统' < 0
ORDER BY content <@> 'postgresql 系统' 
LIMIT 10;

 id |                 content                  |     bm25_score
----+------------------------------------------+---------------------
  1 | PostgreSQL is a powerful database system | -1.6182063817977905
  4 | PostgreSQL结合结巴分词非常强大           |  -1.511040449142456
(2 行记录)

```