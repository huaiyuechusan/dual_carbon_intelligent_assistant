# “双碳”智能问答系统项目计划（plan.md）

## 1. 项目目标

基于 **Ollama 本地部署的 DeepSeek R1 8B**，构建一个面向中小企业的“双碳”智能问答系统，实现：

1. **知识整合**：构建可持续更新的私有知识库，覆盖国家及地方“双碳”政策、技术标准、行业指引、通知公告。
2. **智能问答**：支持自然语言政策咨询、条文定位、适用范围判断、政策对比、报告背景自动生成与润色。
3. **低成本可部署**：适配 **RTX 4060 8GB** 单卡，优先本地化、轻量化、低依赖。
4. **高隐私性**：所有推理、检索、知识库与日志默认本地部署，不依赖外部 API。

---

## 2. 关键约束与设计结论

### 2.1 已知约束

- GPU：**RTX 4060 8GB**
- 核心模型：**DeepSeek R1 8B（Ollama）**
- 后端：**FastAPI + Python 3.11**
- 前端：**Streamlit**
- 数据库：**SQLite**

### 2.2 设计结论

1. **主模型可用，但上下文不能贪大**
   - DeepSeek R1 8B 可以在 Ollama 本地运行，但 8GB 显存下不应追求超长上下文。
   - 实际工程中建议上下文控制在 **4K～8K** 级别，避免显存压力和吞吐下降。

2. **必须采用 RAG，而不是把政策全文直接塞给模型**
   - 政策咨询、条文定位、报告生成都需要可追溯引用，单纯聊天式调用无法保证准确性与时效性。

3. **建议生成模型与 embedding 模型分离**
   - DeepSeek R1 8B 负责生成与推理。
   - 另起一个轻量 embedding 模型（同样走 Ollama，本地部署）负责向量化检索。

4. **SQLite 可以保留，但向量检索不要一上来做复杂中间件**
   - 初版建议用：
     - SQLite 存元数据、版本、知识条目、任务状态
     - SQLite FTS5 做关键词检索
     - Python 内存/本地文件中的向量索引做语义召回
   - 若后续知识量明显增长，再考虑替换为专门的向量引擎。

5. **前端Streamlit**
   -  **Streamlit 管理台/演示端**。

---

## 3. 成熟技术调研结论

## 3.1 模型与推理层

### 推荐方案
- **主生成模型**：`deepseek-r1:8b`（Ollama）
- **Embedding 模型（建议新增）**：优先 `bge-m3`；低资源兜底 `embeddinggemma`

### 原因
- DeepSeek R1 8B 适合中文问答、归纳、报告润色、步骤推理。
- embedding 模型应独立，避免用生成模型硬做向量任务，降低复杂度并提高召回质量。

### 不建议
- 不建议在初版中引入多模型路由、多智能体编排。
- 不建议一开始就使用超长上下文、复杂工具链或重型向量数据库。

---

## 3.2 知识库与检索层

### 推荐路线：轻量混合检索（Hybrid Retrieval）

**检索流程：**
1. 用户问题进入系统
2. 查询改写（可选）
3. FTS5 关键词召回 TopK
4. Embedding 语义召回 TopK
5. 合并去重
6. 轻量重排（规则分 + 相似度分）
7. 将最相关片段送给 DeepSeek R1 8B 生成答案
8. 输出答案 + 引用来源 + 生效日期/发布日期

### 推荐存储结构

#### SQLite 表设计建议
- `documents`
  - `id`
  - `title`
  - `source_url`
  - `source_site`
  - `region`
  - `doc_type`
  - `publish_date`
  - `effective_date`
  - `status`
  - `hash`
  - `version`
  - `raw_text`
  - `created_at`
  - `updated_at`

- `chunks`
  - `id`
  - `document_id`
  - `chunk_index`
  - `section_path`
  - `content`
  - `token_count`
  - `keywords`
  - `embedding_path`
  - `created_at`

- `knowledge_sources`
  - `id`
  - `name`
  - `base_url`
  - `source_type`（html/pdf/api/list）
  - `crawl_config_json`
  - `enabled`
  - `last_sync_at`

