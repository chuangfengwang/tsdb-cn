
# 启动并使用环境

```bash
# 环境变量里配置数据库默认密码
export tsdb_password=xxxxxxxxx

# 启动容器
podman run \
--name tsdb \
-e POSTGRES_PASSWORD=${tsdb_password} \
-p 15435:5432 \
-v $(pwd)/tsdb_data:/home/postgres/pgdata/data:Z,U \
ghcr.io/chuangfengwang/timescale-bm25-cn:pg17-v1.1.2

# 调试容器
podman exec -it tsdb /bin/bash

# 连接数据库
psql -h podman-box.orb.local -p 15435 -U postgres -d postgres

psql -d "postgres://postgres:${echo -n "${tsdb_password}" | jq -sRr @uri}@podman-box.orb.local:15435/postgres"
```

postgresql.conf

```
shared_preload_libraries = 'pg_stat_statements,timescaledb,pg_textsearch,zhparser,pg_jieba'		# (change requires restart)

#------------------------------------------------------------------------------
# PLUGIN MANAGEMENT
#------------------------------------------------------------------------------
# zhparser default config
zhparser.dict_in_memory = true

```

确认插件已经安装

```sql
-- 确认插件生效
SHOW shared_preload_libraries;
```

# 对于指定的库, 启用插件 zhparser

```sql
-- 启用插件 zhparser
CREATE EXTENSION zhparser;

-- 查看 zhparser 的配置参数
SELECT name, setting FROM pg_settings WHERE name LIKE 'zhparser%';

-- 查看 zhparser 的扩展词典
show zhparser.extra_dicts;

-- 词典目录位置 /usr/share/postgresql/17/tsearch_data
ALTER SYSTEM SET zhparser.extra_dicts = 'custom_words.txt';
ALTER SYSTEM RESET zhparser.extra_dicts;
ALTER SYSTEM RESET zhparser.dict_in_memory;
-- 针对某个库增加自定义词典
ALTER DATABASE db_a SET zhparser.extra_dicts = 'medical_dict.txt';

SELECT pg_reload_conf();

-- 第二种写入自定义词汇的方式: 往系统表里插入自定义词, 然后触发函数加载到内存. 不同库之间可能因为dump文件重名问题相互影响
INSERT INTO zhparser.zhprs_custom_word (word, tf, idf, attr) VALUES ('波音737', 15.0, 1.0, 'n');
-- 执行官方自带的同步函数
SELECT sync_zhprs_custom_word();
-- 重建 psql session 才能看到修改效果


-- 创建名为 'chinese_zh' 的检索配置，使用 zhparser 解析器
CREATE TEXT SEARCH CONFIGURATION chinese_zh (PARSER = zhparser);
-- 映射需要的词性 (n:名词, v:动词, a:形容词, i:成语, e:叹词, l:习语等)
ALTER TEXT SEARCH CONFIGURATION chinese_zh ADD MAPPING FOR n,v,a,i,e,l WITH simple;

-- 验证分词效果
SELECT to_tsvector('chinese_zh', '你好啊啊,我爱北京天安门');
-- 验证匹配效果
SELECT to_tsvector('chinese_zh', '你好啊啊,我爱北京天安门') @@ to_tsquery('chinese_zh', '北京 & 天安门');

```

# 对于指定的库, 启用插件 pg_jieba

```sql
-- 启用插件 pg_jieba
CREATE EXTENSION pg_jieba;

-- pg_jieba 默认会创建名为 jiebacfg,jiebaqry 的全文检索配置. jiebacfg 用于碎切, jiebaqry 用于精切
SELECT * FROM ts_debug('jiebacfg', 'PostgreSQL结合结巴分词非常强大');
SELECT * FROM ts_debug('jiebaqry', 'PostgreSQL结合结巴分词非常强大');
```

# 对于指定的库, 联合使用 pg_jieba 和 pg_textsearch
```sql
CREATE EXTENSION IF NOT EXISTS pg_textsearch;
CREATE EXTENSION IF NOT EXISTS pg_jieba;

create database if not EXISTS pg_search_demo;
\c pg_search_demo
       
CREATE TABLE documents (id bigserial PRIMARY KEY, content text);
INSERT INTO documents (content) VALUES
    ('PostgreSQL is a powerful database system'),
    ('BM25 is an effective ranking function'),
    ('Full text search with custom scoring');

-- 创建 bm25 索引
CREATE INDEX idx_documents_bm25 
ON documents 
USING bm25(content) WITH (text_config = 'public.jiebacfg');

-- 创建全文检索索引(pg自带的全文检索索引)
CREATE INDEX idx_docs_gin_fts 
ON documents 
USING gin(to_tsvector('jiebacfg', content));

-- 使用 pg textsearch 索引(bm25): 示例1
SELECT id, content, content <@> '自然语言处理' AS bm25_score
FROM documents
ORDER BY content <@> '自然语言处理' 
LIMIT 10;

-- 使用 pg textsearch 索引(bm25): 示例2
SELECT id, content, content <@> 'database' AS bm25_score
FROM documents
where content <@> 'database' < 0
ORDER BY content <@> 'database' 
LIMIT 10;

-- 使用双重索引
SELECT id, content, content <@> 'database system' AS bm25_score
FROM documents
where to_tsvector('jiebacfg', content) @@ plainto_tsquery('jiebacfg', 'database system')
ORDER BY content <@> 'database system' 
LIMIT 10;
```


# 对于指定的库, 联合使用 pg_jieba 和 pg_textsearch, 自定义词性映射
```sql
-- 创建自定义分词器配置(log_jiebacfg 是自定义的分词器名称)
DROP TEXT SEARCH CONFIGURATION IF EXISTS public.log_jiebacfg;

CREATE TEXT SEARCH CONFIGURATION public.log_jiebacfg (PARSER = jieba);
-- CREATE TEXT SEARCH CONFIGURATION public.log_jiebacfg (COPY = jiebacfg);
       
-- 查看 pg_jieba 的词性
\pset pager off
\dFp+ jieba

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
WITH simple; -- simple jieba_stem

-- ALTER TEXT SEARCH CONFIGURATION public.log_jiebacfg 
--     ALTER MAPPING REPLACE jieba_stem WITH simple;
-- ALTER TEXT SEARCH CONFIGURATION public.log_jiebacfg 
-- DROP MAPPING IF EXISTS FOR x;

-- 查看分词器配置
\dF+ public.log_jiebacfg

-- 验证分词效果
SELECT * FROM ts_debug('public.log_jiebacfg', 'PostgreSQL结合结巴分词非常强大	database system');

```

