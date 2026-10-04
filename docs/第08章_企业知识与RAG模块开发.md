# 第8章 企业知识与RAG

[文档导航](README.md) · [上一章：报告](第07章_生产运行日报与事件复盘模块开发.md) · [下一章：会议录音知识化](第09章_会议录音转写与知识发布.md)

## 8.0 RAG、Elasticsearch与Neo4j的基本概念

阅读顺序与第5章一致：先理解概念，再看项目如何使用，最后看代码、接口和验收。下面的示意资料仅用于讲解，不是实际规程或生产结论。通用技术能力与当前实现分别说明，不能把面试中的优化思路当成已上线功能。

### 8.0.1 RAG：先找证据，再组织回答

RAG是Retrieval-Augmented Generation，即检索增强生成。假设用户问“某型号阀门出现压力波动，应查哪些处置要求”，模型仅靠训练知识无法知道企业当前有效的规程。RAG先从允许访问的资料中找证据，把问题和证据一起交给模型，再检查回答能否回到原文。

```text
入库：原文件 → 结构解析 → 分块与关系抽取 → 全文/向量/关系索引 → 审核发布
问答：问题 → 当前权限与版本范围 → 召回 → 融合 → 重排
     → 证据上下文 → 生成结论 → 引用与支持性校验 → 回答或拒答
```

RAG不改变模型权重；微调通过训练调整模型行为或能力；长上下文直接容纳更多输入，但仍受长度、成本和信息筛选影响。三者可以组合。资料频繁更新且要求可追溯时，检索适合提供外部证据，但不能保证模型一定正确使用证据。

本项目使用固定RAG流水线，主要入口为`retrieval.py`与`answers.py`。它没有让模型自主循环选择任意工具；与第5章的事件调查Agent应分开理解。

### 8.0.2 文档、块、证据与上下文不是同一个对象

| 概念 | 含义 | 项目对应 |
|---|---|---|
| Document | 稳定的资料身份 | `KnowledgeDocument`，例如同一份设备手册 |
| Version | 一次确定的资料版本 | `KnowledgeVersion`，带生效时间、审核和发布状态 |
| Chunk | 用于召回的片段 | `KnowledgeChunk`，保留正文和定位信息 |
| Evidence | 本次回答实际使用的证据 | 片段ID、原文、版本、标题与定位等 |
| Context | 本次交给模型的信息 | 问题、筛选后的证据及必要历史信息 |

块太小可能丢失前提、表头和否定条件；块太大可能混合主题、稀释相关性并占满预算。overlap表示相邻块重叠，能保留部分边界信息，也会产生重复。本项目优先使用结构和意图分段，并保留父级文本，不能描述为统一固定窗口加重叠。

父子检索的思路是用较小片段找到位置，再带回较大的父上下文。项目在父文本不超过8000字符时使用父文本，否则退回片段；即使命中，预算不足也可能不进入最终上下文。Chunk、中文字符和模型Token不是同一计量单位。

### 8.0.3 稀疏召回、稠密向量与Embedding

词法检索关注查询词与资料词项的匹配，适合术语和编号；稠密检索把文本映射成固定维度向量，利用距离或相似度寻找语义接近的片段，适合不同措辞表达相近意思。语义相近不表示型号、日期、否定条件相同，因此两路需要互补。

Embedding是生成向量的模型或过程，不负责直接生成回答。相同维度并不意味着两个模型的向量可比较：模型权重版本、文本处理方式和距离度量也影响向量空间。本项目按模型快照与代次隔离，查询与入库模型必须兼容；Qdrant使用Cosine距离配置，ES不承担本项目的向量存储。

精确近邻逐项比较；ANN（近似最近邻）用索引减少搜索开销，通常以一定召回损失换取速度。HNSW通过多层邻接图缩小候选范围，是常见ANN算法；它的内部索引图不是Neo4j里的业务知识图谱。当前代码没有实现自行训练向量模型或自动调优HNSW参数。

### 8.0.4 召回、RRF融合和Rerank重排

召回先尽可能找到相关候选，重排再对候选做更细的排序。重排无法找回从未进入候选池的证据。常见重排模型联合处理“问题—片段”，比离线编码后直接比较向量更适合细粒度匹配，但在线计算成本更高。

不同通道原始分数的尺度不同，不能把BM25分数与向量相似度直接当成同一单位相加。RRF（Reciprocal Rank Fusion）只用排名：

```text
RRF(d) = 对所有命中列表求和 1 / (k + rank(d))
本项目 k = 60；rank从1开始；代码的零起始rank写成60 + rank + 1。
```

