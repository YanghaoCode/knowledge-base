# 美团地址RAG Agent技术方案设计

> **Workspace:** /Users/yanghao/Documents/尚硅谷-AI/05-项目_掌柜智库/day10_查询模块_out输出以及项目总结/代码

---

**User**

我是美团App通用地址选择页的前端研发，我们支持把这些通用的地址页面提供给美团的各个业务方使用，他们在接入页面的过程中可能会有很多问题，为了节约和业务对接的时间，我想开发一个rag agent，支持各个业务方通过agent来对话解决他们的问题。地址通用页面由很多页面组成，比如地址选择页、城市选择页、收货地址选择页、海外城市选择页等等，我们沉淀了页面接入文档、FAQ文档等。帮我设计一个技术方案，包括agent的上传文档链路和检索链路，如果有疑问可以先问我再设计方案

**Assistant**

我先快速了解一下你当前在做的项目（看技术栈是否可以复用到新方案里），然后再向你提问。

我已经了解你当前项目（掌柜智库）的架构：import_process（文档导入：PDF→MD→切分→BGE向量化→Milvus）+ query_process（查询：embedding/HyDE检索→rerank→RRF→生成回答，支持SSE流式）。这套模式可以参考到新方案里。

在设计美团地址页 RAG Agent 方案前，我先问你几个关键问题，这些会直接影响架构选型：

**1. 模型与部署环境（最关键）**
- 美团内部是否有可用的 LLM 服务（内部部署的开源模型或公司统一采购的 API）？合规上是否允许调用外部大模型 API（GLM / DeepSeek / OpenAI 等）？
- Embedding / Rerank 模型是否有内部服务可用？

**2. 文档情况**
- 文档目前存放在哪里？语雀 / Confluence / 内部 Wiki / Git 仓库？格式是 Markdown、Word 还是 PDF？
- 数量级大概多少（几十篇 / 几百篇）？更新频率如何（每周都有改动，还是相对稳定）？
- 文档更新后希望自动同步进知识库，还是手动重新上传即可？

**3. 交互形式**
- 业务方通过什么方式使用：Web 聊天页面、大象（内部 IM）机器人、还是提供 API 让业务方自己接入？
- 需要多轮对话（支持追问、上下文继承）吗？

**4. 回答质量与兜底**
- 回答是否需要标注引用来源（哪篇文档的哪个章节），方便业务方自己去看原文？
- 问题在文档里找不到答案时，期望怎么处理：直接拒答、转人工答疑（比如拉群里的人工客服）、还是允许 LLM 自由发挥？
- 是否希望统计“没被解决的问题”，用来反哺补充 FAQ 文档？

**5. 权限与规模**
- 文档内容是否有权限分级（比如海外城市页文档只有特定业务可见）？
- 预期使用规模：接入的业务方数量、日均提问量级（几十 / 几千）？

你可以按编号简单回答，不确定的我可以给出默认建议。回答后我会给出完整的技术方案（包括上传链路和检索链路的详细设计）。

---

**User**

1、LLM和Embedding / Rerank 模型都是调用外部模型
2、文档可以是一个md文件、几十篇、文档更新后手动重新上传即可
3、业务方通过什么方式使用：Web 聊天页面，需要多轮对话
4、回答不需要标注引用来源；问题在文档里找不到答案时，期望转人工答疑（直接调用一个现成接口就可以）；希望统计“没被解决的问题”，用来反哺补充 FAQ 文档
5、文档内容无权限分级；预期使用规模几十/日


**Assistant**

需求已经清楚了，规模不大（几十篇 MD、日均几十次提问），方案可以做得简单可靠，不需要过度设计。以下是完整技术方案。

# 美团地址页接入 RAG Agent 技术方案

## 一、总体架构

```
                    ┌─────────────────────────────────────────┐
                    │              Web 聊天页面                │
                    │   (React/Vue + SSE 流式渲染 + 👍👎反馈)  │
                    └──────────────┬──────────────────────────┘
                                   │ HTTP / SSE
┌──────────────┐          ┌────────▼─────────┐
│ 运营/你自己    │──上传MD──▶│   FastAPI 服务    │
│ (手动重新上传) │          │                  │
└──────────────┘          │  ┌────────────┐  │   ┌─────────┐
                          │  │ 导入Agent   │──┼──▶│ Milvus  │
                          │  │ (LangGraph)│  │   │ 向量库   │
                          │  └────────────┘  │   └────▲────┘
                          │  ┌────────────┐  │        │
                          │  │ 查询Agent   │──┼────────┘
                          │  │ (LangGraph)│  │
                          │  └─────┬──────┘  │   ┌─────────┐
                          └────────┼─────────┘   │ MongoDB │
                                   │             │ 历史/统计 │
                     ┌─────────────┼──────────┐  └─────────┘
                     ▼             ▼          ▼
              外部 LLM API   外部Embedding   人工答疑接口
              (生成/改写)    /Rerank API     (兜底转接)
```

**技术选型**（直接复用你掌柜智库项目的骨架）：

| 组件 | 选型 | 说明 |
|---|---|---|
| Agent 编排 | LangGraph | 导入链路、查询链路各一个 graph |
| 服务框架 | FastAPI + SSE | 参考你现有的 query_server / import_server |
| 向量库 | Milvus（单机 standalone） | 数据量小，单实例足够 |
| 会话/统计存储 | MongoDB | 会话历史、导入任务状态、未解决问题 |
| LLM | 外部 API（GLM-4 / DeepSeek 等，支持流式） | 改写、生成、判兜底 |
| Embedding | 外部 API（BGE 系列） | 批量向量化 |
| Rerank | 外部 API（BGE-reranker） | 精排 |

规模小（几十 QPS 都不到），**单服务单实例部署即可，不需要缓存、多副本、限流这些**。

---

## 二、上传文档链路（导入 Agent）

几十篇 MD 文档，**不需要 PDF 转换、OCR、图片处理**，链路比掌柜智库简单很多。

### Graph 结构

```
START → node_md_parse → node_md_split → node_embedding → node_import_milvus → END
```

**1. node_md_parse（解析）**
- 校验文件是 `.md`，读取原文
- 记录导入任务到 Mongo：`{task_id, doc_name, status: running/success/failed, chunk_count, timestamp}`
- **版本处理（关键）**：按 `doc_name` 删除 Milvus 中该文档旧的所有 chunk（`delete(expr='doc_name == "xxx"')`），实现“重新上传 = 整体替换”，避免新旧版本共存导致检索到过期内容

**2. node_md_split（切分）—— 对检索质量影响最大的一环**

按 Markdown 标题层级切分，因为接入文档天然是树状结构：

- 以 `##` / `###` 标题为边界切出 section，每个 section 携带**章节路径元数据**（如 `"接入指南/海外城市页/唤起事件"`）
- FAQ 文档：每个 `## Q` 天然就是一个 chunk，完美匹配
- 超长 section：按段落二次切分，chunk 间保留 ~50 token overlap
- 超短 section：与相邻兄弟 section 合并，避免语义碎片
- **代码块保护**：接入文档里有大量 JS 调用示例，一个代码块必须在同一个 chunk 内，绝不能从中间切开

目标 chunk 大小：300–500 token（BGE 对这个长度召回效果最好）。

**3. node_embedding（向量化）**
- 批量调外部 Embedding API（注意批大小 + 简单重试）
- 建议向量化内容 = `章节路径 + 正文`（如 `"【接入指南/唤起事件】\n调用方式：..."`），让标题语义参与召回，用户问“怎么唤起”时命中率明显提升

**4. node_import_milvus（入库）**

Milvus Collection Schema：

```python
schema = milvus_client.create_schema()
schema.add_field("pk_id", DataType.VARCHAR, is_primary=True, max_length=64)   # chunk唯一ID
schema.add_field("doc_name", DataType.VARCHAR, max_length=256)   # 文档名（版本管理/删除的key）
schema.add_field("section_path", DataType.VARCHAR, max_length=512)  # 章节路径
schema.add_field("content", DataType.VARCHAR, max_length=8192)   # chunk原文
schema.add_field("dense_vector", DataType.FLOAT_VECTOR, dim=1024)  # BGE向量
# 可选：sparse_vector（BM25），用于混合检索
```

导入 API：
- `POST /api/import/document`（multipart 上传 MD）→ 返回 task_id，异步跑 graph
- `GET /api/import/status/{task_id}` → 前端/命令行查看进度（你不需要做上传管理页面，用 curl / 简单脚本上传即可）

---

## 三、检索链路（查询 Agent）

### Graph 结构

```
START → node_query_rewrite → node_retrieve → node_rerank → node_answer_check
                                    │                            │
                                    │                     能答 ────▶ node_answer_stream ──▶ END
                                    │                     不能答 ───▶ node_fallback_human ──▶ END
```

**1. node_query_rewrite（多轮对话改写）**

多轮对话的核心。业务方提问经常是指代式的：

> 用户：“海外城市页怎么接入？”
> Agent：回答了接入方式
> 用户：“那它的回调事件呢？” ← 单看这句无法检索

做法：从 Mongo 取当前会话**最近 3–5 轮**历史，LLM 一次调用把新问题改写成独立完整的查询（“海外城市页的回调事件有哪些？”），同时输出改写后的问题。改写失败时降级用原始 query。

会话历史存储（Mongo）：

```json
{
  "conversation_id": "xxx",
  "user_id": "业务方标识",
  "messages": [
    {"role": "user", "content": "...", "ts": 1693700000},
    {"role": "assistant", "content": "...", "message_id": "m1", "feedback": null}
  ]
}
```

**2. node_retrieve（混合检索）**

强烈建议 dense + BM25 混合检索（你项目里的 RRF 模式直接照搬），原因是接入类问题里有大量**精确关键词**（页面名、方法名、参数名、错误码），纯向量检索容易丢：

```
query → ① BM25 检索 top10 ─┐
                           ├→ RRF 融合 → top10 候选
      → ② 向量检索 top10 ──┘
```

BM25 可以用 Milvus 2.4+ 的 sparse vector（BGE-M3 自带 sparse 输出，不用额外维护一个 ES）。纯向量方案也行，但实测接入类文档混合检索召回明显更好，而你的项目代码现成。

**3. node_rerank（精排）**
- 调 BGE-reranker API，RRF 候选 → 重排取 top 3–5
- 数据量小，这步成本可忽略

**4. node_answer_check（兜底判断）—— 方案的关键决策点**

你要求“找不到答案转人工”，这里有两种实现：

- ❌ **边生成边判断**：流式输出一半发现答不了再撤回，体验差
- ✅ **推荐：先判断再生成**。一个小 prompt 把 top chunks + 问题给 LLM，只输出 `CAN_ANSWER / CANNOT_ANSWER`（一次很便宜的调用）。能答 → 进入流式生成；不能答 → 走兜底，用户端不会看到答一半翻车

判断 prompt 要点：明确“只有当文档内容能直接支撑回答时才输出 CAN_ANSWER，禁止靠模型自身知识编造接入方式”（接入参数答错比不答更糟糕）。

**5. node_answer_stream（流式生成）**
- LLM 流式生成，SSE 逐 token 推给前端（复用你的 sse_utils）
- System prompt 约束：仅基于提供的文档内容回答；文档没提到的就说不知道
- 按你的要求不输出引用来源，但 chunk 的 `section_path` 可以内部留存，用于排查问题

**6. node_fallback_human（人工兜底）**
- 调你说的现成接口，带上：原始问题、改写后问题、conversation_id、用户标识
- 前端收到 fallback 事件，展示“已转人工同学跟进，请留意大象消息”之类的固定话术
- **同时写入未解决问题表**（见下节）

