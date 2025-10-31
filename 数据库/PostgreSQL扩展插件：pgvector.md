# 1. 简介

pgvector 是一个开源的 PostgreSQL 扩展插件，用于在关系型数据库中实现高效的向量存储和相似性搜索。

源码地址：https://github.com/pgvector/pgvector

它的优势在于与 PostgreSQL 生态的继承，从而支持 ACID 事务、表连接等原生特性。

pgvector 为 PostgreSQL 新增 vector 数据类型，支持单精度（vector）、半精度（halfvec）、二进制（bit）、稀疏向量（sparsevec）。

pgvector 内置多种距离度量，包括 L2距离（欧式距离）、内积、余弦距离、L1距离（曼哈顿距离）、汉明距离、Jaccard 距离。

pgvector 的使用场景有：

* RAG（检索增强生成）：为大语言模型（LLM）从知识库中快速检索与问题最相关的文档片段；
* 推荐系统：计算用户向量和内容向量之间的相似度，进行个性化推荐；
* 语义搜索：基于语义搜索相似关键字；
* 图片检索：通过向量化表示文字、图片，支持以图搜图或以文搜图；
* 相似性匹配：识别重复或相似的内容，如新闻去重、抄袭检测；

相比 Faiss 的优势：

* 基于 PostgreSQL 生态，无须加载全部向量到内存中；
* 支持向量的删除；

# 2. 安装

下载源码并安装。

```bash
git clone --branch v0.8.1 https://github.com/pgvector/pgvector.git
cd pgvector
make
make install
```

启用扩展插件。

```postgresql
CREATE EXTENSION IF NOT EXISTS vector;
```

查看插件是否安装成功。

```postgresql
SELECT * FROM pg_available_extention;
SELECT * FROM pg_extention;
```

# 3. 使用

创建一个表，其中 embedding 字段类型为 vector(3)，表示 3 维向量。

```postgresql
CREATE TABLE items (
  id bigserial PRIMARY KEY,
  embedding vector(3)
);
```

插入数据：

```postgresql
INSERT INTO items (embedding) VALUES ('[1,2,3]'), ('[4,5,6]');
```

更新数据：

```postgresql
UPDATE items SET embedding = '[1,2,3]' WHERE id = 1;
```

删除数据：

```postgresql
DELETE FROM items WHERE id = 1;
```

查询数据，计算使用向量距离：

```postgresql
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
SELECT * FROM items WHERE embedding <-> '[3,1,2]' < 5;
```

向量距离计算函数：

* <->：L2距离
* <#>：内积
* <=>：余弦距离
* <+>：L1距离
* <~>：汉明距离
* <%>：Jaccard 距离

计算平均向量：

```postgresql
SELECT AVG(embedding) FROM items;
```

# 4. 索引

默认情况下，pgvector 精确计算最近邻搜索，提供最高的召回率。

可以添加索引来使用近似的最近邻搜索，牺牲一些召回率来换取查询速度，添加索引后查询结果会有所不同。

## 4.1 HNSW

HNSW 索引基于分层图结构，查询性能和召回率非常出色，但构建索引速度较慢，且占用内存更多。

可以在表中无数据时创建索引，不需要训练步骤。

创建 L2距离索引：

```postgresql
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops);
```

指定参数，m 表示每层的最大连接数，ef_construction 表示构建图的动态候选列表大小：

```postgresql
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops) WITH (m = 16, ef_construction = 64);
```

## 4.2 IVFFLAT

IVFFLAT 索引通过聚类机制将向量划分到多个列表中进行搜索，构建速度更快，内存占用更小，但查询性能与召回率的权衡通常不如 HNSW。

建议在表有数据后再创建索引。

创建 L2距离索引：

```postgresql
CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops) WITH (lists = 100);
```

参数 list 建议为表数据量 / 1000。