例如某证据分别排第1和第3，贡献为`1/61 + 1/63`。这是融合优先级，不是答案正确概率。本项目每个查询的全文/向量通道各取40条，融合后最多60个候选，再调用重排；图结果也按返回顺序贡献排名分，图通道当前按事实ID排序，并非语义相关度排序。global另走范围汇集路径，不能套用普通Top-K解释。

### 8.0.5 Elasticsearch：索引、文档、字段与分片

Elasticsearch（ES）是基于Lucene的搜索引擎。Index是存放一类搜索文档的逻辑集合；Document是一个JSON记录；Field是其中的字段；Shard是索引的分片，主分片承接数据，副本分片提供冗余与读取能力。分片内由Lucene维护可搜索的数据结构。

ES文档不一定对应一份上传文件。本项目一条ES文档对应一个证据片段，`_id`使用证据UUID，同一资料版本会有多条片段。索引名称由前缀与generation构成，实际名称通过`generation_name()`生成。原件在对象存储，发布与权限事实在PostgreSQL。

副本不能与其主分片放在同一节点来提供节点级容错。仓库Compose为`single-node`，不能宣称已部署多节点高可用；主副分片数也没有在当前适配器中显式设置。green表示主副分片均已分配，yellow表示主分片可用但存在未分配副本，red表示有主分片未分配。健康状态不能代替业务权限和召回验收。

### 8.0.6 倒排索引：为什么能从词找到资料

通常说的“反向索引”在全文检索中更常称为**倒排索引**。正向查看是“片段 → 内容或词项”；倒排查看是“词项 → 包含它的片段列表”，列表还可以保存词频、位置等信息。

以下使用人为划分的词项说明结构，不代表当前`cjk`的真实分词输出：

```text
片段A：阀门 / 压力 / 波动
片段B：阀门 / 检查

阀门 → A、B
压力 → A
波动 → A
检查 → B
```

搜索“压力”时从词项入口找到A，无须逐段扫描全部原文。词项位置还能支持短语匹配。倒排索引解决“哪些片段包含这些词”；BM25等相关性算法解决“候选如何排序”；向量检索解决另一类语义相似性问题。

`_source`用于返回JSON原文，倒排结构用于词项查找，doc values是常用于排序和聚合的列式结构。它们目的不同，不能认为拿到了`_source`就表示所有字段都可搜索。

### 8.0.7 Mapping：字段应该怎样被理解和索引