- `sync_jobs`
  - `id`
  - `source_id`
  - `job_type`
  - `status`
  - `message`
  - `started_at`
  - `finished_at`

- `qa_logs`
  - `id`
  - `question`
  - `rewritten_question`
  - `answer`
  - `retrieved_chunk_ids`
  - `latency_ms`
  - `feedback_score`
  - `created_at`

### 为什么这样设计
- **可追溯**：保留来源、版本、地区、日期
- **可更新**：每次同步只更新变更文档
- **可审计**：可回溯“这条回答引用了哪些政策片段”
- **可扩展**：以后接 Milvus/Qdrant 也不需要大改业务层

---

## 3.3 文档解析与数据接入

### 推荐文档解析链路

1. **HTML 政策页面**：`requests + BeautifulSoup4`
2. **PDF（文字型）**：`PyMuPDF`
3. **PDF（扫描型/复杂版式）**：`unstructured`（按需启用）
4. **Office 文档转 Markdown**：`markitdown`（可选）

### 原则
- **默认走轻量链路**：HTML / 可复制文本 PDF 优先
- **遇到扫描件再启 OCR/复杂解析**，不要让所有文档都走重型处理
- 对每个文档保存：
  - 原始文件
  - 解析文本
  - 清洗文本
  - chunk 结果
  - 版本 hash

### 适合纳入知识库的来源

#### 国家级
- 中国政府网政策文件库
- 国家发展改革委
- 生态环境部
- 国家标准信息公共服务平台

#### 地方级
- 各省/市发改委
- 各省/市生态环境厅（局）
- 地方政府政策文件库

### 采集原则
- 只采 **官方来源**
- 优先采集 **政策正文页 / 标准公告页 / PDF 原文**
- 对转载站点只做“发现”，不做最终入库来源

---

## 3.4 后端技术选型

### 推荐
- **FastAPI**：API 主框架
- **SQLModel**：SQLite ORM/模型层
- **Pydantic Settings**：环境变量配置管理
- **Uvicorn**：ASGI 运行
- **APScheduler**：定时更新任务
- **Ollama Python / HTTP API**：本地模型调用

### 不建议
- 初版不建议上 Celery + Redis
- 初版不建议拆成太多微服务
- 初版不建议加入过多 AI 中间层框架

### 推荐服务划分

- `app/api/`
  - 问答接口
  - 报告生成接口
  - 文档管理接口
  - 同步任务接口

- `app/services/`
  - `llm_service.py`
  - `embedding_service.py`
  - `retrieval_service.py`
  - `ingestion_service.py`
  - `sync_service.py`
  - `report_service.py`

- `app/repositories/`
  - 面向 SQLite 的数据访问层

- `app/core/`
  - 配置
  - 日志
  - 安全
  - 常量

---

## 3.5 前端技术选型

### 推荐决策

- **Streamlit**

---

## 4. 系统架构设计

## 4.1 总体架构

```text
[前端 Streamlit]
        |
        v
[FastAPI API Layer]
        |
        +--------------------+
        |                    |
        v                    v
[Retrieval Service]     [Report Service]
        |                    |
        v                    v
[SQLite + FTS5]         [Jinja Template Engine]
        |
        v
[Embedding Index / Local Vector Cache]
        |
        v
[Ollama: DeepSeek R1 8B + Embedding Model]

另一路：
[APScheduler]
   -> [Source Crawler]
   -> [Parser / Cleaner / Chunker]
   -> [SQLite + Embedding Refresh]
```

---

## 4.2 核心能力拆解

### A. 政策问答
输入：用户自然语言问题
输出：
- 结构化回答
- 引用政策来源
- 发布日期/地区/适用范围
- 不确定性提示

### B. 政策比对
输入：两个地区或两份政策
输出：
- 共同点
- 差异点
- 适用对象差异
- 时间差异

### C. 报告背景自动生成
输入：
- 报告类型（如碳盘查、节能改造、绿色工厂、ESG/双碳方案）
- 企业所在地区
- 行业
- 项目关键信息
输出：
- 背景综述
- 政策依据
- 项目必要性表述
- 可继续润色版本

