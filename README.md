# China IPO Inquiry Corpus · 沪深 IPO 问询洞察库（2023–2026）

> **China IPO Inquiry Corpus** 是基于沪深交易所公开 IPO 审核问询材料构建的结构化派生语料库，提供监管问题、AI 案例摘要、回复结构标注和监管问题 taxonomy；不镜像完整原始披露文件，所有 AI 派生内容均可回溯至交易所公开来源。
>
> A structured derivative corpus built from public IPO review inquiry disclosures of the Shanghai and Shenzhen Stock Exchanges (2023-01 to 2026-09): regulatory questions, AI case summaries, reply-structure annotations, and a regulatory-question taxonomy. Full original disclosure files are **not** mirrored; every AI-derived record links back to the exchange-published source PDF.

**关键词 / Keywords**：沪深交易所 IPO 审核问询函、问询回复、监管问询、案例摘要、回复结构、监管问题地图、问题分类 taxonomy、收入确认、验收周期、发出商品、毛利率差异、客户集中度、关联交易、资金流水、存货跌价、募投项目；China IPO inquiry letters, SSE SZSE stock exchange IPO review questionnaires, regulatory question taxonomy, IPO review Q&A dataset, inquiry response summaries, structured derivative corpus, LLM-ready JSONL, prospectus review, exchange comment letters.

## 设计出发点：为什么不是又一个 RAG 语料包

把问询回复全文切块做向量检索的传统 RAG 知识库，在这类长文档上切片极易切断论证链（一份回复动辄三五百页、一问一答数万字），召回的碎片既答非所问又吞上下文。本库换了一条路：**以"审核问题"为最小单元做精度总结**——每个案例只保留监管问题原文、一段 80–180 字的高压缩摘要、一条回复论证结构链和主题归属。Agent 用几百字符的输入就能判断"这个历史案例是不是用户要找的问题"，命中后再经交易所直链回原文精读。**检索的是问题，不是答案碎片**；小规模输入、高命中精度、全程可溯源，是本库相对全文 RAG 的三个取舍。

## 本仓库收录了什么

覆盖 **2023-01-01 至 2026-09-30 受理**的沪深 IPO 项目（上交所科创板/沪主板、深交所创业板/深主板）审核问询回复家族文件（问询函回复、多轮回复、落实函回复、上市委审议意见落实函回复），共 2,299+ 份披露文件、816 个项目。在原文之上做了四层加工：

| 加工层 | 内容 | 规模（v1.0.1） |
|---|---|---|
| 问题级拆分 | 每份回复切分为"监管问题—对应回复"案例，保留问题原文、题号、页码范围 | 20,509 个案例（公开层 15,224 条） |
| 一句话案例摘要 | 80–180 汉字：监管关注什么 + 发行人实际从哪些角度、用什么证据解释 | 15,224 条 |
| 回复结构标注 | 2–6 个动作短语组成的论证链（如：列示数据→拆解变动原因→补充在手订单→说明合作持续性） | 同上 |
| 监管问题地图 | 自下而上归纳的主题层级：18 个一级 / 111 个二级，10,574 条唯一问题全量映射（未映射率 0.31%） | 1 套 taxonomy |

每条案例同时携带**溯源三件套**：交易所官网原始 PDF 直链（`source_url`，覆盖率 100%）、规范引用串（`citation`：项目、文件标题、披露日期、PDF 页码）、库内相对路径（`source_rel`）。

### 关键文件

| 文件 | 用途 |
|---|---|
| `data/cases/SSE_2023.jsonl … SZSE_2026.jsonl` | 案例主数据（8 个分片，按交易所×披露年份） |
| `data/manifest.json` | 数据集版本、快照日、分片计数与 SHA256（引用请钉版本） |
| `data/regulatory_question_map.md` / `.json` | 监管问题地图（人类阅读版 / 程序版，含每节点全部 case_id） |
| `data/taxonomy_v1.json` | 主题树 + 归纳过程别名映射（可审计） |
| `data/question_topic_map.jsonl` | 问题 → 主题路径映射 |
| `data/stats.json` | 回复结构动作短语频次、分布统计 |
| `AGENTS.md` | **AI Agent 接入指南**（字段语义、检索工作流、信任依据） |
| `CHANGELOG.md` | 数据集更新记录 |