Mapping规定字段类型、分析器及是否参与索引等规则，类似搜索文档的结构契约。动态映射可以根据新字段推断类型；显式映射把规则写清楚；`dynamic: strict`拒绝未声明字段，避免拼写错误或任意字段膨胀。本项目使用以下显式映射，来源为[ES适配器](../app/infrastructure/knowledge/http.py)：

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "id": {"type": "keyword"},
      "version_id": {"type": "keyword"},
      "title": {
        "type": "text", "analyzer": "cjk",
        "fields": {"exact": {"type": "keyword"}}
      },
      "text": {"type": "text", "analyzer": "cjk"},
      "locator": {"type": "object", "enabled": false}
    }
  }
}
```

| 配置 | 为什么这样设计 | 当前使用边界 |
|---|---|---|
| `id`、`version_id: keyword` | 整体词项适合ID精确筛选 | `version_id`承接数据库给出的可见版本列表 |
| `title`、`text: text` | 分析后支持全文检索 | 两者都使用`cjk` |
| `title.exact: keyword` | 同一标题的多字段表示，支持整体值操作 | 当前查询不使用它，不等于已实现精确标题查询接口 |
| `locator.enabled: false` | 原样保留定位对象，不解析其内部索引字段 | 可从`_source`返回，不能按内部页码字段直接检索 |

`text`适合分析后的正文，`keyword`适合ID、枚举和整体值。排序、分组聚合通常选keyword字段；不能为了对正文聚合就随意开启占用堆内存的fielddata。类型说明见[Elastic text文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html)与[keyword文档](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/keyword)。

已索引字段通常不能原地改成另一种类型，改变分析规则也不会自动重写旧词项。一般需要建立新索引并重新索引后切换读取。项目按generation隔离索引，但`initialize()`遇到已存在索引就返回，**不会验证或修复旧mapping，也没有自动别名切换流程**；更新分析器时必须安排重建和验证。

### 8.0.8 分词与Analyzer：写入和查询都要看

Analyzer（分析器）由字符过滤器、一个Tokenizer和若干Token Filter组成：字符过滤器先处理原字符，Tokenizer切出词项，Token Filter再做大小写转换、停用词或词项组合等处理。Tokenizer只是分析器的一部分。

索引时分析决定倒排索引写入哪些词项；查询时分析决定搜索哪些词项。两侧规则不协调会导致漏召回。ES允许独立设置`search_analyzer`；本项目没有单独设置，使用字段配置的`cjk`。见[分析器选择规则](https://www.elastic.co/docs/manage-data/data-store/text-analysis/specify-an-analyzer)。

`cjk`使用standard tokenizer，并结合宽度归一化、lowercase、CJK bigram及停用词处理。bigram对相邻CJK字符组词，不是按管道专业词典识别完整词语。比如连续汉字的词项可能跨越人理解的词语边界；必须检查实际输出，不能把中文词语直觉当成分词结果。见[Elastic语言分析器源码文档](https://github.com/elastic/elasticsearch/blob/main/docs/reference/text-analysis/analysis-lang-analyzer.md)。

下面是在ES Dev Tools中检查分词的只读请求；它不经过本项目业务API，需使用具备ES访问权限的环境：

```http
POST /_analyze
{
  "analyzer": "cjk",
  "text": "阀门压力波动 XJ-100"
}
```

观察token和position，尤其检查中文术语、型号连字符、大小写及编号是否被拆开。项目没有安装IK、维护行业自定义词典或配置同义词；这些是可评测的后续方案，不能写成已有实现。增加同义词也可能混淆型号或上下位概念，需要检查误召回。

### 8.0.9 Query DSL：match、term、bool与BM25

`match`对查询文字分析后匹配；`multi_match`扩展到多个字段。`term/terms`匹配索引中的精确词项，不执行与全文查询相同的分析过程，因此常用于keyword字段，不能把整句直接交给text字段的term查询并期待自动分词。

`bool.must`要求匹配并参与评分，`filter`只限定范围不贡献相关度，`must_not`排除，`should`表达可选或按`minimum_should_match`约束的条件。本项目查询体的核心结构如下，UUID需换成服务端实际授权的版本ID：

```json
{
  "size": 40,
  "query": {
    "bool": {
      "filter": [{"terms": {"version_id": ["已授权的版本UUID"]}}],
      "must": [{"multi_match": {
        "query": "阀门压力波动", "fields": ["title^3", "text"]
      }}]
    }
  }
}
```

`title^3`提高标题字段匹配的权重，不保证标题命中永远排第一。ES默认BM25综合词频、逆文档频率和长度归一化：词频贡献会趋于饱和，少见词更有区分力，长文本不会仅靠堆词数占优势。分数不是概率，不能跨问题直接解释成准确率。项目没有自定义similarity或调优BM25参数。见[BM25设置](https://www.elastic.co/docs/reference/elasticsearch/index-settings/similarity)。

组织、ACL、发布状态、生效时间与设备型号先由PostgreSQL计算成版本白名单，再通过`terms(version_id)`传入ES；当前mapping没有这些独立权限字段。ES返回后仍回查数据库，客户端不能自行提交一个ES过滤器来获得授权。

### 8.0.10 写入可见性：Bulk、refresh、flush与merge

Bulk把多条写入组成批请求，减少往返；HTTP成功不代表所有条目成功。本项目使用NDJSON，逐片段固定`_id`写入，检查响应`errors`，局部失败抛出`INDEX_PARTIAL_FAILURE`，并非全批原子事务。

refresh使新写入的内容对搜索可见，ES因此属于近实时搜索。项目Bulk使用`refresh=wait_for`等待刷新可见，而非每批强制刷新。flush涉及Lucene提交及translog代际管理，主要服务恢复与持久化；segment merge合并段、回收已删除文档占用的空间。三者不能互相替代，refresh也不等于业务版本已审核发布。见[refresh API](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-indices-refresh)。

translog是为恢复记录写操作的日志；写入确认的持久性还受相关配置影响。不要把“搜索到了”推断成“四个数据库已在一个事务中提交”。撤回先由PostgreSQL切断业务可见性，即使ES删除失败，残留命中也不能继续进入答案。

### 8.0.11 ES常见追问：分页、nested、重建与排障

| 面试问题 | 概念与回答方向 | 本项目情况 |
|---|---|---|
| 深分页为什么贵？ | `from + size`增大时各分片需维护更多候选；深遍历常用稳定排序的`search_after`，结合PIT固定搜索视图 | 当前RAG只取Top-K，没有实现搜索结果深分页/PIT |
| object与nested有什么区别？ | 对象数组扁平化可能丢失同一数组元素内字段配对；nested保留独立匹配边界，但增加成本 | `locator`是禁用索引的object，没有nested查询 |
| 分片越多越好吗？ | 分片增加并行性，也增加协调、内存和小分片开销，应按数据量与负载验证 | 当前单节点配置不能作为容量结论 |
| 重建如何保持可用？ | 新建索引、回填、校验、追平变更、切换读取；别名切换只是其中一环 | 已有代次隔离，没有完整在线迁移编排 |
| 写入成功却搜不到？ | 依次检查索引名/代次、Bulk条目错误、refresh、授权版本、分词输出、查询字段 | 还需确认数据库版本已发布且在生效范围 |
| 编号搜不准怎么办？ | 检查分析结果，必要时建独立keyword字段；词法匹配不天然等于编号精确匹配 | 当前没有独立文档编号字段的ES精确查询 |

这些通用机制可继续查阅[Elastic分页](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results)、[nested类型](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/nested)与[translog说明](https://www.elastic.co/docs/reference/elasticsearch/index-settings/translog)。

### 8.0.12 Neo4j：节点、关系、属性、标签与Cypher

Neo4j采用属性图模型：节点表示对象，关系连接节点，节点与关系可以携带属性；Label为节点分类，关系类型描述连接类型，Path是节点和关系构成的路径。Cypher用图模式表达查询，`MATCH`查找、`WHERE`过滤、`RETURN`返回，`CREATE`创建，`MERGE`匹配或创建模式。见[Neo4j图概念](https://neo4j.com/docs/getting-started/appendix/graphdb-concepts/)。

知识图谱是组织领域实体与关系的方法，图数据库是存储和查询手段，GraphRAG是利用图结构辅助检索与生成的一类方法。这三个概念不能画等号。关系型数据库同样能保存关系；图模型适合围绕连接展开的问题，并非所有查询都比SQL更快。

例如“设备A—存在现象—压力波动”必须关联原文和版本。同名实体也不必然是同一个真实设备；当前用实体字符串作key，没有完整的实体消歧或企业主数据对齐。

### 8.0.13 项目图模型：为什么把事实单独建成节点

本项目不是为每一种业务关系动态创建一种边，而是将source、relation、target、evidence_id、version_id保存到`KnowledgeFact`节点，通过两类固定关系连接实体：

```text
(KnowledgeEntity {key: 设备A})
             ↑ SUBJECT
