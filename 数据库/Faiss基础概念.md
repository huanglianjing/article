# 1. 简介

Faiss 全称 Facebook AI Similarity Search，是 Facebook 开发的高维向量相似性搜索和聚类的库，支持的语言有 C++、Python 等。

源码地址：https://github.com/facebookresearch/faiss

Faiss 支持稠密向量（Dense Vector），支持多种索引算法、距离度量。

应用场景：

* LLM 知识库问答；
* 图像检索：通过图片提取特征向量，进行相似图片检索；
* 语义搜索：通过 NLP 模型将文本转换为向量，在 Faiss 检索语义相似的句子或文档；
* 人脸识别和聚类：比对提取的人脸特征，进行相似人脸的聚类；

不足之处：

* 将索引结构和向量数据完全加载到内存，数据量较大时会消耗大量内存；
* 不支持删除单个向量；

# 2. 索引

Faiss 支持多种索引结构：

* Flat Index（扁平索引）：如 IndexIVFFlat 和 IndexFlatIP，执行暴力搜索，计算查询向量和每个向量的精确距离，精度最高，但速度最慢，适用于小规模数据机；
* IVF Index（倒排文件索引）：如 IndexIVFFlat，先通过 k-means 聚类将向量划分到多个单元中，搜索时在最近 nprobe 个单元内进行惊喜搜索，极大减少了计算量；
* PQ Index（乘积量化索引）：如 IndexPQ 和 IndexIVFPQ，将高维向量切分为多个子向量分别进行量化编码，压缩向量的占用空间，适合超大规模数据机和内存首先场景，精度会有一定的损失；
* HNSW Index（分层索引）：如 IndexHNSWFlat，基于图算法，构建多层网络结构实现快速近邻搜索，查询速度快精度高，但构建索引慢且内存占用较大；

# 3. 相似度度量

Faiss 支持多种相似度度量：

* L2距离（欧式距离）
* 内积
* 余弦相似度
* L1距离（曼哈顿距离）