查询 API：
- `POST /api/chat`（SSE）：`{conversation_id, question}` → 事件流：`step`（改写/检索进度，可选展示）→ `answer`（逐 token）→ `fallback` / `done`
- 会话 ID 由前端首次提问时生成（UUID），后续携带

---

## 四、未解决问题统计 & FAQ 反哺闭环

Mongo 新增 collection `unresolved_questions`：

```json
{
  "question": "原始提问",
  "rewritten_query": "改写后的query",
  "conversation_id": "xxx",
  "user_id": "xxx",
  "retrieved_chunks": ["命中的chunk_id..."],   // 排查为什么没答上
  "reason": "judge_cannot_answer | user_thumb_down",
  "status": "pending | resolved",
  "created_at": 1693700000
}
```

**两个采集入口**：
1. 自动：node_fallback_human 触发时落库（`judge_cannot_answer`）
2. 手动：每条回答带 👍👎 按钮，👎 落库（`user_thumb_down`）——有些问题 judge 判断“能答”但答非所问，只能靠用户反馈捕获

**反哺流程**（人工，简单就好）：
每周导出 pending 记录（`GET /api/admin/unresolved`，或直接查 Mongo）→ 高频问题写进 FAQ 文档 → 手动重新上传该 MD → 闭环。

---

## 五、工程结构

直接照搬你掌柜智库的目录结构，改动点已经标注：

```
app/
├── conf/                    # lm_config / embedding_config / reranker_config（指向外部API）
├── clients/                 # milvus_utils / mongo_utils（去掉 neo4j、minio 可选保留存原始MD备份）
├── lm/                      # lm_utils / embedding_utils / reranker_utils
├── import_process/
│   ├── agent/
│   │   ├── nodes/
│   │   │   ├── node_md_parse.py        # 新：MD校验+版本删除
│   │   │   ├── node_md_split.py        # 新：标题层级切分+代码块保护
│   │   │   ├── node_embedding.py       # 参考 node_bge_embedding.py
│   │   │   └── node_import_milvus.py   # 参考现有实现
│   │   ├── main_graph.py
│   │   └── state.py
│   └── api/import_server.py
├── query_process/
│   ├── agent/nodes/
│   │   ├── node_query_rewrite.py       # 新：多轮改写
│   │   ├── node_search.py              # 参考 node_search_embedding + RRF 混合
│   │   ├── node_rerank.py              # 参考现有
│   │   ├── node_answer_check.py        # 新：CAN/CANNOT 判断
│   │   ├── node_answer_stream.py       # 参考 node_answer_output
│   │   └── node_fallback_human.py      # 新：调人工接口+落库
│   ├── api/query_server.py             # SSE 接口
│   └── agent/{main_graph.py, state.py}
├── utils/                   # sse_utils / escape_milvus_string_utils 直接复用
└── core/                    # logger / load_prompt 复用
```

相对掌柜智库**砍掉**的部分：PDF→MD 转换、图片处理（node_md_img）、商品名称识别路由（文档量小，全量检索 + rerank 足够）、Mongo 历史 neo4j 全部不需要。

---

## 六、分期建议

**P0（MVP，先跑通）**：
- 导入链路：标题切分 + 纯向量检索 + rerank
- 查询链路：改写 + 检索 + rerank + answer_check + 人工兜底 + 未解决落库
- Web 聊天页 + SSE

**P1（上线后按需加）**：
- BM25 混合检索 + RRF（如果发现纯向量召回不够准）
- 👍👎 反馈按钮、未解决问题管理查询页
- 意图路由（识别问题属于哪个页面，Milvus 按 `doc_name` 过滤，参考你的 node_item_name_recognition——等文档涨到上百篇再做）

几个我替你做的默认决策，如不同意可以调：
1. **judge 先行再流式生成**（避免答一半翻车），而不是生成后检测
2. **重新上传 = 按 doc_name 整体替换**（不做增量 diff，几十篇文档不值得）
3. **原始 MD 备份到 MinIO**（可选，方便追溯历史版本，不想要可以去掉）
4. 会话历史取**最近 3–5 轮**做改写（太长会稀释重点且增加成本）

方案整体就是你掌柜智库架构的一个“减配特化版”，大部分节点代码可以直接改着用。需要我先搭出项目骨架（目录 + 两个 main_graph + 各 node 的空实现），还是你想先对某部分细化？

---

**User**

Milvus存储的数据结构是什么样的

**Assistant**

我先看下你现有项目里 Milvus 的 schema 写法，对着讲更直观。

看了你现有的写法，新方案基本沿用同一套模式（dense + sparse 混合 + HNSW/倒排索引），只是字段按文档 chunk 的场景重新设计。

## Milvus Collection 设计

### 概念结构（一张表：`address_doc_chunk`）

每行 = 一个文档切分后的 chunk：

| 字段名 | 类型 | 说明 | 示例值 |
|---|---|---|---|
| `pk` | INT64, 主键, 自增 | 唯一标识 | 4521 |
| `doc_name` | VARCHAR(256) | 文档名，**重新上传时按它整体删除旧数据** | `"海外城市选择页接入指南"` |
| `section_path` | VARCHAR(512) | 章节路径，检索出的 chunk 归属哪一章 | `"接入指南/海外城市页/唤起事件"` |
| `content` | VARCHAR(8192) | chunk 原文（含代码块） | `"调用方式：window.MT...."` |
| `chunk_id` | VARCHAR(64) | chunk 业务ID，用于反馈溯源 | `"海外城市页接入指南::3"` |
| `dense_vector` | FLOAT_VECTOR(1024) | BGE 稠密向量，语义召回 | `[0.12, -0.05, ...]` |
| `sparse_vector` | SPARSE_FLOAT_VECTOR | BGE-M3 自带稀疏向量，关键词(BM25类)召回 | `{523: 0.8, 1024: 0.3}` |

几十篇文档切完大约几千个 chunk，**单 collection 足够，不需要 partition**（partition 是给百万级/权限隔离场景用的）。

### 建表代码（照你现有风格写，可直接参考改造）

```python
schema = milvus_client.create_schema(auto_id=True, enable_dynamic_field=True)

schema.add_field("pk", datatype=DataType.INT64, is_primary=True, auto_id=True)
schema.add_field("doc_name", datatype=DataType.VARCHAR, max_length=256)
schema.add_field("section_path", datatype=DataType.VARCHAR, max_length=512)
schema.add_field("content", datatype=DataType.VARCHAR, max_length=8192)
schema.add_field("chunk_id", datatype=DataType.VARCHAR, max_length=64)
schema.add_field("dense_vector", datatype=DataType.FLOAT_VECTOR, dim=1024)
schema.add_field("sparse_vector", datatype=DataType.SPARSE_FLOAT_VECTOR)

index_params = milvus_client.prepare_index_params()
# 稠密向量：HNSW索引，余弦相似度
index_params.add_index(
    field_name="dense_vector",
    index_type="HNSW",
    index_name="dense_vector_index",
    metric_type="COSINE",
    params={"M": 16, "efConstruction": 200},  # 几千条数据，小参数即可
)
# 稀疏向量：倒排索引，内积
index_params.add_index(
    field_name="sparse_vector",
    index_type="SPARSE_INVERTED_INDEX",
    index_name="sparse_vector_index",
    metric_type="IP",
    params={"inverted_index_algo": "DAAT_MAXSCORE"},
)
milvus_client.create_collection(
    collection_name="address_doc_chunk",
    schema=schema,
    index_params=index_params,
)
```