(KnowledgeFact {source: 设备A, relation: 存在现象,
                target: 压力波动, evidence_id: …, version_id: …})
             ↓ OBJECT
(KnowledgeEntity {key: 压力波动})
```

两条箭头都从事实节点发出。业务关系名称是事实节点上的`relation`属性，不是`SUBJECT/OBJECT`边的类型。这样同一对实体的不同资料、不同版本和不同证据可以各自保留事实身份，查询后还能回到PostgreSQL核验。

唯一约束分别是事实的`(namespace, generation, id)`和实体的`(namespace, generation, key)`。namespace来自索引前缀，generation隔离代次；它们不是用户权限校验的替代品。`MERGE`复用身份，`SET`更新属性，唯一约束约束重复节点；不要宣称单靠任意MERGE语句就能保证所有并发业务幂等。见[MERGE语义](https://neo4j.com/docs/cypher-manual/current/clauses/merge/)。

### 8.0.14 邻接、多跳与图检索的边界

邻接查询从一个实体找到相连事实；多跳把事实另一端的实体作为下一轮起点。项目每次`neighbors()`查一轮，应用层最多扩展两轮，每轮前沿最多5个实体，每个实体最多取20个事实；`seen`避免重复扩展。这里“一跳”按业务实体扩展理解，物理图中实体到实体经过事实节点和两条边。

Cypher通过参数传递实体名和版本列表，使用固定查询；没有模型生成并执行任意Cypher的能力。每轮还回查PostgreSQL中的事实ID、source/relation/target与evidence_id，防止把外部图返回值直接当作可信证据。

图查询常见风险是起点扫描、超级节点与路径爆炸。索引/约束帮助定位起点，限制方向、关系类型、跳数和结果范围控制扩展；`EXPLAIN`查看计划，`PROFILE`实际执行并报告统计，二者不能混淆。项目已有有限邻接扩展，但没有社区发现、社区摘要或完整全局图推理。

### 8.0.15 RAG面试常问：效果、幻觉与评测指标

| 概念或问题 | 怎样解释 | 当前实现边界 |
|---|---|---|
| Recall@K | 前K条找回了多少已标注相关证据，占全部相关证据的比例 | 已计算recall，标注依赖片段ID |
| MRR | 首条相关结果排名倒数的平均值，关心“多快看到第一条正确证据” | 已计算mrr |
| nDCG | 对靠前的相关结果给予更大权重，再除以理想排序得分 | 当前使用二元相关标签，不是专家多级相关度 |
| Precision@K | 前K条中相关证据的比例，衡量噪声 | 当前评测结果未输出此指标 |
| 忠实性与正确性 | 忠实性看结论是否由上下文支持；正确性还需可信标准答案 | 引句匹配和模型复核不能替代领域正确性评审 |
| 检索失败与生成失败 | 先看证据是否召回、是否被重排或预算丢弃，再看模型是否误读 | 结合trace的routes、rerank、selected_ids定位 |
| 提示注入 | 资料中的指令可能诱导模型偏离用户任务 | 资料只作证据，权限靠服务端校验；不能只依靠提示词 |
| 拒答率越高越安全吗？ | 既要看无证据时拒答，也要看有证据时是否错误拒答 | should_refuse匹配只是一个指标，不等于答案准确率 |

Recall示例：标注相关证据为A、B、C，返回B、X、A，则Recall为2/3，首条即相关所以该样本倒数排名为1。代码对“标注为空”的情况另有约定，详见[evaluation.py](../app/modules/knowledge/application/evaluation.py)；比较不同实验必须保持数据集和口径一致。

常见优化应按失败阶段进行：先检查解析与分块，再检查版本范围、分析器与召回，再检查重排、上下文和生成。增大Top-K可能带来噪声和延迟，不保证改善答案；更换Embedding后必须处理已有向量兼容性。第8.13节给出项目实际评测边界，[项目面试题](项目面试题.md)提供可口述的问答。

## 8.1 业务定位

知识模块面向规程、标准、设备手册、案例、交接班材料和通知等复杂资料，提供可审核、可追溯、可撤回的知识检索与问答。复杂性来自资料结构、权限、版本、生效时间和引用可信度，不来自堆叠Agent。

典型问题包括：某型号设备如何操作、指定历史日期应适用哪个版本、案例中的异常与处理措施有什么关系、不同资料是否存在冲突。没有足够证据时返回未知或拒答，不把模型常识冒充企业制度。

它是现有模块化单体中的knowledge模块，不是另一个独立业务系统。交接班、事件、巡检和报告不会自动全部进入知识库；会议模块提供显式发布适配器，但仍进入知识审核流程。知识内容仍需显式入库、审核和发布。第5章的多Agent调查是另一个业务流程，当前RAG问答不使用多Agent/ReAct循环。

## 8.2 代码与部署边界

| 层次 | 位置或组件 | 责任 |
|---|---|---|
| 业务 | [app/modules/knowledge](../app/modules/knowledge) | 权限、版本、意图分段、检索融合、引用、评测 |
| 装配 | [app/bootstrap/knowledge.py](../app/bootstrap/knowledge.py) | 知识资源构造与生命周期 |
| 知识运行时 | [services/knowledge_runtime](../services/knowledge_runtime) | 一个进程提供Docling解析、OCR、Embedding与Rerank接口 |
| 部署 | [infra/compose](../infra/compose) | `project.yml`编排知识运行时，`shared.yml`编排三个索引服务 |
| 业务事实库 | PostgreSQL | ACL、发布状态、版本、证据、任务和运行记录 |
| 全文索引 | Elasticsearch | 词法/全文召回，不保存项目的向量检索职责 |
| 向量索引 | Qdrant | 稠密向量召回，按模型代次隔离 |
| 关系索引 | Neo4j | 带证据来源的实体关系与邻接扩展 |
| 对象存储 | StorageProvider | 原文件和解析产物等对象 |

三个索引是可重建的派生视图，不是权限事实源。自有服务代码仍在仓库里，Docker只是隔离依赖和资源；业务编排没有被拆进解析或模型容器。

## 8.3 核心数据模型

定义见 [domain/models.py](../app/modules/knowledge/domain/models.py)；输入契约见 [domain/schemas.py](../app/modules/knowledge/domain/schemas.py)。

| 实体 | 含义 |
|---|---|
| KnowledgeBase | 知识空间、组织与读者ACL |
| KnowledgeDocument | 稳定文档身份、编号、类型及文档级ACL |
| KnowledgeVersion | 上传文件、摘要、生效区间、设备型号、审核发布与索引代次 |
| KnowledgeChunk | 证据片段、父级上下文、定位信息 |
| KnowledgeFact | 实体关系、证据ID和原文引句 |
| KnowledgeConversation | 多轮问答的会话身份 |
| KnowledgeRun / KnowledgeRunEvent | 问答结果、证据、检索轨迹和可续读事件 |
| KnowledgeFeedback | 用户反馈，不直接改写模型或知识 |
| KnowledgeEvaluation | 数据集、评测任务和结果 |

文档类型为standard、manual、procedure、case、handover、notice。版本上传上限为200 MiB，要求小写十六进制SHA-256和带时区的生效时间，结束时间必须晚于开始时间。版本可携带设备型号集合及authority元数据；这些字段不是自动证明资料权威性的机制。

## 8.4 版本生命周期

```text
创建版本/上传授权 → 上传原文件 → complete 服务端校验 → uploaded
→ process 创建任务 → processing → review → approved → published
                                          ↘ rejected