### D. 知识库自动更新
输入：预设官方源列表
输出：
- 新增文档
- 更新文档
- 失效文档标记
- 同步日志与报错记录

---

## 5. RAG 具体实现方案

## 5.1 分块策略

建议不要只按固定 token 切块，而是采用：

1. **优先按文档结构分块**
   - 一级标题
   - 二级标题
   - 条/款/项
2. 再做长度控制
   - 单 chunk 建议 300～700 中文字
   - chunk overlap 50～100 中文字
3. 对政策类文档额外保留：
   - 文号
   - 颁布单位
   - 发布时间
   - 生效日期
   - 地区
   - 适用行业
   - 主题标签

### 为什么
政策问答非常依赖“条款上下文”和“适用边界”，纯 token 切块容易丢失法律/政策结构。

---

## 5.2 检索策略

### 推荐：Hybrid
- **FTS5**：解决政策名、术语、文号、专有名词精确命中
- **Embedding 检索**：解决语义表达不一致
- **规则重排**：强化以下字段权重：
  - 地区匹配
  - 时间新鲜度
  - 文档级别（国家 > 省 > 市县，可配置）
  - 文档状态（有效 > 废止/失效）

### 初版即可落地的排序公式

```text
final_score = 0.40 * keyword_score
            + 0.35 * semantic_score
            + 0.15 * region_score
            + 0.10 * freshness_score
```

---

## 5.3 回答生成策略

### 建议 prompt 原则
- 只允许基于检索到的内容回答
- 未检索到充分依据时必须明确说“不足以判断”
- 必须输出引用来源
- 对政策咨询类问题优先输出：
  1. 结论
  2. 依据
  3. 适用范围
  4. 风险提示

### 输出结构建议

```json
{
  "answer": "...",
  "citations": [
    {
      "title": "...",
      "source": "...",
      "publish_date": "...",
      "section": "..."
    }
  ],
  "confidence": "high|medium|low",
  "risk_note": "..."
}
```

---

## 6. 报告背景自动生成方案

## 6.1 适合的模板化场景

- 碳达峰/碳中和项目背景
- 节能降碳改造项目背景
- 绿色工厂申报背景
- 零碳园区/低碳园区背景
- 企业双碳实施方案背景
- ESG / 可持续发展报告中的政策背景段落

## 6.2 推荐做法

### 方式：模板 + 检索 + 大模型润色
1. 先根据报告类型选择模板
2. 检索相关政策依据
3. 用结构化 prompt 让模型填充
4. 再调用一次润色 prompt 输出正式表述



### 报告生成模块建议
- `templates/`
  - `project_background.jinja2`
  - `policy_basis.jinja2`
  - `necessity_analysis.jinja2`
- `report_service.py`
  - 模板装配
  - 证据抽取
  - 结构化生成
  - 风格润色

---

## 7. 自动更新知识库设计

## 7.1 同步机制

### 定时任务
- 每天凌晨执行一次轻量同步
- 每周执行一次全量校验
- 支持后台手动触发同步

### 同步流程
1. 读取 `knowledge_sources`
2. 抓取列表页
3. 抽取详情页链接
4. 判断 URL / hash / 发布时间是否变化
5. 新增或更新文档
6. 解析文本
7. 重新分块与 embedding
8. 更新检索索引
9. 写入日志

## 7.2 去重与版本控制

### 去重维度
- URL
- 标题
- 发布日期
- 正文 hash
- 文号（若存在）

### 版本策略
- 发现同 URL 正文变更：旧版本保留，生成新版本
- 标记：`active / superseded / invalid`
- 对问答默认只检索 `active`

---

## 8. API 设计建议

## 8.1 问答接口

### `POST /api/v1/chat/query`
请求：
```json
{
  "question": "江苏制造业企业有哪些节能降碳相关政策支持？",
  "region": "江苏省",
  "industry": "制造业"
}
```