和你 [node_item_name_recognition.py](file:///Users/yanghao/Documents/尚硅谷-AI/05-项目_掌柜智库/day10_查询模块_out输出以及项目总结/代码/app/import_process/agent/nodes/node_item_name_recognition.py#L186-L249) 的写法几乎一致，参数说明：

- **`M=16, efConstruction=200`**：你代码注释里的调参表（10万条才需要 M=64）对应几千条 chunk 的规模，用最小档就够，建索引快、内存省
- **`max_length=8192` 的 content**：MD 切分目标 300–500 token，8192 字符留了余量，防止个别长代码块插入报错
- **`enable_dynamic_field=True`**：保留扩展性，以后想加 `doc_version`、`update_time` 之类字段不用改 schema

### 检索时怎么查

查询链路一次请求里对同一张表发两路搜索（[node_rrf.py](file:///Users/yanghao/Documents/尚硅谷-AI/05-项目_掌柜智库/day10_查询模块_out输出以及项目总结/代码/app/query_process/agent/nodes/node_rrf.py) 那套 RRF 之前的双路召回）：

```python
# 向量检索（语义）
results_dense = milvus_client.search(
    collection_name="address_doc_chunk",
    data=[query_dense_vector],
    anns_field="dense_vector",
    limit=10,
    output_fields=["doc_name", "section_path", "content", "chunk_id"],
)
# 稀疏检索（关键词，页面名/方法名/错误码精准匹配）
results_sparse = milvus_client.search(
    collection_name="address_doc_chunk",
    data=[query_sparse_vector],
    anns_field="sparse_vector",
    limit=10,
    output_fields=["doc_name", "section_path", "content", "chunk_id"],
)
```

两路结果 RRF 融合 → rerank → 取 top 3–5 送入 LLM。

### 版本替换（重新上传）

和你的 `step_6_save_to_vector_db` 里"先 delete 再 insert"的模式完全一样，只是删除键从 `item_name` 换成 `doc_name`：

```python
# 上传《海外城市选择页接入指南》新版本时
milvus_client.delete(
    collection_name="address_doc_chunk",
    filter=f"doc_name == '{escape_milvus_string(doc_name)}'",  # 复用你的转义工具
)
milvus_client.insert(collection_name="address_doc_chunk", data=chunks)
```

### 注意一个和你现有项目的差异

你的 `file_title`、`item_name` 用了 `max_length=65535`，这个值超过了 Milvus VARCHAR 上限（65535 是字节上限但会显著浪费内存）。新方案里 `content` 压到 8192、其他元数据字段更小——几千行的小表，字段宽度收敛一点更省内存也更规范。

另外查询侧检索时只需要 `output_fields` 里带 `content/section_path/chunk_id`，不需要返回向量本身，减少网络传输。

---

**User**

有不同类型的文档，比如地址页接入文档、历史问题Q&A文档等等，这些不同类型的文档的数据都保存在一个集合里吗

**Assistant**

**是的，建议都放一个 collection，加一个 `doc_type` 标量字段区分**，不按类型拆多张表。

## 为什么单集合更合适

你的检索场景决定了这一点：**用户提问时你不知道答案在哪类文档里**。

> 业务方问：“唤起海外城市页没反应怎么办？”

这句话可能命中的是：
- 接入文档里“唤起事件”章节（`doc_type='access_doc'`）
- 历史 Q&A 文档里某条“唤起失败排查”（`doc_type='faq'`）

单表检索 → 统一 rerank → 谁相关谁排前面，一条链路搞定。如果拆成多个 collection，就得每张表各查一路再做跨表融合，链路复杂度翻倍，而收益在你这个规模（几千 chunk）下为零。

## Schema 只需要加一个字段

```python
schema.add_field("doc_name", datatype=DataType.VARCHAR, max_length=256)
schema.add_field("doc_type", datatype=DataType.VARCHAR, max_length=64)   # 新增：access_doc / faq_doc / ...
schema.add_field("section_path", datatype=DataType.VARCHAR, max_length=512)
schema.add_field("content", datatype=DataType.VARCHAR, max_length=8192)
...
```

`doc_type` 是普通标量字段，两个用途：

**1. 需要时做标量过滤检索**（Milvus 原生支持，且能利用字段索引，性能无损）：

```python
# 场景：用户问题明显是报错排查类，只想搜 FAQ
results = milvus_client.search(
    collection_name="address_doc_chunk",
    data=[query_dense_vector],
    anns_field="dense_vector",
    filter="doc_type == 'faq_doc'",
    limit=10,
    ...
)
```

默认不传 filter 就是全表混合搜。以后如果做了意图路由（判断问题属于接入咨询还是故障排查），再按类型过滤，这正是你现有项目 `node_item_name_recognition` 的思路——数据先放一起，路由是查询侧的优化，两者不冲突。

**2. 运维排查**：按类型统计 chunk 数量、看哪类文档覆盖率低。

删除/版本替换逻辑不变，仍然按 `doc_name` 删（一个文档只属于一个类型，类型只是元数据）。

## 不同类型文档的差异放在导入侧处理，而不是存储侧

这是关键设计思想：**切分策略差异在导入链路做，存储 schema 保持统一**。

- **接入文档**：按 `##/###` 标题切，代码块保护，300–500 token
- **FAQ 文档**：每个 `## Q` 就是一个 chunk，天然切好
- 两者产出统一结构 `{doc_name, doc_type, section_path, content, dense_vector, sparse_vector}` 入库

FAQ 有个切分细节值得注意：**向量化时把 Q 和 A 拼在一起**：

```python
# chunk content 存 Q+A 原文
content = "Q: 唤起海外城市页白屏怎么办？\nA: 检查 xxx 参数是否..."
# embedding 输入同样用这段拼接
```

原因：用户会用自己自然的话提问（“海外页打开白屏”），语义上更接近 FAQ 的**问题**描述；但答案里的关键词（参数名、错误码）又靠 sparse 向量召回，Q+A 一起向量化两头都能命中。

## 什么时候才需要拆

给你一个判断标准，以后文档规模变了再对照：

| 信号 | 动作 |
|---|---|
| 某类文档需要完全不同的字段（如 FAQ 要把 question/answer 拆成两个字段分别检索） | 拆 collection |
| 数据量到百万级、且查询总是限定单一类型 | 拆 collection 或用 partition |
| 需要权限隔离（FAQ 全员可见，接入文档某些业务不可见） | 拆 collection 或 partition |
| 几十~几百篇文档，类型只是元数据 | **单 collection + doc_type 字段（当前方案）** |

Partition（分区）是介于两者之间的选项，但它的价值主要在“查询时明确只查某个分区”的裁剪加速，你的场景每次都全量搜，分区没有意义，跳过。

---

**User**

在检索链路中，意图识别怎么确定用户问题是属于哪个类型呢

**Assistant**

问到点子上了——这里有个容易踩的坑：**把意图识别的目标定成“分到哪个文档类型”，然后按类型硬过滤检索**。意图识别一旦出错，正确答案会被 filter 直接过滤掉，检索环节就彻底没救了（后面 rerank、生成都无米下锅）。所以我建议分层来看这件事。

## 一、意图识别的两个价值层次

| 层次 | 作用 | 风险 | 建议 |
|---|---|---|---|
| **① 拦截 out_of_scope** | 用户问“帮我写个周报”、问支付业务的问题 → 不检索不生成，直接走兜底 | 误杀真实问题 | 高置信才拦截 |
| **② doc_type 路由过滤** | 故障排查类 → 优先搜 FAQ 文档 | **过滤错 = 召回失败** | P0 不做，P1 按置信度软过滤 |

在你当前规模（几十篇、全量检索延迟可忽略）下，意图识别的**核心价值是 ①**：省掉无意义的检索和生成、防止模型拿着不相关文档硬凑答案。② 是文档涨到几百篇、检索噪音变大后的优化项。

## 二、意图类别：按用户视角设计，不按文档视角

不要让 LLM 直接分类 `access_doc / faq_doc`——那是你的存储结构，用户嘴上说的是意图。类别应该是：

```
- access_how_to    接入咨询：怎么接入、API怎么调、参数怎么配
- troubleshooting  故障排查：报错、白屏、不生效、回调收不到
- capability       能力询问：支不支持 xxx 场景
- out_of_scope     与地址通用页面无关（含闲聊、其他业务问题）
- unclear          改写后仍无法判断
```

意图到 `doc_type` 的映射是**一对多的软映射**，维护一张表：

```python
INTENT_TO_DOC_TYPES = {
    "access_how_to":   ["access_doc"],
    "troubleshooting": ["faq_doc", "access_doc"],  # 排查问题答案也可能在接入文档的注意事项章节
    "capability":      ["access_doc", "faq_doc"],
    # unclear / out_of_scope → 不映射
}
```

这和掌柜智库里 [node_item_name_recognition.py](file:///Users/yanghao/Documents/尚硅谷-AI/05-项目_掌柜智库/day10_查询模块_out输出以及项目总结/代码/app/import_process/agent/nodes/node_item_name_recognition.py) 的设计是同构的：LLM 从 query 识别出 `item_name`（实体），查询时用它做 Milvus 标量过滤——只是那边过滤的商品名是精确实体，我们这里过滤的是意图类别，模糊性更高，所以要加置信度保护。

## 三、实现：和 query_rewrite 合并成一次调用

意图识别和多轮改写的**输入完全相同**（历史 + 新问题），没必要调两次 LLM。合并成一个节点，一次输出结构化 JSON：

```python
REWRITE_INTENT_PROMPT = """你是美团地址通用页面客服系统的查询分析器。

结合对话历史，把用户最新问题改写成脱离上下文也能独立理解的完整问题，
并判断意图类别。只输出JSON，不要输出其他内容：

{
  "rewritten_query": "...",
  "intent": "access_how_to / troubleshooting / capability / out_of_scope / unclear",
  "confidence": "high / low"
}

意图判断标准：
- access_how_to: 询问接入方式、调用方法、参数配置
- troubleshooting: 出现了报错/异常/不生效等故障现象的描述
- out_of_scope: 与地址选择、城市选择等通用地址页面完全无关
- 拿不准时 intent 填 unclear、confidence 填 low，不要猜

对话历史（最近几轮）：
{history}

用户最新问题：{question}
"""
```

节点实现：

```python
def node_query_rewrite(state):
    raw = llm_chat(REWRITE_INTENT_PROMPT.format(
        history=load_recent_history(state["conversation_id"], turns=5),
        question=state["question"],
    ))
    try:
        parsed = json.loads(extract_json(raw))
        state["rewritten_query"] = parsed["rewritten_query"]
        state["intent"] = parsed["intent"]
        state["intent_confidence"] = parsed["confidence"]
    except Exception:
        # 降级：LLM输出不合法时，用原问题、不带意图，保证链路不断
        state["rewritten_query"] = state["question"]
        state["intent"] = "unclear"
        state["intent_confidence"] = "low"
    return state
```

Graph 里意图的分支消费：

```
node_query_rewrite
    ├── intent == out_of_scope 且 confidence == high
    │        → 直接跳 node_fallback_human（不检索不生成，话术"该问题不在地址页面范围内"）
    └── 其他 → node_retrieve（正常检索）
```

## 四、P1 阶段：意图怎么用于检索过滤（软过滤）

等需要做类型路由时，检索节点这样用——**置信度是开关**：

```python
def node_retrieve(state):
    filter_expr = None
    intent = state.get("intent")
    conf = state.get("intent_confidence")

    # 只有高置信意图才启用过滤；unclear/low 全量检索
    if conf == "high" and intent in INTENT_TO_DOC_TYPES:
        doc_types = INTENT_TO_DOC_TYPES[intent]
        filter_expr = f"doc_type in {doc_types}"

    results_dense = milvus_client.search(
        collection_name="address_doc_chunk",
        data=[query_dense],
        anns_field="dense_vector",
        filter=filter_expr,   # None 时即全量
        limit=10,
        output_fields=["doc_name", "doc_type", "section_path", "content"],
    )
    ...
```

比硬过滤更稳的进阶做法是**加权不排除**：两路检索一路带 filter、一路不带，RRF 融合时带 filter 的结果权重略高。这样意图对时排序更准，意图错时答案也不会被丢掉。不过这属于锦上添花，P1 先用上面的置信度门槛即可。

## 五、兜底原则

最后回答不了的统计表里，建议加一个字段记录意图链路信息：

```json
{
  "question": "...",
  "intent": "troubleshooting",
  "intent_confidence": "low",
  "retrieved_chunks": [...]
}
```

排查“没答上来”的问题时能快速定位：是意图识别错了（filter 错杀）、还是检索没召回（chunk 切分问题）、还是 judge 太严——三类问题的修法完全不同，这个字段能省你很多排查时间。

总结一句：**意图识别在 RAG 里的正确姿势是“先当守门员（拦 out_of_scope），后当路由器（软过滤 doc_type），永远给低置信留全量检索的退路”**。

---

**User**

我准备把这个项目写到我的简历里，你看看该怎么写

**Assistant**

写简历前先明确一个定位问题：你是前端研发，这个项目在简历里的最大价值是**“前端 + AI 全栈”的稀缺组合**。我先给出可粘贴的简历文案，再说怎么按求职方向调整、面试怎么防深挖。

## 简历文案（可直接改用）

---

**项目名称**：地址通用页面智能答疑 Agent（RAG）

**项目描述**：面向美团各业务方的地址通用页面（地址选择页、城市选择页等）接入答疑场景，从 0 到 1 设计并落地基于 RAG 的多轮对话 Agent，将接入文档与历史 Q&A 沉淀为知识库自动解答，无法解决的问题自动转人工并形成 FAQ 反哺闭环，大幅降低业务对接的答疑成本。

**技术栈**：Python / LangGraph / FastAPI / SSE / Milvus / MongoDB / BGE-M3 / Reranker / LLM API

**核心工作**：

- **主导双链路架构设计**：基于 LangGraph 设计文档导入链路（解析→切分→向量化→入库）与检索问答链路（意图识别→混合检索→精排→生成→兜底），FastAPI + SSE 提供流式问答服务
- **设计 Markdown 结构化切分策略**：按标题层级切分并保留章节路径元数据，实现代码块完整性保护与超短段合并，保证 chunk 语义完整；按文档名整体替换实现知识库版本管理
- **构建混合检索链路**：针对接入类问题关键词密集（页面名/方法名/错误码）的特点，采用 dense + sparse 双路召回 + RRF 融合 + Rerank 精排，解决纯向量检索漏检问题
- **多轮对话与意图识别**：将查询改写（指代消解）与意图识别合并为单次 LLM 调用输出结构化 JSON，异常时降级保证链路可用；高置信 out_of_scope 问题直接拦截，低置信全量检索防误杀
- **设计“先判断后生成”兜底策略**：生成前先做可答性判断，不可答自动调用人工答疑接口并落库，杜绝模型编造接入参数
- **建立未解决问题数据闭环**：自动判别与用户点踩双入口采集，周度分析反哺 FAQ 文档，形成知识库自增长机制

**项目成果**：沉淀接入文档与 FAQ 共 X 篇 / X 千 chunk，日均问答 X 次，问题自动解决率 X%，业务答疑人力节省 X%（*上线后填真实数据，没有数据的项目在面试官眼里减半*）

---

## 按求职方向调整侧重

| 目标岗位 | 调整方向 |
|---|---|
| **前端岗（AI 加分）** | 补一条前端亮点：“实现 SSE 流式聊天界面（打字机渲染、多会话管理、点赞点踩反馈组件）"，把 Web 聊天页做扎实后这反而是你最容易讲深的差异化点 |
| **AI 应用工程师** | 现在这版直接用；再补评估环节（bad case 归因分析：意图错/召回错/判断严三类）会很加分 |
| **前端 + AI 复合** | 两条腿都要有：前端写交互与流式渲染，AI 写架构与检索优化 |

## 面试必被深挖的 6 个问题（写上去就要能答）

1. **为什么混合检索，不用纯向量？** → 接入问题里有精确词（`selectCity`、错误码），向量检索语义泛化反而丢精度，sparse 路保底
2. **为什么先判断可答性再生成？** → 流式输出答一半才发现编造无法撤回；judge 是廉价小调用，换来确定性的拒答体验
3. **chunk 为什么按标题切？** → 接入文档天然树状结构，一个 section 就是一个功能点，切在标题边界 = 切在语义边界；FAQ 更是天然一问一答
4. **意图识别错了怎么办？** → 软过滤不做硬路由，低置信全量检索，意图只用于排序加权不用于排除
5. **多轮改写怎么做？** → 最近 3–5 轮历史 + 新问题 → LLM 一次输出独立完整查询，解析失败降级用原问题
6. **效果怎么评估的？** → 这题答不上整个项目会露馅。准备一套说法：抽真实提问标注“可答/不可答”，统计解决率；bad case 按意图错/召回错/切分错归因

## 两个诚实的提醒

1. **这版简历的前提是你真的把它做出来**。目前我们只完成了方案设计，上面的每一行都对应着需要落地的代码。如果只写方案没写代码，面试第一个追问就会穿帮。你手上掌柜智库的代码骨架可以快速改出这个项目，工作量不大。
2. **X 数据要么填真实值要么删掉整行**，编造数据在背调和深挖里都是雷。

需要的话，我可以基于你现有的掌柜智库代码直接帮你把项目骨架搭出来，让简历上每一行都有真实代码支撑。

---

**User**

基于以上项目，面试官有可能问哪些问题？

**Assistant**

这是面试准备的黄金问题——简历上的每一行都可能被展开。我按“必问送分题 → 技术深挖题 → 压力测试题”分层列出，并标注考察意图。

## 一、必问题（100% 会被问，答不上直接出局）

**1. 整体链路讲一遍**
白板题。30 秒版：上传（MD→标题切分→BGE-M3 向量化→Milvus 入库）+ 查询（改写/意图识别→dense+sparse 混合检索→RRF→rerank→可答性判断→流式生成/转人工兜底）。讲不清全局的人，后面所有细节都会被怀疑。

**2. 为什么用 RAG，不用微调？**
答题要点：接入文档高频更新（手动重新上传即生效 vs 微调周期长）；引用事实必须准确，RAG 天然可控可溯源；文档量小，微调既不划算也没必要。

**3. 为什么用 LangGraph，不用 LangChain / 自己写 if-else？**
答题要点：链路有条件分支（judge 能答/不能答、out_of_scope 拦截），graph 声明式编排比 if-else 清晰；节点可单独测试复用；state 显式传递便于调试。别贬低 LangChain，说“Chain 适合线性流程，我的链路有分支所以选 Graph”。

**4. 项目效果怎么衡量的？**
最大送命题。要有数据：自动解决率、日均问答量、bad case 归因分布。哪怕说“我自己抽了 50 个真实业务问题标注测试”都比“感觉不错”强一百倍。

## 二、RAG 链路深挖题（体现技术深度的主战场）

**5. chunk 怎么切的？为什么 300–500 token？**
要点：按 MD 标题层级切（接入文档天然树状，一个 section = 一个功能点 = 一个语义单元）；太大→召回的 chunk 噪音多、embedding 语义被稀释；太小→上下文碎片化、检索到的信息不够生成用。追问“overlap 作用”→ 跨边界的语义不被硬切断。

**6. 代码块保护怎么实现的？**
切分前先扫描把 ` ``` ` 代码块替换为占位符，切完再还原，保证任何代码块不被从中间切开。能讲出实现细节说明真是自己写的。

**7. 为什么混合检索？纯向量不行吗？**
要点：接入类问题大量精确 token（方法名 `selectCity`、参数、错误码），dense 向量做语义泛化会损失精确匹配能力；sparse（BM25 类）保底精确词命中；RRF 融合两路。反问陷阱：“那只用 BM25 行不行”→ 不行，用户口头提问和文档用语有词汇鸿沟（“打开白屏”vs“渲染异常”）。

**8. RRF 是什么？**
公式：`score = Σ 1/(k + rank_i)`，k 常取 60。不需要分数归一化（dense 的 cosine 分和 sparse 的 IP 分不可比），只用排名，简单稳定。能写出公式这题就赢了。

**9. 为什么还要 rerank？检索完取 top-k 直接给 LLM 不行吗？**
要点：embedding 是双塔结构，query 和 doc 独立编码，交互浅；rerank（如 BGE-reranker）是交叉编码，精度高但慢。所以架构上“召回用双塔保效率，精排用 cross-encoder 保精度”——这是推荐系统的经典两段式，能类比说明你有架构迁移能力。

**10. HNSW 了解吗？为什么参数选 M=16？**
要点：分层跳表式近邻图，上层稀疏快速定位、下层稠密精确搜索；M 是每个节点的连接数，efConstruction 是建图时候选队列长度——数据量几千条用 16/200，百万级才需要 64/400，**参数和数据规模匹配**是关键，不是越大越好（内存和建索引时间代价）。

**11. 文档更新怎么处理的？中途有查询进来怎么办？**
要点：按 doc_name 先 delete 再 insert 实现版本整体替换。追问“删除和插入之间有查询进来会查到空数据”——诚实回答：日几十量级，窗口毫秒级，概率极低；严格方案可以先 insert 新数据（带版本号）、查询过滤最新版本、再删旧数据。能答出这个 trade-off 是加分项。

**12. 不同类型文档（接入文档/FAQ）为什么放一个 collection？**
要点：用户提问时不知道答案在哪类文档，单表统一检索 + rerank 自动排序最优；doc_type 是标量字段，需要时可 filter；类型差异在导入侧用不同切分策略处理，存储 schema 保持统一。追问“什么时候拆”→ 字段结构差异大/权限隔离/百万级数据。

## 三、LLM 应用层深挖题

**13. 多轮对话怎么支持的？**
要点：最近 3–5 轮历史 + 新问题 → LLM 一次改写成独立完整查询（指代消解，“那它的回调事件呢”→“海外城市页的回调事件有哪些”）。追问“为什么 3–5 轮不是全部”→ 太长稀释重点 + token 成本 + 大部分指代在近几轮内。

**14. 意图识别为什么和改写合并成一次调用？**
要点：输入相同（都是历史+新问题）、输出都是轻量结构化 JSON，两次调用延迟和成本翻倍无收益。追问“合并后怎么保证 JSON 可解析”→ prompt 强约束 + 解析失败降级（用原问题、不带意图），链路永远不断。

**15. 为什么先 judge 可答性再生成，不是生成后检测？**
要点：流式输出答一半发现编造无法撤回，体验差；judge 是一次廉价调用，换来确定性拒答；接入参数答错比不答危害大得多（业务方拿错误参数排查半天）。这个设计权衡很能体现产品思维。

**16. 幻觉怎么防的？**
多层：judge 挡“文档没有的”；system prompt 约束“仅基于提供内容回答”；检索质量本身保证有料可依。诚实补充：没有 100% 根除，靠点踩反馈捕获漏网的。

**17. SSE 和 WebSocket 怎么选的？**
要点：问答是单向服务器推流，SSE 足够且更简单（纯 HTTP、自动重连、无需协议升级）；WebSocket 适合双向（用户中途打断、语音）。前端出身答这题要答好，可能追问 EventSource API、断线重连、SSE 在 HTTP/1.1 下 6 连接数限制。

**18. LLM 调用失败/超时怎么办？**
降级链：JSON 解析失败→用原问题；embedding 失败→重试→拒绝服务；生成失败→给固定话术+落库。逐级降级而不是整体报错。

## 四、压力测试题（区分度题，答好直接进 offer 讨论）

**19. “对比一下地址选择页和收货地址选择页的接入差异”这种聚合问题怎么办？**
跨 chunk/跨文档聚合是朴素 RAG 的弱项。回答：当前方案对这类问题会召回两页文档的 chunk，依赖 LLM 生成时聚合；如果这类问题高频，演进方向是多查询拆解（一次提问拆成两个检索）或 Agentic RAG 多轮自主检索。

**20. 为什么不用 Agentic RAG（让 LLM 自主决定检索什么、检索几轮）？**
要点：场景简单（单跳问题为主），固定链路延迟可控、结果可预测；Agentic 适合复杂多跳推理，但引入不可控的循环和成本。**简单场景用简单架构**——这题考察的是克制能力，不是技术广度。

**21. 召回效果差怎么排查？一步步说。**
排查框架（体现工程素养）：先看 judge 结果（是没召回还是不敢答）→ 看 rerank 前的候选列表（召回问题 vs 排序问题）→ 召回没有→查切分（该内容是否被切碎/是否含在了不相关 section）→ 召回有但排序靠后→rerank/混合权重问题 → 都没有→文档本身缺失，进 FAQ 反哺。有 `unresolved_questions` 表的 retrieved_chunks 字段支撑归因。

**22. 一次问答的 token 成本多少？**
LLM 调用 3 次（改写~500、judge~2000、生成~3000 token 左右）+ embedding 1 次 + rerank 1 次。能粗算出单次成本和月成本，说明你不是只会调 API。

**23. 日均几十的量，为什么要做这么多优化（混合检索/rerank/意图）？**
反直觉题，考动机真实性。答：规模小不代表质量要求低——答疑场景答错一次，业务方就失去信任再也不用了；且架构在先，规模增长不需要重写。

## 五、前端出身特有的追问

**24. 流式渲染前端怎么处理的？**
必须能答：EventSource/fetch ReadableStream 处理 SSE、逐 chunk setState 的性能问题（高频 setState 卡顿→批量/节流渲染）、Markdown 实时渲染（代码块高亮在流式未闭合时的容错）、自动滚动到底部的策略（用户上翻时暂停 auto-scroll）。

**25. fallback 转人工事件前端怎么感知的？**
SSE 事件类型区分（`answer`/`fallback`/`step`/`done`），fallback 时终止打字机、展示转人工卡片。

---

## 准备建议

按优先级排：
1. **必问题 1–4 先背熟**——尤其第 4 题效果数据，先做一套小规模标注测试拿到真实数字
2. **5/7/9/15 这四题是项目的技术灵魂**，建议写成逐字稿
3. 压力题 19/21 体现深度，其他答不好可以承认“没做到”，但 21（排查方法论）必须会
4. 诚实边界：没做的别吹。面试官问“有没有做缓存”答“没有，量级不需要”比编一个答不上的方案强得多

需要我针对其中任何一题展开写完整的参考答案吗？

---

**User**

我们一个问题一个问题来，首先，整体链路详细讲一遍 

**Assistant**

好，这题是面试的“开屏题”，答好了面试官会顺着你的框架问，答差了后面全程被动。我给你一个**结构化讲法**：先 30 秒总述建立框架，再逐层展开。每个节点我都标注了**设计理由**——面试官在每个节点都会问“为什么这么做”。

---

## 第一步：30 秒总述（背下来）

> "这个项目分两条链路：**导入链路**负责把文档变成可检索的知识库，**查询链路**负责把业务方的自然语言问题变成靠谱的回答。
>
> 导入链路是：Markdown 解析 → 结构化切分 → BGE-M3 向量化 → 存入 Milvus。
>
> 查询链路是：多轮改写加意图识别 → dense+sparse 混合检索 → RRF 融合 → rerank 精排 → 可答性判断 → 能答就流式生成，不能答就转人工兜底，同时落库统计。
>
> 整体用 LangGraph 编排，FastAPI 提供 SSE 流式接口，MongoDB 存会话历史和未解决问题。"

这段话的目的是**给面试官一张地图**，他接下来的追问基本都在这张地图上，你就有主场感。

---

## 第二步：导入链路详解

### 节点 1：MD 解析（node_md_parse）

**做什么**：校验 `.md` 格式、读取原文、在 Mongo 创建导入任务记录状态。

**关键设计**：**版本管理**。知识库以 `doc_name` 为唯一键，重新上传时先 `delete(filter="doc_name == 'xxx'")` 删旧数据再插入——**“重新上传 = 整体替换”**，避免新旧版本共存导致检索到过期内容。

> 💡 面试官可能追问：“删除和插入之间有查询进来怎么办？” 答：日均几十量级，窗口毫秒级，概率极低；如果要严格保证，可以引入版本号字段先插新再删旧。

### 节点 2：结构化切分（node_md_split）—— 对检索质量影响最大

**做什么**：三个策略——

1. **按标题层级切**：以 `##`/`###` 为边界切出 section，每个 chunk 携带章节路径元数据（如 `"接入指南/海外城市页/唤起事件"`）。理由：接入文档天然是树状结构，一个 section 就是一个功能点，**切在标题边界 = 切在语义边界**
2. **代码块保护**：切分前把 ` ``` ` 代码块替换成占位符，切完还原，保证 JS 调用示例绝不被从中间切断
3. **长短修正**：超长 section 按段落二次切分 + 50 token overlap；超短 section 和相邻合并

**目标 chunk 大小 300–500 token**：太大→embedding 语义稀释、送进 LLM 的上下文噪音多；太小→信息碎片化，单个 chunk 不够生成用。

**不同类型文档差异化处理**（这里体现“存储统一、导入差异化”的思想）：
- 接入文档：标题切分
- FAQ 文档：每个 `## Q` 天然一个 chunk；**向量化时 Q+A 拼接**——用户口语化提问语义上接近 Q，答案里的参数名/错误码靠 A 里的 sparse 命中，两头兼顾

### 节点 3：向量化（node_embedding）

- 批量调 BGE-M3 API，**一次调用同时产出 dense + sparse 两种向量**（这是后面混合检索的基础）
- 向量化输入 = `章节路径 + 正文` 拼接，让标题语义参与召回
- 批量 + 失败重试

### 节点 4：入库（node_import_milvus）

**Schema**（一个 collection `address_doc_chunk`，每行一个 chunk）：

| 字段 | 类型 | 作用 |
|---|---|---|
| `doc_name` | VARCHAR | 版本删除的 key |
| `doc_type` | VARCHAR | 标量字段区分接入文档/FAQ，供查询侧过滤 |
| `section_path` | VARCHAR | 章节溯源 |
| `content` | VARCHAR(8192) | chunk 原文 |
| `dense_vector` | FLOAT_VECTOR(1024) | HNSW 索引，COSINE |
| `sparse_vector` | SPARSE_FLOAT_VECTOR | 倒排索引，IP |

**为什么所有类型文档放一个 collection**：用户提问时不知道答案在哪类文档（问“唤起没反应”可能命中的是 FAQ 也可能是接入文档的注意事项章节），单表统一检索 + rerank 自动排序是最优解；类型差异用 `doc_type` 标量字段表达，需要时 `filter` 过滤，不需要就是全量搜。

---

## 第三步：查询链路详解

### 节点 1：改写 + 意图识别（node_query_rewrite）—— 合并成一次 LLM 调用

**做什么**：从 Mongo 取最近 3–5 轮历史 + 新问题，一次 LLM 调用输出：

```json
{"rewritten_query": "海外城市页的回调事件有哪些", "intent": "access_how_to", "confidence": "high"}
```

**为什么合并**：两者输入完全相同（都是历史+新问题），分两次调用延迟和成本翻倍无收益。

**为什么必须改写**：多轮对话里用户大量使用指代——“那它的回调事件呢”单看这句根本无法检索，必须改写成独立完整查询。

**意图的用途（关键设计思想）**：
- `out_of_scope + high confidence` → 直接拦截走兜底，不浪费检索和生成
- **意图不用于硬过滤**——过滤错了正确答案就被 filter 杀掉了，检索环节彻底没救。低置信一律全量检索，宁可多搜不可漏搜

**降级**：JSON 解析失败 → 用原始问题、不带意图，链路永远不断。

### 节点 2：混合检索（node_retrieve）

```
改写后的 query ──┬─→ dense 检索 top10（BGE-M3 稠密向量，HNSW，COSINE）
                 ├─→ sparse 检索 top10（稀疏向量，倒排索引）
                 └─→ RRF 融合 → top10 候选
```

**为什么必须混合**：接入类问题有大量精确 token——方法名 `selectCity`、参数名、错误码。dense 向量做语义泛化会丢精确匹配能力；但只用 sparse 也不行，用户口语（“打开白屏”）和文档用语（“渲染异常”）有词汇鸿沟。**dense 管语义、sparse 管精确，缺一不可**。

### 节点 3：RRF 融合

公式：`score = Σ 1/(k + rank_i)`，k 取 60。

**为什么用 RRF 而不是分数加权**：dense 的 cosine 分数和 sparse 的 IP 分数**量纲完全不可比**，直接加权要先做分数归一化，很脆；RRF 只用排名不用分数，简单稳定。

### 节点 4：Rerank 精排（node_rerank）

BGE-reranker 交叉编码器，对 RRF 的 top10 精排取 top 3–5。

**为什么不直接取检索 top-k 给 LLM**：embedding 是双塔结构，query 和 doc 独立编码、交互浅、快但糙；rerank 是 cross-encoder，query 和 doc 拼一起过模型，精度高但慢。所以**召回用双塔保效率，精排用交叉编码保精度**——推荐系统的经典两段式架构。

### 节点 5：可答性判断（node_answer_check）—— 本项目的特色设计

**做什么**：把 top chunks + 问题给 LLM，只输出 `CAN_ANSWER / CANNOT_ANSWER`，一次廉价调用。

**为什么先判断再生成，而不是生成后检测**：
1. 流式输出答到一半发现是在编造，**无法撤回**，用户体验极差
2. 答疑场景**答错的代价远大于不答**——业务方拿着编造的接入参数排查半天，就永远不再信任这个系统了
3. judge 判断不能答 → 直接走转人工，用户得到确定性结果

prompt 里明确约束：“只有文档内容能直接支撑回答时才输出 CAN_ANSWER，禁止靠模型自身知识编造”。

### 节点 6：流式生成（node_answer_stream）

- LLM 流式生成，SSE 逐 token 推给前端
- system prompt 约束：仅基于提供的文档内容回答，文档没提到就说不知道
- 不展示引用来源，但 `section_path` 内部留存用于排查

### 节点 7：人工兜底（node_fallback_human）

judge 判断不能答 → 调现成的人工答疑接口（带原始问题、改写后问题、conversation_id），前端收到 `fallback` SSE 事件展示固定话术，**同时写入 `unresolved_questions` 表**。

### 横切：未解决问题闭环

两个采集入口：
1. 自动：fallback 触发落库（reason = judge_cannot_answer）
2. 手动：每条回答的 👎 按钮落库（reason = user_thumb_down）——捕获 judge 判断“能答”但答非所问的情况

周度导出 pending 记录 → 高频问题人工写进 FAQ 文档 → 重新上传 → 知识库自增长。**RAG 系统上线只是起点，这个数据闭环才是它越用越准的原因**。

---

## 讲述技巧（重要）

1. **顺序**：总述 → 导入 → 查询 → 闭环。查询链路花 60% 的时间，那是技术密度最高的部分
2. **每个节点都用“做什么 + 为什么”结构**，主动把设计权衡说出来，别等面试官问——主动权和被动的区别就在这
3. **埋钩子**：讲到“可答性判断”时放慢语速强调“这是我们踩过体验坑后的设计”，面试官大概率会追问，而那正是你准备最充分的题
4. **画图**：如果面试官给你纸笔或共享屏幕，主动画那条双链路框图，边画边讲——视觉化讲述比纯口述的清晰度高一个量级

准备好这题后告诉我，我们进入第二题“为什么用 RAG 不用微调”。

---

**User**

好的，进入第二题

**Assistant**

第二题：**为什么用 RAG，不用微调？**

这题是送分题，但 90% 的人只会答“成本低、更新快”两句就没了。面试官问这题的真实目的是考察你**对两种技术本质区别的理解**。我先给答题框架，再给深挖防御。

---

## 第一步：30 秒核心答案（背下来）

> “这个决策我从四个维度做过权衡：**知识更新、数据规模、准确性要求、可维护性**。
>
> 1. **知识更新**：接入文档高频迭代，RAG 重新上传文档分钟级生效；微调每次更新都要重新构造数据、训练、评估，周期以周计
> 2. **数据规模**：只有几十篇文档，几千个 chunk，这个量级根本撑不起一次有效微调——微调需要大量高质量 QA 对，我们最缺的就是这个
> 3. **准确性要求**：答疑场景里接入参数、方法名必须一字不差，RAG 的回答有原文可依、可溯源可校验；微调把知识‘腌’进参数里，输出无法保证逐字精确，模型可能把两个页面的参数张冠李戴
> 4. **可维护性**：RAG 答错了，我能定位到是检索没召回还是切分有问题；微调答错了，只能笼统地‘再训一次’
>
> 一句话总结：**微调改变模型‘怎么说’，RAG 决定模型‘说什么’。这个项目的问题恰恰是‘说什么’必须精确，所以 RAG 是必然选择。"

最后那句总结是这题的题眼，说出来的瞬间就和背题的人拉开差距了。

---

## 第二步：理解底层逻辑（防深挖必须懂）

面试官如果继续追问“为什么微调就不能保证参数精确？”，你要能讲到机制层面：

**微调的知识注入机制**：微调是通过梯度更新把训练语料里的模式“压缩”进模型参数。这个过程是**有损压缩**——模型学到的是统计规律而非精确存储。一个在训练集里出现过 3 次的参数名，模型可能在生成时输出一个“长得像但不一样”的变体（比如 `onCitySelect` vs `onSelectCity`）。这种现象在低频事实上尤其严重。

**RAG 的机制**：知识原文存在向量库里，生成时作为上下文喂给模型。LLM 做 copy 任务的可靠性远高于做“记忆回想”——**上下文里有的内容，模型近乎逐字复述的能力是很强的**。相当于开卷考试 vs 闭卷考试。

这个类比面试可以直接用：**微调是让学生把课本背下来考试，RAG 是开卷考试**。背下来的会记岔，开卷抄的一般不会错。

---

## 第三步：深挖防御（3 个必被追问的点）

### 追问 1：“那什么场景你会选微调？”（考你是不是只会一种方案）

千万别说“RAG 都能搞定”。标准答案：

> "微调擅长改变模型的**行为模式**而不是注入知识：
> - **输出格式/风格**：比如要求严格的 JSON 输出、特定的话术风格，几千条样本微调一个小模型效果显著且省 token
> - **领域语言的适配**：比如医疗术语的表述习惯，让模型‘学会说行话’
> - **蒸馏降本**：用大模型产出 QA 对微调小模型，替换高频调用
>
> 我们项目里如果以后 prompt 模板越堆越长、成本失控，我会考虑把 judge 这类固定任务的 prompt 微调进小模型——那是行为问题，不是知识问题。"

这个答案展示了你把两者当作**互补工具**而非对立方案。

### 追问 2：“几十篇文档而已，为什么不直接全塞进上下文（Long Context）？”（2024 年后这题比“为什么不用微调”更常见）

这是最锋利的追问，因为你的文档量真的很小。诚实且分层地答：

> "我认真评估过这个方案，它其实是可行的，但被我否掉了，三个理由：
>
> 1. **成本是持续性的**：几十篇文档约几十万 token，每次对话都要全量携带，日积月累的成本远高于检索方案；RAG 只送 top 3–5 个 chunk，单次上下文几千 token
> 2. **长上下文有‘迷失中间’问题**：多项研究（如 Lost in the Middle）表明 LLM 对超长上下文中间部分的信息利用率明显下降，全量塞入 ≠ 全量利用；检索是**主动筛选**出最相关的 1%，注意力集中
> 3. **无法定位与迭代**：全量塞入时回答错了，我不知道该修哪篇文档；检索方案有 chunk 级的命中记录，能精确归因
>
> 但我不教条——如果文档只有两三篇且永不增长，直接塞上下文确实更简单，我会选它。**架构选择跟着场景走**。"

"迷失中间"+“成本持续性”+“不教条”三连击，这题答完基本技术面就过了。

### 追问 3：“RAG 和微调结合的方案了解吗？”（考知识面）

> "了解，业界叫 RAFT（Retrieval-Augmented Fine-Tuning）之类：先微调让模型学会‘更好地利用检索结果’（比如训练它在文档没有答案时说不知道，而不是硬编），再上线 RAG。本质是微调管行为、RAG 管知识的组合。我们项目用 prompt 约束达到了类似效果，数据量上来后可以考虑。"

---

## 常见答法翻车点（避开）

| 翻车说法 | 为什么翻车 |
|---|---|
| “微调太贵了” | 不成立。几十篇文档微调一次也就几百块 API 费，成本论在面试官的反问下站不住。**成本是长期的算力/维护成本，不是单次训练费**，要说清楚是哪个成本 |
| “微调效果不好” | 笼统且暴露不懂。必须说到“有损压缩”“低频事实容易错”这个机制层 |
| “RAG 是主流所以用 RAG” | 从众论证，直接暴露没有独立技术判断 |
| 贬低微调一无是处 | 面试官里很可能有做过微调的，展示互补认知才安全 |

---

## 本题的隐藏考点

这题表面考选型，实际考的是**你的决策过程是否结构化**。答案里 “知识更新 / 数据规模 / 准确性 / 可维护性” 四个维度的框架本身，比每个维度的具体内容更能加分——它证明你做技术决策有方法论，不是拍脑袋。

面试官下一句很可能是“那你们文档量涨到几千篇怎么办？"——顺势就引出了第三题（为什么用 LangGraph）之外的另一个话题：**规模化演进**。回答方向：文档多了之后检索层的演进（意图路由过滤、分区）、切分策略的优化、评估体系的建设，架构不需要推翻。

准备好了说一声，进入第三题：**为什么用 LangGraph，不用 LangChain 或自己写 if-else**。

---

**User**

好的，进入第三题

**Assistant**

第三题：**为什么用 LangGraph，不用 LangChain 或自己写 if-else？**

这题是“选型题”三连的最后一题，也是最容易被反杀的一题——因为你的链路其实不复杂，面试官会想“这不就是十来个函数串起来吗，为什么引入框架”。答案的核心策略是：**承认备选方案都可行，再讲清 LangGraph 在你的场景下解决了什么具体问题**。

---

## 第一步：30 秒核心答案（背下来）

> "三个方案我都评估过，决策依据是**我的链路有条件分支和状态流转**：
>
> 1. **自己写 if-else**：链路确实能跑通，但查询链路里有多个分支点——意图识别后分（正常检索 / out_of_scope 拦截）、judge 后分（流式生成 / 转人工兜底）、每处都有异常降级路径。手写意味着状态在一个大函数里靠局部变量传递，分支逻辑和业务逻辑搅在一起，**两个月后自己都看不懂**
>
> 2. **LangChain 的 Chain**：Chain 本质是线性管道（DAG 的特例），用 `LCEL` 串联 `prompt | llm | parser` 很优雅，但表达条件分支要靠 `RunnableBranch`，路由一多可读性急剧下降。我的导入链路是线性的确实可以用 Chain，但查询链路不行
>
> 3. **LangGraph**：图结构天然表达分支——节点是函数、边是流转、条件边是路由，state 显式地在节点间传递。我的两条链路恰好是‘一条线性 + 一条带分支’，用 Graph 统一表达，还能白板画出来和产品对齐
>
> 一句话：**链路是线性的时候 Chain 够用，出现分支的时候 Graph 是正解，永远不想回到面条代码**。"

---

## 第二步：展示你对 LangGraph 的真实理解（防“你只是跟着教程用的吧”）

面试官验证你是否真用过，会问具体概念。准备好这几个：

**1. State 是什么、怎么流转的**

> "State 是一个 TypedDict（或 Pydantic 模型），是全链路唯一的共享数据结构。每个节点函数接收 state、返回 dict（部分更新），框架做合并。比如查询链路的 state：

```python
class QueryState(TypedDict):
    conversation_id: str
    question: str
    rewritten_query: str
    intent: str              # access_how_to / troubleshooting / out_of_scope...
    intent_confidence: str
    retrieved_chunks: list   # RRF融合后的候选
    reranked_chunks: list    # 精排后 top 3-5
    can_answer: bool         # judge 结果
    answer: str
```

好处是**数据流显式化**：任何节点需要什么、产出什么，看 state 定义一目了然，调试时断点在节点入口就能看到全量上下文。"

**2. 条件边怎么写的**

> "分支用 `add_conditional_edges`，路由函数根据 state 返回下一个节点名：

```python
graph.add_conditional_edges(
    "node_answer_check",
    lambda s: "node_answer_stream" if s["can_answer"] else "node_fallback_human",
)
```

这和 if-else 的本质区别：**路由逻辑集中声明在图构建处，节点函数保持纯净**——节点不知道也不关心自己之后流向哪，改路由不用动业务代码。"

**3. （加分项）checkpointer——LangGraph 的差异化能力**

> "LangGraph 内置 checkpointer，可以把 state 持久化到 MongoDB/Redis，支持中断恢复和 time travel。我的场景里多轮对话的会话历史如果直接用 LangGraph 的 thread 持久化，连自己维护会话存储都可以省掉。这是我选它而非自己写状态机的实际收益之一。"

哪怕你项目里没用 checkpointer，知道它并说出“我评估过但自己用 Mongo 管理更可控”都比不知道强。

---

## 第三步：深挖防御

### 追问 1：“就你这十来个节点，自己写一个 pipeline 类不就够了吗？引入框架是不是过度设计？”（最锋利的反问）

千万别说“框架功能多”。诚实且自信地答：

> "如果只看代码行数，自己写可能更少。我选框架的理由不在行数，在三件事：
>
> 1. **异常降级的结构化表达**：我的链路里几乎每个 LLM 调用节点都有降级路径（JSON 解析失败用原问题、embedding 失败重试）。手写的话降级逻辑散落在各处；Graph 里每个节点自己处理自己的降级，返回正常 state 继续流转，主流程代码里一行 try-catch 都看不到
> 2. **两条链路的节点复用**：导入和查询链路各自独立成 graph，但像 Milvus 客户端管理、LLM 调用封装是共享的，框架的节点即函数的设计让复用边界清晰
> 3. **我评估过成本**：LangGraph 的学习成本我半天就上手了，核心 API 就那几个（add_node / add_edge / compile / invoke）。**半天的学习成本换长期的维护性，这个账算得过来**。
>
> 反过来如果链路真就是一个 prompt 进一个答案出，我直接写函数，不会为了简历好看上框架。"

最后一句“反过来”很重要——展示你不是框架原教旨主义者。

### 追问 2：“LangGraph 和 LangChain 什么关系？”（考概念清晰度）

> "同一个公司（LangChain Inc.）的产品。LangChain 是先行者，主打快速拼装原型，抽象层很厚，被吐槽最多的就是封装过度、出问题难调试；LangGraph 是官方对‘复杂可控 agent’的回应，抽象更薄——节点就是你自己的函数，框架只管编排和状态，调试直接下断点进你的函数。社区共识基本是：**原型用 Chain 快速验证，生产级 agent 用 Graph 拿回控制权**。我的项目直接跳过了 Chain 阶段。"

### 追问 3：“为什么不用 LlamaIndex / Dify？”（知识面题）

> "- **LlamaIndex**：定位是数据框架，强在索引和数据连接器（各种文档源接入），agent 编排能力相对弱。我文档就是几十个 MD 文件，用不上它的数据连接器生态，而我的核心复杂度在查询链路的分支编排，恰好是它的弱项
> - **Dify**：低代码平台，适合非研发快速搭对话应用。但我的链路有大量定制逻辑（代码块保护的切分、RRF、可答性判断），用 Dify 会不停撞到平台能力边界，最后变成‘平台里嵌一堆自定义代码’，两边都不纯粹
> - 选型逻辑：**我的定制复杂度在链路逻辑本身，不在数据接入也不在界面**，所以选了对链路控制力最强的 LangGraph。"

### 追问 4：“Graph 的执行是并行的吗？你的链路里有没有并行节点？”（考实现深度）

> "LangGraph 支持 fan-out/fan-in。我的链路里有一个天然的并行点：dense 检索和 sparse 检索互不依赖，可以并行发出去再汇合到 RRF 节点，用 `add_node` 声明两个检索节点、都以 RRF 为汇聚点即可。不过实测 Milvus 的 hybrid search 本身一次请求就能查两个字段，所以我把两路检索放在一个节点里一次调用了——**并行框架能力我了解，但更简的单次调用能达成同样效果时，选简单方案**。"

（注：这个答案默认你用了 Milvus hybrid search；如果你是两路分开查的，就答并行节点的版本。两者都对，关键是说清 trade-off。）

---

## 常见翻车点

| 翻车说法 | 为什么翻车 |
|---|---|
| “LangGraph 更流行/更先进” | 从众论证，等于告诉面试官没有自己的判断 |
| “LangChain 太烂了” | 贬低性论述危险——Chain 处理线性管道依然优秀，贬低它暴露视野窄 |
| 把 LangGraph 描述得无所不能 | 面试官随手抛一个“那它的事务/回滚怎么处理”你就露馅。它是轻量编排器，不是工作流引擎，别吹成 Airflow |
| 答不出 State 和条件边的写法 | 直接暴露没写过代码。这题的概念部分（第二步）必须滚瓜烂熟 |

---

## 本题的隐藏考点

和第二题一样，这题考的是**技术决策的结构化表达能力**。注意我给的答案里反复出现的模式：

> “自己写 if-else → 什么场景够用 → Chain → 什么场景够用 → 我的需求是什么 → 所以 Graph”

这个“备选方案各有适用域 + 我的需求落在哪”的论证结构，可以迁移到任何选型题（为什么用 Milvus 不用 ES？为什么用 MongoDB 不用 MySQL？）。面试官问选型题不是要标准答案，是要看**你的决策是不是可复现的推理过程**。

准备好进入第四题（**项目效果怎么衡量的**——最大送命题，建议做项目时真的把评估跑出来）说一声。

---

**User**

好的，进入第四题

**Assistant**

第四题：**项目效果怎么衡量的？**

我之前说过这是最大送命题，原因很直白：**前面三题答得再好，说明你“会做”；这题答不上，说明你“做完没验证”**——在面试官眼里，没验证过的系统等于没做完。这题的唯一解法是：**做项目时真的把评估跑出来，面试带着数字进场**。

---

## 第一步：30 秒核心答案（背下来）

> “我建立了**离线评估 + 在线指标 + 归因闭环**三层体系：
>
> **离线**：从真实业务提问里抽了 50 个问题构建 golden 测试集（问题 + 标准答案 + 出处 chunk），每次改 prompt、改切分策略后跑回归。检索层看 Hit Rate@5（正确 chunk 是否出现在 top5）和 MRR（排得靠不靠前），生成层用 LLM-as-judge 按‘正确性、忠实性、拒答准确性’三个维度打分
>
> **在线**：核心北极星指标是**自动解决率**——问题被 Agent 解决且没被点踩的比例；辅以转人工率、点踩率、首 token 延迟
>
> **归因**：所有没解决的问题落 `unresolved_questions` 表，带 retrieved_chunks 和意图识别结果，按‘意图错 / 召回缺 / 排序差 / judge 严 / 文档缺’五类归因，每周分析反哺 FAQ
>
> 最终数据：自动解决率 X%，点踩率 X%，其中归因分布是文档缺失占大头——这也是我做 FAQ 反哺闭环的原因。"

带数字、带归因、带体系，这三样齐了这题就从送命题变成得分题。

---

## 第二步：分层指标详解（防深挖的弹药库）

### 离线评估——按链路分层测

RAG 的评估必须**分层**，因为答错可能错在任一环节，混在一起测就无法归因：

| 层 | 指标 | 测什么 | 怎么算 |
|---|---|---|---|
| 检索层 | **Hit Rate@5** | 正确 chunk 在不在 top5 | golden 出处 chunk 是否命中 / 总题数 |
| 检索层 | **MRR** | 命中了排第几 | 平均 1/排名 |
| 生成层 | **正确性** | 答案对不对 | LLM-as-judge 对比标准答案 |
| 生成层 | **忠实性** | 有没有编造 | judge 检查答案每句话是否有上下文支撑 |
| 生成层 | **拒答准确性** | 该拒的拒了没、不该拒的有没有误拒 | 分开统计两类错误率 |

**拒答准确性单独拎出来**是我们项目的特点：因为有 judge 兜底环节，存在“过度保守”风险——文档里有答案但 judge 不敢答，用户明明能自助解决却被转人工。这两类错误（该答不答 / 不该答硬答）代价不对称，必须分开盯。

**Golden set 怎么建的**（必被追问）：
> "来源两个：一是从历史答疑群、工单里捞的真实业务提问；二是拿文档反向生成——让 LLM 从每个章节生成提问，人工筛掉不自然的。50 个题里按意图类型配比：接入咨询 30、故障排查 15、out_of_scope 5（out_of_scope 专门测拦截能力）。"

### 在线指标——业务视角

- **自动解决率（北极星）**：`1 - 转人工率 - 点踩率`（简化口径，够用）
- **点踩率**：单独看趋势，突增说明某次文档更新引入了问题
- **首 token 延迟**：用户感知层面最敏感的数

### LLM-as-judge 的具体做法

> “judge prompt 给它：问题、golden 答案、Agent 实际回答、命中的 chunks，让它输出 0/1/2 三档（错 / 部分 / 对）加一句理由。三档比十分制稳定——LLM 打连续分数抖动很大，粗档位一致性好。”

---

## 第三步：深挖防御

### 追问 1：“LLM-as-judge 自己都不准，你凭什么信它？”（必问）

> "三个手段控制质量：
> 1. **先校准再用**：我先人工评了 30 条，和我自己的判断算一致率，85% 以上才敢用 judge 跑全量；不一致的 case 看是 judge 的模式化偏差还是我的标准模糊，修 prompt
> 2. **judge 用强模型**：评估模型和生成模型不能用同一个档位——生成用便宜的，judge 用强的，避免‘差生给自己打分’
> 3. **点踩数据做最终裁判**：离线 judge 只是回归测试的快速信号，真正的效果认定以线上用户反馈为准"

### 追问 2：“50 个样本太少了吧，有统计显著性吗？”（压力题）

> "诚实说没有，50 个样本 ±10 个百分点的波动都正常。但我的用法是**回归测试不是科研结论**——目的是每次改动后看指标是‘明显掉’还是‘没变化’，抓大方向问题；细粒度的质量靠线上点踩和 unresolved 归因去捕。以日均几十的量级，把评估做到统计显著既不经济也没必要。"

承认局限 + 说清定位，比硬撑“够用了”可信十倍。

### 追问 3：“用 RAGAS 这类现成评估框架了吗？”（知识面题）

> "评估过。RAGAS 的 RAG 三元组（上下文相关性 / 忠实性 / 答案相关性）和我手工搭的三个维度基本对应，框架的价值是指标现成、工业化。我没接入的原因：我的链路有 judge 拒答环节和转人工路径，标准 RAGAS 评估覆盖不了‘拒答准确性’这种定制维度，自己写 judge prompt 反而更贴合。但指标设计的思想是参考它的。"

答出三元组名字 = 知识面分到手；说清为什么没用 = 独立思考分到手。

### 追问 4：“改了 prompt 之后指标变好了 3 个点，你怎么确定是 prompt 的功劳不是波动？”（考严谨性）

> "三个控制手段：golden set 固定不变（题目和标准答案冻结）、同参数跑 3 次取均值（LLM 生成有随机性，temperature 也可以调 0 做确定性对比）、改动一次只变一个变量。3 个点在这种控制下如果稳定复现，就采信；不稳定就当噪声。"

---

## 常见翻车点

| 翻车说法 | 为什么翻车 |
|---|---|
| “用户反馈都说挺好的” | 没有体系，一句追问就崩：“多少用户？怎么统计的？” |
| “准确率 90%+”（拍脑袋数字） | 追问“怎么定义准确、怎么算的”立刻穿帮 |
| 只报生成质量，不提检索指标 | 暴露对 RAG 分层评估没有认知——这是这题的核心考点 |
| “还没来得及做评估” | 诚实但致命。**宁可现在花半天做，别面试时说没做** |

---

## 给你的行动项（面试前必须完成）

这题没有捷径，但半天到一天能搞定最小闭环：

1. **建 golden set**：拿你准备导入的文档，写 30–50 个问答对（含 5 个 out_of_scope）
2. **跑离线评估**：写个脚本批量调查询接口，LLM-as-judge 打分，输出 Hit Rate / 正确率
3. **记录真实数字**：哪怕 30 题的评估，“Hit Rate@5 = 82%，生成正确率 76%，拒答误杀 4%”这种数字也足够撑起整场面试
4. 线上指标没上线就诚实说“内测阶段，离线数据是 X，上线后计划盯自动解决率”——**有时间线的规划**比编造的线上数据可信得多

---

## 本题的隐藏考点

前三题考“会不会做”，这题考的是**工程师和调 API 的区别**：有没有质量意识、有没有闭环思维。而且注意——你简历里那个“未解决问题反哺 FAQ”的亮点，唯一的证据支撑就是这题的归因体系。这两题在面试官眼里是一体的：**评估给出问题在哪，反哺闭环解决问题在哪**，合起来才是一个完整的“越用越好”的故事。

准备好进入第五题（**chunk 怎么切的，为什么 300–500 token**——RAG 技术深挖的开胃菜）说一声。

---

**User**

下一题

**Assistant**

第五题：**chunk 怎么切的？为什么 300–500 token？**

这题是 RAG 技术深挖的开胃菜，也是区分“调过 API”和“真调过 RAG 效果”的第一道筛子。切分是整个 RAG 里**质量天花板**——chunk 切碎了，后面检索再强、rerank 再准都救不回来。面试官问这题，就是想看你是否理解这个层级关系。

---

## 第一步：30 秒核心答案（背下来）

> "我的切分原则是**按语义边界切，不按固定长度切**，四条策略：
>
> 1. **按 MD 标题层级切**：以 `##`/`###` 为边界切出 section，每个 chunk 携带章节路径元数据（如 `"接入指南/海外城市页/唤起事件"`）。接入文档天然是树状结构，一个 section 就是一个功能点——**切在标题边界，就是切在语义边界**
>
> 2. **代码块原子保护**：切分前把 ` ``` ` 围栏代码块替换成占位符，切完还原。JS 调用示例从中间断开，前半段没有方法名、后半段没有参数说明，两半都变废料
>
> 3. **长短修正**：超长 section 按段落二次切分 + 50 token overlap 衔接；超短 section（如只有一句提示语的小节）与相邻兄弟 section 合并，避免语义碎片
>
> 4. **FAQ 差异化**：每个 `## Q` 天然一个 chunk，向量化时 Q+A 拼接——用户口语化提问语义上接近 Q，答案里的方法名/错误码靠 A 的关键词命中，两头兼顾
>
> 为什么 300–500 token：这是**三重约束的交集**——embedding 模型的有效表达区间、检索精度、生成上下文需求。chunk 越大，一个向量要同时表达多个主题，语义稀释，哪个主题都不像；越小，信息碎片化，召回了但单独不够生成用。而接入文档里‘一个功能点’的自然大小恰好落在这个区间——**不是 300–500 这个数字神奇，是它和文档的语义单元大小重合**。"

最后一句是题眼：把“经验数字”升华成“和场景匹配”，直接和套通用参数的人拉开差距。

---

## 第二步：实现细节（证明真是自己写的）

面试官验证代码真实性就靠这步。切分器的处理流程：

```
原始MD
  ↓ ①预处理：正则扫描所有 ``` 围栏代码块，替换为 ⟦CODE_BLOCK_0⟧ 占位符，暂存到列表
  ↓ ②逐行扫描标题行（#数量确定层级），维护标题路径栈，切出 section
  ↓ ③长度检查（近似 token 数）：超长的按 \n\n 段落边界二切，相邻子块尾部/头部保留 ~50 token overlap
  ↓ ④短块合并：低于阈值（~100 token）的 section 与下一个兄弟 section 合并
  ↓ ⑤占位符还原为真实代码块
  ↓ 输出：{doc_name, doc_type, section_path, content}
```

两个容易被问的细节：

**token 还是字符？** 说清你的度量方式：“中文场景下我用字符数近似（一个汉字 ≈ 1 token 量级），没有精确 tokenizer 计数——切分目标是量级正确而不是精确到 token，误差 20% 不影响效果。”（诚实的工程判断，比假装精确计token可信）

**为什么不用固定大小滑动窗口（LangChain 默认那种）？** 滑动窗口在任意字符位置切，“一个参数说明从中间断开”是常态，chunk 开头结尾都是随机边界。标题切分的本质是**借用作者已经写好的语义结构**——文档作者用标题划分功能点，我直接沿用，这比任何算法切得都准。固定窗口只用作超长 section 的兜底手段。

---

## 第三步：深挖防御

### 追问 1：“为什么不是 128 或 1024？你实验过吗？”（必问）

> "两个极端的害处我都能说清：
> - **128 的害处**：一个 chunk 装不下完整的功能描述，经常是‘参数说明被切成三块’。召回其中一块，LLM 拿到的信息不完整；而且同一功能产生 3 个高相似 chunk 互相竞争排名，挤占 top-k 名额
> - **1024 的害处**：一个 chunk 混两三个主题，embedding 是全 chunk 的均值化表达，问其中一个小主题时相似度被稀释；同时塞给 LLM 的无关内容翻倍，稀释注意力还费 token
>
> 验证上，我用 golden set 对比过 256 和 512 两档的 Hit Rate@5（配合第四题的评估体系）。行业默认值也印证这个区间：LlamaIndex 默认 256、LangChain 默认 500——两个主流框架的默认值都落在这里不是巧合，是这个量级对 embedding 模型最友好。"

（如果你还没跑对比实验，这就是第四题行动项的又一个理由——半天能跑完。）

### 追问 2：“检索想要小 chunk（embedding 聚焦），生成想要大 chunk（上下文完整），这个矛盾你怎么解决？”（高级题，答好直接加分）

这题在考你知不知道 chunk 大小的**内在矛盾**。标准答案：

> "这个矛盾确实存在，300–500 是折中而不是消解。业界更彻底的解法是 **small-to-big 检索（父子块分离）**：embedding 和检索用小块（比如 128 的句子级），命中后把它的父块（整个 section 或 500 的窗口）送给 LLM 生成——**检索精度和生成上下文各取所需**。
>
> 我的方案里其实有个轻量版的影子：按标题切出的 section 本身就是‘父’，如果发现召回精度瓶颈在粒度上，演进路径就是引入子块索引、父块生成，Milvus 的 schema 加一个 parent_id 字段就行。当前 300–500 在 golden set 上没有暴露粒度问题，所以没上——**先有评估证据再上复杂度**。"

Small-to-big（也叫 sentence window / parent document retrieval）是 RAG 进阶知识面的硬通货，这题答出来面试官会记一笔。

### 追问 3：“overlap 到底解决什么问题？为什么是 50 不是 200？”

> "解决**边界语义连续性**：一个话题正好横跨两个 chunk 的切点时，没有 overlap 的话，切点附近的句子两边都不完整。overlap 保证边界内容在相邻 chunk 里各存一份，至少有一个版本被完整召回。
>
> 不能大：overlap 是以**重复存储**为代价的——200 token 的 overlap 意味着约 40% 内容冗余，同一个内容以两个 chunk 身份被召回、互相竞争排名、重复占用 LLM 上下文窗口。50 token 约一两句话，刚好覆盖‘一个话题的尾巴’，是冗余和连续性的平衡点。"

### 追问 4：“MD 里的表格怎么处理？”（接入文档里参数表很常见，这题命中率不低）

> "表格和代码块同等对待，做原子保护——参数表（参数名/类型/必填/说明）从中间切开和切代码块一样致命。具体分两种：正常长度的表格整体作为一个不可切单元；超长表格（比如几十行的错误码表）按行组切分，**每个子块重复表头行**——没有表头的表格行是不可读的。"

### 追问 5：“section_path 除了排查溯源还有什么用？”（考有没有把元数据用透）

> "三个用途：
> 1. **拼进 embedding 输入**：`【接入指南/海外城市页/唤起事件】\n + 正文`，让标题语义参与向量化——用户问‘怎么唤起’时，section 标题本身就是高匹配信号，纯正文里‘唤起’这个词可能一次都不出现
> 2. **给 LLM 的隐式定位**：生成时 chunk 前带 section_path，模型知道这段内容属于哪个页面的哪个功能，避免把海外城市页的参数安到地址选择页头上
> 3. 排查溯源（基础用途）"

第 2 点大多数人答不出，是“把元数据喂给生成侧”的意识。

### 追问 6：“如果一个 section 本身就是一个超大代码块，超过字段上限怎么办？”

> "代码块内部二切：按函数定义边界或空行分组切，保证每个子块至少是完整的函数/配置段。这是兜底逻辑——真实接入文档里极少有单个代码块超过 8KB，但我把 schema 的 content 上限（8192）设成了报警线，插入报错说明切分器有 bug 而不是默默截断。"

---

## 常见翻车点

| 翻车说法 | 为什么翻车 |
|---|---|
| "300–500 是 BGE 官方推荐的 chunk 大小” | BGE 没有官方推荐 chunk 大小，一句追问就穿帮。这个数字来自框架默认值和社区实践，要说对出处 |
| 只讲大小不讲切分策略 | 这题一半的分数在“按标题切、语义边界”上，大小只是参数 |
| 说不清 overlap 的代价 | 只知道“要 overlap”不知道“为什么不能大”，暴露没理解本质是冗余换连续性 |
| 混淆字符和 token | 说“300–500 token”结果描述的是字符数，被抓到一个概念含糊，后面所有数字都会被怀疑 |
| 不知道自己文档里代码块占多大比重 | 接入文档以代码示例为主体，这是这个场景切分的头号问题，答不出“代码块保护”这题基本就挂了 |

---

## 本题的隐藏考点

面试官在考一个层级认知：**RAG 质量的决定权排序是 数据质量 > 切分 > 检索 > 重排 > prompt**。切分排第二，且是唯一不可逆的环节——检索参数可以随时调，切坏了要重新导入。答这题时如果能主动说出“我把切分当作不可逆决策对待，所以它的策略设计最谨慎”，等于向面试官宣告你理解 RAG 的质量观。

另一个隐藏得分点贯穿始终：**所有策略都从“接入文档是什么形状”推导出来**（树状结构→标题切分；代码示例多→代码块保护；FAQ 天然问答对→Q+A 拼接）。场景驱动的技术决策，比背十个通用最佳实践值钱。

---

下一题（第六题）：**代码块保护具体怎么实现的？**——这题是第五题的实现细节放大版，考“占位符替换”的工程实现，比较短。要继续说一声。

---

**User**

下一题

**Assistant**

第六题：**代码块保护具体怎么实现的？**

这题是第五题的实现细节放大版，题面小但区分度高——**说“用占位符替换”只要五个字，但边界情况的处理暴露你是不是真的拿真实文档跑过**。真实接入文档一跑，嵌套围栏、超长代码块、说明句分离这些问题立刻撞上来。

---

## 第一步：30 秒核心答案（背下来）

> "方案是**提取 → 占位符替换 → 原子切分 → 还原**四步：
>
> 1. 切分前，正则扫描所有围栏代码块，整体摘出来存到列表，原位置换成一个独立成段的占位符 `⟦CB0⟧`
> 2. 后续的标题切分、段落二切都在“不含真实代码”的文本上做——占位符是一个不可分割的原子段落，跟着所在 section 走
> 3. 切分完成后把占位符还原成真实代码块
> 4. **embedding 输入用的是还原后的真实内容**——代码里的方法名、参数名是 sparse 召回的重要信号，不能只 embed 文字说明
>
> 三个关键工程细节：占位符前后加换行独立成段（保证段落级切分时整体移动）、长度估算按代码块真实长度计（否则带大代码块的 section 看起来很‘短’永远不会触发二切）、说明句粘连（‘调用示例如下：’这种引导句和代码块粘住不分家）。"

---

## 第二步：实现（背下代码骨架就够）

```python
import re

# 围栏代码块：``` 或 ~~~ 开头，闭合围栏必须与开头一致
FENCE_RE = re.compile(r"(?m)^(`{3,}|~{3,})[^\n]*\n(.*?)\n\1\s*$")

def extract_code_blocks(md: str):
    """提取代码块替换为占位符，返回(处理后文本, 代码块列表)"""
    blocks = []
    def _replace(m):
        blocks.append(m.group(0))
        return f"\n⟦CB{len(blocks)-1}⟧\n"   # 独立成段
    return FENCE_RE.sub(_replace, md), blocks

def restore(text: str, blocks: list) -> str:
    for i, b in enumerate(blocks):
        text = text.replace(f"⟦CB{i}⟧", b)
    return text

def real_len(text: str, blocks: list) -> int:
    """长度估算：占位符按其代表的真实代码块长度计"""
    n = len(text)
    for i, b in enumerate(blocks):
        n += len(b) - len(f"⟦CB{i}⟧")
    return n
```

**三个坑的展开**（面试官听到这三个细节会知道你真跑过数据）：

**坑 1：长度估算必须用真实长度**。占位符只有 6 个字符，一个“一句说明 + 三个大代码块”的 section 看起来只有几十字，永远不触发二次切分；还原后 chunk 直接超 8192，Milvus 插入报错，BGE-M3 的输入上限（8192 token）也会被击穿。这就是 `real_len` 存在的原因。

**坑 2：说明句粘连**。“调用示例如下：”和它引出的代码块是一体的，段落二切时如果切开，代码块孤零零地没有上下文。规则很简单：以 `：`/`如下` 结尾的短段落和紧随的占位符段落强制合并。

**坑 3：行内代码不需要保护**。`` `selectCity` `` 这种单反引号在行内，而切分发生在标题/段落边界，行内代码天然不会被切开。被问“行内代码怎么办”时别慌着也上占位符。

---

## 第三步：深挖防御

### 追问 1：“文档里教人写 Markdown 的章节，代码块里嵌套代码块（外层 ```` 四个反引号），你的正则怎么办？”（最锋利的一问）

> "诚实地讲，纯正则处理不好嵌套——非贪婪匹配会在内层 ` ``` ` 提前闭合，切分错位。我的应对分两层：
> 一来我统计过语料，接入文档和 FAQ 里几乎不存在‘教人写 Markdown’的内容，正则覆盖的是 99% 的常规围栏；
> 二来就算撞上，降级表现是‘这段切分不理想’，不是崩溃——切错的 chunk 依然完整包含内容，只是可能偏大。
> 如果语料里嵌套围栏成规模出现，正则方案就该升级为 **AST 解析**：用 markdown-it 或 mistune 把文档解析成语法树，code_block 节点天然是原子单元，切分就变成树的遍历，任意嵌套都正确。我选正则是因为它对这个语料够用，**不是因为它更对**。"

### 追问 2：“那为什么不直接上 AST，一步到位？”（你是前端，这题是你的主场）

> "这是我作为前端出身最有共鸣的取舍点——markdown-it 的 token 流我在前端渲染 MD 时天天打交道，渲染管线里代码块本来就是不可分割的 token，**切分借用渲染的同一套结构认知是最稳的**。最后没用的原因很朴素：语料是可控的几十篇内部文档，格式规整，AST 引入一个解析依赖换来的鲁棒性收益接近零；正则 + 占位符 40 行代码、零依赖、行为完全可控。**AST 方案在我的演进清单里**，触发条件是语料格式失控。"

这个答案把前端背景变成 AI 项目的加分项——面试官会记住“这个候选人把渲染管线的认知迁移到了切分器上”。

### 追问 3：“未闭合的代码块呢？写到一半的 `` ` `` "

> "正则匹配不到闭合围栏就当普通文本处理，后面的内容全被当作‘代码’留在原文里参与正常切分——**降级可用，不会丢内容**。但我在导入日志里加了告警：提取出的代码块数量和围栏行数量奇偶性不一致时就报 warning，提示人工看一眼这篇文档。切分器对脏数据的正确姿态是降级 + 可观测，不是假装数据永远干净。"

### 追问 4：“占位符万一和文档真实内容撞了呢？”

> "占位符用了 `⟦CB0⟧` 这种真实文档里几乎不可能出现的 Unicode 字符组合，而且提取前做一次预检——原文里如果已含 `⟦CB` 就换个哨兵串。成本一行代码，消掉一个概率虽小但极难排查的 bug（占位符还原错位会把 A 的代码塞进 B 的 chunk）。"

---

## 常见翻车点

| 翻车表现 | 为什么翻车 |
|---|---|
| 只答“正则替换成占位符”五个字 | 面试官等的就是细节，一句话等于交白卷 |
| 长度估算用占位符文本算 | 踩过坑的人不可能不知道这个——没说等于没跑过真实数据 |
| 说行内代码也要保护 | 暴露没想清楚切分发生在什么粒度 |
| 嵌套围栏答“正则能处理” | 硬撑，被追问一句写不出正则就当场翻车；诚实说“正则的边界，AST 是升级路径”反而是得分点 |

---

## 本题的隐藏考点

这题考的是**对“简单问题”的工程化态度**。占位符方案本身五分钟能写完，但真实世界的 Markdown 是脏的：嵌套、未闭合、超长、说明句分离——**难度全在边界情况，而边界情况只有真跑数据才会撞见**。面试官用这题筛选“跑过 demo”和“跑过真实语料”的人，你的答案里每提一个坑，都是在给“我真跑过”加一个证据。

另外注意追问 2 那个答案的套路：**把前端经验和 AI 项目焊接起来**。整场面试里你最大的风险是“AI 项目像跟着教程做的”，最大的资产是“前端出身带来的独特视角”——每找到一个焊接点（SSE 流式渲染、MD 渲染管线、EventSource），你的故事就真实一分。

---

下一题（第七题）：**为什么混合检索？纯向量不行吗？**——RAG 检索层的第一道深挖题，双塔 vs 交叉编码、RRF 细节都在后面排队。继续说一声。

---

