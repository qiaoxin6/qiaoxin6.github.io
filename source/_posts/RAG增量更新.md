---
title: RAG 增量更新
date: 2026-09-01 15:25:00
tags:
  - RAG
  - 面试
categories:
  - AI
---

**在面试中除了回答基础的rag的整体流程外还有关于rag的许多优化或使用问题。 其中更新就是最重要的一个点，在实际使用中rag中的知识不可能是不变的，……**

**核心原则：以文档id作为主键做原子替换，chuck永远是文档的派生数据。** **永远不要只更新变化的chuck ，那是万恶之源。**  

------



## 1.数模型数设计



```sql
-- 文档表：唯一真相来源
CREATE TABLE documents (
  doc_id      VARCHAR PRIMARY KEY,   -- 文档唯一ID（如 飞书file_token / URL hash）
  title       VARCHAR,
  source      VARCHAR,               -- 来源：feishu/confluence/upload
  source_url  VARCHAR,
  version     INT,                   -- 版本号，每次修改 +1
  status      VARCHAR,               -- active / deleted
  content_hash VARCHAR,              -- 内容hash，判断是否真的变了
  updated_at  TIMESTAMP,
  created_at  TIMESTAMP
);

-- chunk 表：记录文档与chunk的映射
CREATE TABLE chunks (
  chunk_id    VARCHAR PRIMARY KEY,   -- 通常用 doc_id + chunk_index 组合
  doc_id      VARCHAR REFERENCES documents(doc_id),
  chunk_index INT,                   -- 第几个chunk
  parent_id   VARCHAR,               -- 父块ID（父子分块时）
  content     TEXT,                  -- chunk原文
  content_prefix TEXT,               -- contextual retrieval 生成的前缀
  chunk_type  VARCHAR,               -- text/table/code/image
  section_path VARCHAR,              -- 标题路径
  page_num    INT,
  embedding_model VARCHAR,           -- 用哪个模型生成的向量
  embedding_dim INT,
  created_at  TIMESTAMP
);
```





```sql
{
  "chunk_id": "doc_123_chunk_5",
  "doc_id": "doc_123",
  "version": 7,
  "content": "用户收到商品后7个工作日内...",
  "vector": [0.012, -0.034, ...],
  "metadata": {
    "title": "退款政策",
    "section_path": "第三章 > 退款流程",
    "page_num": 12,
    "source_url": "...",
    "updated_at": "2026-08-20T10:30:00Z",
    "access_level": 2
  }
}
```

关键点：每个chuck都要有doc_id和version。 doc和chuck都要有content_hash



## 2.新增文档

走正常流程添加即可。

```sql
监听到新文档（Webhook/定时扫描/手动上传）
    ↓
① 计算 content_hash，检查是否已存在（防重复）
    ↓
② 文档解析（PDF→文本/表格/图片）
    ↓
③ 切分（递归/结构/父子分块）
    ↓
④ 增强（Contextual Retrieval 前缀、关键词、摘要）
    ↓
⑤ Embedding 向量化
    ↓
⑥ 写入 chunks 表 + 向量数据库
    ↓
⑦ 写入 documents 表，status=active, version=1
```



## 3.修改文档

修改推荐做文档级替换，不做chuck级替换

```sql
监听到文档变更（Webhook/定时扫描）
    ↓
① 拉取最新内容，计算新 content_hash
    ↓
② 与数据库中的旧 hash 比较
   ├─ 相同 → 跳过，文档未实际变化
   └─ 不同 → 继续
    ↓
③ 删除旧版本所有 chunk（按 doc_id）
   - DELETE FROM chunks WHERE doc_id = ?
   - 向量库 DELETE WHERE doc_id = ?（带 version 过滤更安全）
    ↓
④ version += 1
    ↓
⑤ 重新走完整管道：解析 → 切分 → 增强 → embedding → 写入
    ↓
⑥ 更新 documents 表：新 content_hash、新 version、新 updated_at
```

**为什么不做 chunk 级 diff（只更新变化的 chunk）？**

- 一个段落的插入 / 删除会导致后续所有 chunk_index 偏移，维护成本高
- 切分结果受上下文影响（语义切分、父子分块），局部变化可能导致整个文档的切分边界都变了
- Contextual Retrieval 前缀依赖全文，任何一处改动理论上所有 chunk 的前缀都应重新生成
- chunk 级 diff 的复杂度远大于重新处理整篇文档的成本

**例外**：如果文档极大（如几百页的手册），且变更频率高，可以按章节 /section 为单位做替换 —— 把 section 当作 "子文档" 管理，每个 section 独立做 delete-then-insert。

## 4.删除文档

```sql
监听到文档删除
    ↓
① 标记 documents.status = 'deleted'（软删除，留审计记录）
    ↓
② 向量库按 doc_id 删除所有 chunk
   DELETE WHERE doc_id = ?
    ↓
③ chunks 表删除或归档
    ↓
④ 如果有父子块/关联索引，级联清理
```

删除有两种策略

| 策略       | 做法                                        | 适用                           |
| ---------- | ------------------------------------------- | ------------------------------ |
| **硬删除** | 直接从向量库和数据库删除                    | 合规要求（被遗忘权）、敏感数据 |
| **软删除** | 标记 status=deleted，向量库保留但加过滤条件 | 需要审计、可能恢复             |

## 5. 级联更新问题  

 文档变更时，以下关联数据需要一起处理：  