响应：
```json
{
  "answer": "...",
  "citations": [...],
  "debug": {
    "retrieved_chunks": [...]
  }
}
```

## 8.2 报告背景生成接口

### `POST /api/v1/report/background`
请求：
```json
{
  "report_type": "节能降碳改造项目",
  "region": "浙江省",
  "industry": "纺织",
  "project_name": "高效节能设备替换项目",
  "project_summary": "..."
}
```

## 8.3 知识源管理接口
- `GET /api/v1/sources`
- `POST /api/v1/sources`
- `POST /api/v1/sources/{id}/sync`

## 8.4 文档管理接口
- `GET /api/v1/documents`
- `GET /api/v1/documents/{id}`
- `POST /api/v1/documents/upload`

---

## 9. 开发阶段规划

## 阶段 1：PoC

### 目标
验证“本地可跑 + 能回答 + 能引用 + 能生成报告背景”

### 范围
- FastAPI 后端
- SQLite 数据库
- Ollama 接入
- 10～50 份政策/标准文档导入
- FTS5 + embedding 混合检索
- Streamlit 简单界面
- 基础问答与背景生成

### 验收
- 能正确引用政策来源
- 单次问答可在可接受时延内完成
- 报告背景能生成可编辑文本

---

## 阶段 2：Beta

### 目标
从“能用”走向“可持续维护”

### 范围
- 自动同步官方站点
- 文档版本管理
- 更完善的 chunk 与元数据抽取
- 管理台
- 问答日志与反馈
- 基本权限控制

### 验收
- 可自动更新知识库
- 可查看同步日志
- 可追溯答案引用链路



---

## 10. 评测方案

## 10.1 建议建立最小评测集
至少 100 条问题，覆盖：
- 政策检索类
- 条文定位类
- 地区适配类
- 时间有效性类
- 报告背景生成类
- “知识库无答案”类

## 10.2 指标建议
- **Retrieval Recall@K**
- **引用准确率**
- **回答可用率**
- **拒答正确率**
- **平均响应时间**
- **报告背景人工评分**

## 10.3 初版评估方式
- 人工标注 + 日志回放
- 后续再接入 RAG 评测框架（如 Ragas）

---

## 11. 风险与规避

## 风险 1：8GB 显存导致响应慢或上下文溢出
**规避：**
- 限制上下文长度
- 缩小 TopK
- 控制 chunk 数
- 优先精检索而非堆上下文

## 风险 2：政策更新频繁导致答案过时
**规避：**
- 只采官方源
- 增量同步
- 输出发布日期/生效日期
- 默认优先最新有效版本

## 风险 3：扫描 PDF / 表格文档解析差
**规避：**

- 先走 PyMuPDF
- 对复杂文档再启用 unstructured
- 保留人工上传修正文档通道

## 风险 4：大模型“编”政策
**规避：**
- RAG 强约束
- 无证据不回答
- 输出引用
- 在系统提示中明确禁止无依据推断

---

## 12. 最终推荐技术栈（建议定稿）

## 12.1 正式推荐版

### 后端
- Python 3.11
- FastAPI
- SQLModel
- SQLite
- APScheduler
- Uvicorn
- Ollama Python / HTTP API

### 模型
- 生成：`deepseek-r1:8b`
- 向量：`bge-m3`（推荐）或 `embeddinggemma`（低资源兜底）

### 检索
- SQLite FTS5
- Python 侧语义检索
- 规则重排

### 解析
- requests
- BeautifulSoup4
- PyMuPDF
- unstructured（按需）
- markitdown（可选）

### 前端
- streamlit

### 模板
- Jinja2

---

## 13. 一句话结论

这是一个**适合采用“轻量本地 RAG + 模板化报告生成”路线**的项目。

在 **RTX 4060 8GB + Ollama + DeepSeek R1 8B** 的约束下，**最稳妥的方案**不是追求复杂 Agent，而是：

> **FastAPI + SQLite + FTS5 + 本地 embedding + DeepSeek R1 8B + 官方政策源自动更新 +Streamlit 前端**

这样能在成本、部署难度、隐私性、可维护性之间取得最佳平衡。