处理失败 → failed；允许的状态可校正产物后重新处理
published → withdrawn（退出普通检索）
```

审核通过不等于已经发布。发布要求对应索引准备完成，并检查生效区间等约束。新版本发布时处理旧版本的有效期衔接；旧版本仍可保留published状态支持历史查询，不应简单理解成“一个文档只能有一个published行”。

产物校正支持失败、待审核或驳回等允许状态，且不能有活动任务。它针对结构块内容保存修订，不篡改原文件字节；之后重新处理并清理旧派生结果，不能只修改一个索引后直接宣称三个索引一致。

撤回先改变数据库可见性，不保证立刻物理删除三个外部索引的所有残留。数据库白名单和读取复核负责防止残留证据继续被使用；物理清理与逻辑撤回是不同动作。

## 8.5 解析：保留结构，再确定边界

[pipeline.py](../app/modules/knowledge/application/pipeline.py) 调用解析服务提交文件字节并轮询任务，传递的不是任意本机文件路径。服务使用持久任务记录和受控子进程，限制文件大小、转换时间和输入；业务轮询也有截止时间，不能无限等待。

结构化产物保留块ID、正文、标题、表格及页面/位置等信息。定位能力取决于文件类型与解析结果，不能保证所有格式都有精确页面框。缺少文本的图像块会暴露质量问题，不代表当前已实现所有图表和扫描件的准确理解。

分段见 [chunking.py](../app/modules/knowledge/application/chunking.py)：

1. 先按结构块处理，不把整个文件直接切成等长滑动窗口。
2. case/handover调用文本模型规划意图边界，每批最多30块，规划输入每块最多1200字符；返回的是已有块ID，未知ID不作为合法边界。
3. 模型规划只决定边界，不替换原文。正文保留原始块内容，避免摘要式分块丢失证据。
4. 标题等结构驱动分组；普通文本缓冲约3500字符。
5. 表格按每组15行拆分并重复前两行表头，同时保留父级上下文，降低表头丢失造成的误读。
6. 超过6000字符的块使用句子切分和硬长度兜底。结构/意图分段并不意味着永远不能使用长度保护。

这里的阈值是当前实现常量，不是经过全行业验证的最优参数。意图边界的规划输入会截断，跨批长主题也可能切断，需要用实际文档集评测。

## 8.6 实体关系与证据约束

文本模型从证据中抽取source、relation、target、evidence_id、quote。关系必须对应已有证据，原文引句必须存在，关系两端需要在引句中出现；按受控批次提取，单次输出数量受Schema限制。

Neo4j用于关系扩展，数据库保存规范事实记录。取回图结果后再次核对事实ID、属性与证据，不能把外部索引返回的任意关系当成可信知识。

当前实现是证据约束的实体关系检索，不包含社区发现、社区摘要或完整的分层GraphRAG。关系抽取模型也会漏报、误判，精确引句校验只能排除部分错误。

## 8.7 三路索引与失败恢复

入库任务生成片段、关系、向量及索引代次，在ES、Qdrant、Neo4j建立对应派生数据，并保存处理检查点。任务使用执行租约，结果落库前检查执行令牌，防止失去执行权的旧Worker覆盖新尝试。

PostgreSQL与三个索引没有分布式原子事务。因此需要阶段状态、重试清理、发布门槛和查询时数据库复核共同保证可用性，不能把它写成“四库同一事务”。

故障处理顺序：先查版本当前Job与失败阶段，再查解析/模型/索引服务，最后按平台重试接口创建新尝试。重试在同一数据库事务中重绑业务资源当前Job；不要手改job_id或把failed直接改成published。

## 8.8 配置快照与向量兼容

知识能力别名为knowledge-text、knowledge-parser、knowledge-embedding、knowledge-rerank。无别名时使用启动配置；显式配置却禁用或类型不符时拒绝执行，而不是静默绕过管理员设置。

[configuration.py](../app/modules/knowledge/application/configuration.py) 固化模型名、model_snapshot、维度和参数。同步与Worker按快照构造作用域资源，运行后释放，不把一个任务的配置覆盖到另一个任务的共享客户端上。

别名只选择参数，不负责下载和部署权重。外部服务必须真的提供匹配模型版本。vector/hybrid/global要求查询模型、维度、版本与目标索引代次相容；跨不兼容历史代次需要缩小范围或重建索引，当前没有自动路由到多套历史Embedding模型。

fulltext和graph不需要为查询调用Embedding，也不因向量模型代次不匹配而拒绝。重排、文本规划等其他能力仍可能被调用；“不调用Embedding”不等于“完全不调用模型”。

## 8.9 权限、时态与可见范围

[access.py](../app/modules/knowledge/application/access.py) 综合账号/组织有效性、操作权限、数据范围、知识库及文档ACL、发布状态、生效时间和设备型号确定可用版本。

- 默认只使用当前可见的已发布资料；as_of支持指定历史时点。
- 显式version_ids可选历史版本，不简单套用默认时效过滤，但仍检查发布状态与访问权限。
- 未发布内容走编辑/管理权限，不是普通读者直接可见。
- ACL更新有expected_version，避免静默覆盖并发修改。
- 超过实现允许的版本范围会拒绝并要求缩小范围，不能悄悄截断后假装结果完整。

数据库白名单先限制检索，取回后再核验证据；生成前后和读取历史运行时继续检查访问资格。会话历史不能成为绕过撤权的后门。当前保存的scope_version_ids涵盖运行涉及的可见范围，范围中资料撤权可能使旧回答整体不可读，而不只是删除某条引用。

## 8.10 检索模式

实现见 [retrieval.py](../app/modules/knowledge/application/retrieval.py)。

| mode | 当前路径 | 注意事项 |
|---|---|---|
| fulltext | ES → 重排 | 不生成查询向量 |
| vector | Embedding → Qdrant → 重排 | 要求向量代次兼容 |
| hybrid | ES + Qdrant + 可用实体的关系扩展 → 融合 → 重排 | 默认模式，不保证每次都有图命中 |
| graph | 查询规划 + ES + 图扩展 → 重排 | 不是纯Cypher查询 |
| global | 多路候选后，在许可范围内汇集全部片段 | 有硬范围/上下文限制，不做普通Top-K重排 |

expand_queries可开启查询改写；graph/global自动使用查询规划。规划可给出多个查询和实体；默认非规划实体识别偏向规则匹配，不能等同完整中文实体识别模型。

融合采用RRF，分数贡献为1/(60+rank+1)，先限制到最多60个候选再重排。关系扩展限制跳数与前沿，避免任意遍历企业图谱。普通模式最终条数由limit控制，默认8，范围1–20。

上下文优先使用不超过8000字符的父文本，否则退回片段，并记录完整性信息。普通预算为24000字符，全局为45000字符；global范围超过1000个片段会拒绝，预算不足也可能不能纳入全部文本。字符预算不是精确Token预算。

因此“总结所有制度”不代表无上限读取或精确全量统计；对穷尽性问题，应先确认范围和完整性标记。

## 8.11 引用问答与多轮上下文

[answers.py](../app/modules/knowledge/application/answers.py) 的流程为：权限范围 → 检索证据 → 保存轨迹 → 生成结构化结论 → 引用检查 → 模型支持性复核 → 保存结果。

输出包含claims、conflicts、unknowns、refused。每条claim使用evidence_id和quote，必须匹配实际检索证据及原文。无证据、引用不合法或支持性复核失败时不能继续包装成确定答案。

模型支持性复核仍使用配置的文本Provider，不是独立事实裁判；精确引句存在也不能证明推理必然正确。人工反馈与领域评测仍必要。

多轮问答可使用最近三次成功且仍有权读取的历史运行进行追问改写，之后重新检索，不直接无限拼接全部聊天历史。会话不等于拥有历史资料的永久访问授权。

这是受控RAG流水线，不是模型自由选择工具、反复规划的ReAct循环。复杂调查需要时走第5章，而不是给每次知识问答强加五个角色。

## 8.12 事件、取消、重试与观测

POST /runs创建异步运行；SSE /runs/{run_id}/events提供retrieving、generating、verified_claim、completed等阶段信息，支持Last-Event-ID或after续读。它不是逐Token输出。连接有轮询时限，客户端需要允许断线重连，并处理重新鉴权失败。

取消是协作式请求，外部模型调用不保证瞬间中止。重试创建新Job并重绑同一业务运行；历史事件序号可继续增长，不应把重试当成新会话或清空旧尝试。

Parser、Embedding、Rerank调用通过能力观测记录日志，未知使用量保留null。日志随业务事务提交，回滚时可能一起回滚，不能宣称是独立持久的全量失败审计；费用配置为零也不代表服务商实际免费。

## 8.13 评测与反馈

[evaluation.py](../app/modules/knowledge/application/evaluation.py) 接收development/held_out数据集，每套1–300条问题，每条包含相关证据ID与should_refuse。当前评测模式支持fulltext、vector、hybrid、graph，不包含global。

结果包含Recall、MRR、nDCG、拒答匹配和延迟。相关证据ID必须来自实际入库版本；换了分段或代次后需要检查标注仍有效。拒答匹配不是完整的答案事实正确率，也没有自动完成领域专家验收。

反馈类别包括helpful、wrong_citation、incomplete、outdated、incorrect。反馈只保存用于分析，不自动微调模型、修改ACL或覆盖已发布知识。

仓库`dev/fixtures/knowledge_corpus`保存合成测试语料，不是实际客户数据或真实企业效果报告。评测报告应说明语料来源、版本、模型快照、模式、标注方法及是否留出测试集。

## 8.14 API导航

以下省略 /api/v1/knowledge前缀，完整契约见 [router.py](../app/modules/knowledge/api/router.py)。

| 方法 | 路径 | 用途 |
|---|---|---|
| POST / GET | /bases | 创建 / 查询知识库 |
| POST | /bases/{base_id}/acl | 更新库ACL |
| GET | /bases/{base_id}/documents | 文档列表 |
| POST | /documents | 创建文档 |
| POST | /documents/{document_id}/acl | 更新文档ACL |
| POST / GET | /documents/{document_id}/versions | 创建 / 查询版本 |
| POST | /versions/{version_id}/uploads:complete | 校验上传 |
| POST | /versions/{version_id}:process | 入库任务，202 |
| GET | /versions/{version_id}/artifact | 解析产物 |
| POST | /versions/{version_id}/artifact:correct | 产物校正 |
| GET | /versions/{version_id}/evidence | 证据片段 |
| POST | /versions/{version_id}:review | 审核 |
| POST | /versions/{version_id}:publish | 发布 |
| POST | /versions/{version_id}:withdraw | 撤回 |
| GET | /versions/{version_id}/original | 原件访问 |
| GET | /versions/{version_id}/diff/{other_id} | 版本差异 |
| POST | /search | 同步检索，不生成完整问答运行 |
| POST / GET | /runs、/runs/{run_id} | 创建问答 / 读取结果 |
| GET | /runs/{run_id}/events | SSE阶段事件 |
| POST | /runs/{run_id}:cancel | 请求取消 |
| POST | /runs/{run_id}/feedback | 反馈 |
| POST | /conversations | 创建会话 |
| GET | /conversations/{conversation_id}/runs | 会话运行历史 |
| POST / GET | /evaluations、/evaluations/{evaluation_id} | 创建评测 / 读取结果 |

检索请求的question必填，支持base_ids、version_ids、equipment_model、as_of、mode、limit、conversation_id和expand_queries。未知输入字段会被拒绝；时间字段需要时区。

## 8.15 启动与验收

完整部署命令与凭据要求统一见[infra/README.md](../infra/README.md)。本地`.env`同时为宿主机后端和两套Compose提供连接配置、服务凭据与固定模型版本，不要提交真实密钥。

Worker消费knowledge_index、knowledge_query、knowledge_eval；解析轮询属于入库任务，不需要虚构独立knowledge_parse执行器。保留maintenance队列和Beat/Outbox发布。容器里的localhost与宿主机不同，后端容器部署必须配置实际可达地址。

验收顺序：

1. 验证数据库迁移、认证、对象存储和任务基础链路。
2. 上传含标题、跨页表格、长段落和案例意图的资料，检查产物、分段与定位。
3. 处理、审核、发布；分别检验全文、向量、混合、图及受限全局问题。
4. 检验版本生效、设备型号、ACL撤销、历史问答重新读取及撤回后的残留索引。
5. 模拟解析/模型/索引失败，确认不可提前发布，并验证重试绑定、租约和旧尝试隔离。
6. 用带真实证据标注的留出集比较召回、排序、引用和拒答，再做人工答案评审。

数据库测试会重建public schema，先配置可丢弃的独立TEST_DATABASE_URL，再按范围运行：

```bash
uv run pytest tests/test_knowledge_foundation.py tests/test_knowledge_quality.py tests/test_knowledge_runtime_service.py -q
uv run pytest tests/test_knowledge_business.py tests/test_knowledge_runtime_db.py tests/test_consistency_db.py -q
```

真实ES/Qdrant/Neo4j测试位于test_knowledge_external.py，由RAG_EXTERNAL_TEST=1显式开启；真实模型测试位于test_knowledge_live_models.py，分别由RAG_MODELS_TEST=1、RAG_LLM_TEST=1开启。仅在隔离服务和可丢弃测试数据上运行；真实LLM会产生网络调用及潜在费用。默认跳过不等于这些能力已验收。

## 8.16 当前限制

- 不提供无限上下文、无损全资料统计或完整社区型GraphRAG。
- 不自动在多个不兼容历史向量模型间路由。
- 不保证任意扫描件、图表、复杂跨页表格都能准确解析。
- 不把引用字符串匹配或同模型复核等价为事实正确性证明。
- 不提供四库原子事务、所有外部调用即时取消或全量日志独立落库保证。
- 不自动把其他业务模块的数据同步、审核并发布为知识。

上述边界应纳入验收标准和后续需求清单。