## 本仓库能做什么、不能做什么

**能做**：

- 用关键词/语义检索**监管问过什么**——按业务情形词（发出商品、验收周期、让步接收、同型号不同售价、第三方回款、首台套……）在 `question_text` / `question_title` / `case_summary` 中定位历史案例；
- 沿**问题地图**逐层下钻：一级主题 → 二级问法 → 具体案例 → 交易所原始 PDF（`source_url` 一键核实）；
- 研究**回复的论证结构**：reply_structure 动作链可直接做模式统计（`data/stats.json` 已预聚合）；
- 让 AI Agent 做以上全部事情（读 `AGENTS.md` 接入，字段语义齐全、无需猜）。

**不能做（重要边界）**：

- **不适合检索"答案原文"**。回复正文经过压缩摘要，细节数字、表格、核查程序原文不在公开层；搜关键词命中摘要不等于命中回复全文。需要答案细节时，请用案例的 `source_url` 回交易所原文核对——这正是本库的设计意图：帮你**找到**该读哪份原文，而不是替代原文；
- 不构成对任何回复质量、过会结果的评价，不构成投资建议；
- 不镜像原文全文，不能替代交易所官网披露渠道。

## 为什么值得信任

1. **来源可核实**：每条案例带交易所官网 PDF 直链与页码范围，摘要对错可一键对原文裁决；
2. **问题原文未改写**：`question_text` 为披露文件逐字摘录，构建过程做过哈希防篡改校验；
3. **质量如实披露**：`quality_flags` 白名单标记五类已知局限（标题级案例、材料缺失、数字溯源待查、整文档过度捕获、字数越界），使用者可主动排除；QC 标记率 5.0%（非"错误率"），LLM 批处理最终失败 0；
4. **版本可钉**：`data/manifest.json` 记录 dataset_version、语料快照日与分片 SHA256，引用可精确复现；
5. **方法可概述**：解析切分→章节信号句压缩→分批生成与校验→抽样归纳主题→全量映射的管线要点与质量数据见下文"数据版本与更新"及 AGENTS.md 字段说明；完整构建记录由维护方留存备查。

## 数据版本与更新

- 当前：**v1.0.1**（语料快照 2026-09-30，15,224 条案例）；
- **每周定期更新**：跟随交易所增量披露，维护方本地跑完整管线（增量解析→摘要→映射→导出→审计）后发布新版本；
- 每个版本以 **GitHub Release + tag** 发布，变更明细见 [CHANGELOG.md](CHANGELOG.md)，分片校验和见 `data/manifest.json`；
- 引用或二次加工时请钉住版本号，例如 `China IPO Inquiry Corpus v1.0.1`。

## 许可与第三方文本边界

- 本项目原创派生内容（案例摘要、回复结构标注、taxonomy、数据组织与统计）：CC BY-NC-SA 4.0（署名—非商业—相同方式共享），见 [LICENSES/CC-BY-NC-SA-4.0.txt](LICENSES/CC-BY-NC-SA-4.0.txt)；
- **监管问题原文及其他来源文本摘录**（`question_text`、代表问法引文等）来源于沪深交易所公开披露文件，其相关权利归原权利主体所有，不因收录于本项目而由本项目重新授权。

## 免责声明

语料来自沪深交易所官网公开披露文件；AI 摘要与主题分类可能存在误差；案例中的人名、机构名均出自公开披露原文。本项目为个人研究用途、非商业，与任何交易所、监管机构及作者雇主无关联。

---
© Verlaski