| **关联数据**    | **是否需要更新**     | **原因**                            |
| --------------- | -------------------- | ----------------------------------- |
| 子 chunk        | 是，随文档整体重建   | 切分边界可能变化                    |
| 父 chunk        | 是，随文档整体重建   | 父子映射关系可能变化                |
| Contextual 前缀 | 是，全部重新生成     | 前缀依赖全文上下文                  |
| 摘要 / 假设问题 | 是，重新生成         | 内容变了                            |
| 关键词 / 实体   | 是，重新提取         | 内容变了                            |
| BM25 索引       | 是，删除旧的写入新的 | 词项变了                            |
| 文档级摘要      | 是，重新生成         | 全局内容变了                        |
| 知识图谱关系    | 视情况               | GraphRAG 场景下需要更新关联实体和边 |

## 6.一致性保障

1. 事务性
   文档替换过程中如果失败，不能出现 "旧 chunk 删了、新 chunk 没写完" 的中间状态：

```sql
方案A：双 collection + 别名切换
  - 写入新 collection → 成功后原子切换别名指向新 collection
  - 类似蓝绿部署

方案B：版本号标记
  - 新 chunk 写入时带 version=new
  - 全部写入成功后，更新文档的 current_version
  - 检索过滤 version = current_version
  - 异步清理旧 version 的 chunk
```

1. 定期对账
   每天 / 每周跑一致性校验：

```sql
# 检查1：源库有但向量库没有（漏索引）
SELECT doc_id FROM documents
WHERE status='active' AND doc_id NOT IN (SELECT DISTINCT doc_id FROM chunks)

# 检查2：向量库有但源库没有（脏数据）
SELECT DISTINCT doc_id FROM chunks
WHERE doc_id NOT IN (SELECT doc_id FROM documents WHERE status='active')

# 检查3：chunk 数量异常（可能部分写入失败）
SELECT doc_id, COUNT(*) FROM chunks GROUP BY doc_id
HAVING COUNT(*) < 2  -- 文档只有1个或0个chunk，可能异常
```

1.  变更监听方式  

| 方式     | 适用场景                                  | 特点                                           |
| -------- | ----------------------------------------- | ---------------------------------------------- |
| Webhook  | 飞书 / Confluence/GitHub 等支持回调的系统 | 实时性好，但需要处理回调丢失                   |
| 定时轮询 | 所有场景                                  | 简单可靠，延迟取决于轮询间隔（通常 5–60 分钟） |
| 消息队列 | 高并发 / 多源接入                         | 解耦、可重试、有顺序保证                       |
| 手动触发 | 低频更新的知识库                          | 最简单                                         |





## 7. 完整的增量管道伪代码  

```python
def process_document_change(doc_id):
    # 1. 拉取最新文档
    doc = fetch_document(doc_id)
    new_hash = hash(doc.content)

    # 2. 检查是否真的变化
    old = db.get_document(doc_id)
    if old and old.content_hash == new_hash:
        return  # 内容没变，跳过

    # 3. 解析 + 切分 + 增强 + embedding
    elements = parse_document(doc)           # 解析
    chunks = chunk_document(elements)         # 切分
    chunks = enrich_chunks(chunks, doc)       # contextual前缀、关键词等
    chunks = embed_chunks(chunks)             # 向量化

    # 4. 原子替换
    new_version = (old.version if old else 0) + 1
    try:
        # 写入新 chunk（带新版本号）
        for chunk in chunks:
            chunk.doc_id = doc_id
            chunk.version = new_version
            vector_store.upsert(chunk)
            db.insert_chunk(chunk)

        # 删除旧版本 chunk
        vector_store.delete(filter={
            "doc_id": doc_id,
            "version": {"$lt": new_version}
        })
        db.delete_chunks(doc_id, version_lt=new_version)

        # 更新文档记录
        db.upsert_document(doc_id, hash=new_hash, version=new_version)

    except Exception as e:
        # 失败回滚：清理本次写入的半成品
        vector_store.delete(filter={"doc_id": doc_id, "version": new_version})
        db.delete_chunks(doc_id, version=new_version)
        raise e


def process_document_delete(doc_id):
    # 软删除文档记录
    db.mark_deleted(doc_id)
    # 硬删除向量（或保留并加 deleted 标记）
    vector_store.delete(filter={"doc_id": doc_id})
    db.delete_chunks(doc_id)
```

## 8. 常见坑  

| 坑                                    | 后果                             | 解法                                   |
| ------------------------------------- | -------------------------------- | -------------------------------------- |
| 只更新 chunk 内容不更新向量           | 文本和向量不一致，检索到旧内容   | 文本和向量必须同生共死，永远一起替换   |
| 删除文档只删数据库不删向量库          | "删了还能搜到"，合规风险         | 删除操作必须覆盖所有存储               |
| 没有 doc_id，用 chunk 内容 hash 当 ID | 内容稍变就无法关联旧 chunk       | 用稳定的文档 ID，不要用内容生成 ID     |
| Webhook 丢了没兜底                    | 文档更新了但索引没更新           | 定时轮询对账兜底                       |
| 并发处理同一文档                      | 新旧版本交错写入，结果不确定     | 按 doc_id 加分布式锁，同一文档串行处理 |
| 切分器版本升级                        | 同一文档切出不同 chunk，新旧混杂 | 记录 chunker 版本，升级时全量重建      |
| 父子块只更新子块不更新父块            | 父块内容和子块对不上             | 父子块随文档整体重建                   |



## 关键出装： 更新时以文档为最小单位，不能只更新chuck。 数据库中要有 doc_id 、 version 、chuck_id 等字段标识