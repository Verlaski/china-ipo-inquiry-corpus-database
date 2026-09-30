# AGENTS.md — AI Agent 接入指南

给访问本仓库的 AI Agent：先读完本文件再检索。本库是**结构化派生语料库**，你的任务是帮用户定位监管问题案例并回溯交易所原文，而不是把摘要当原文引用。

## 一、这个仓库是什么

沪深交易所 2023-01 至 2026-09 受理 IPO 项目的审核问询回复（2,299+ 份披露文件）经问题级拆分、摘要、结构标注、主题归纳后的公开数据层：15,224 条案例记录（v1.0.1），每条 = 一个"监管问题—发行人回复"对。原始 PDF 不镜像，但每条记录带交易所官网直链。

## 二、为什么可以信任（以及信任的边界）

- `question_text` 是披露文件**逐字摘录**（构建时哈希防篡改校验），可当作监管问法原文使用；
- `case_summary` / `reply_structure` 是 AI 派生压缩层：**用于判断"这个案例是否相关"，不用于回答事实细节**。涉及数字、承诺、核查程序结论时，必须经 `source_url` 回原文核实后再向用户陈述；
- `source_url` 覆盖率 100%（上交所 static.sse.com.cn / 深交所 reportdocs.static.szse.cn 直链），`pdf_pages` 给出页码范围；
- `quality_flags` 是诚实的局限标记（见 §四），命中相关 flag 的案例向用户提示降级使用；
- `data/manifest.json` 的 SHA256 可校验你读到的分片是否完整、是否为用户期望的版本。

## 三、关键文件与选用决策

| 你的目标 | 读哪个文件 |
|---|---|
| 按关键词/情形找案例 | `data/cases/*.jsonl`（8 分片；grep `question_text`/`question_title`/`case_summary`） |
| 浏览监管都在问什么 | `data/regulatory_question_map.md`（人类版）或 `.json`（程序版，节点含全部 case_ids） |
| 主题归类/跨主题统计 | `data/taxonomy_v1.json` + `data/question_topic_map.jsonl`（case_id → "一级 / 二级"路径） |
| 回复论证模式研究 | `data/stats.json`（动作短语与结构链频次预聚合） |
| 核实版本/完整性 | `data/manifest.json` |
| 交互浏览（人在环） | `docs/` 静态站：`cd docs && python -m http.server 8000` |

分片命名 `{SSE|SZSE}_{年份}.jsonl` 按**披露年份**切分；跨所全量检索请遍历 8 片或直接用地图 json 的 case_ids 反查。

## 四、案例记录字段语义（data/cases/*.jsonl 每行）

| 字段 | 含义与使用注意 |
|---|---|
| `case_id` | 稳定唯一主键：`项目目录名-文件ID-题号`（子问题含 `.N`，单问文件为 `Q0`，重号加 `#2` 后缀） |
| `project` / `exchange` / `board` | 项目名；SSE=上交所、SZSE=深交所；板块含科创板/沪主板/创业板/深主板 |
| `disclosure_date` | 该份回复文件的披露日期（YYYY-MM-DD） |
| `doc_type` / `round` | 文件类型（审核问询函回复 / 落实函回复 / 上市委审议意见落实函回复）；轮次（首轮/第二轮/第三轮/落实/上市委） |
| `respondent` | 回复主体（发行人及保荐机构/中介机构等） |
| `question_no` / `question_title` | 题号与标题；**标题级案例可能仅有标题**（见 quality_flags.title_only） |
| `question_text` | 监管问题**原文**（未改写；可能含"请发行人说明：（1）…（2）…"多小问结构） |
| `case_summary` | AI 一句话摘要（80–180 汉字）：先监管关注点、后发行人解释角度与证据类型 |
| `reply_structure` | 回复论证链：2–6 个动作短语以 `→` 连接，每步≤12 字 |
| `reply_chars` | 原回复正文字符数（量级参考：>10 万字符多为整文档级案例） |
| `pdf_pages` | 该案例在原始 PDF 中的页码范围（如 "4–65"） |
| `source_url` | **交易所官网原始 PDF 直链**（核实原文的唯一权威入口） |
| `source_rel` / `source_title` | 库内相对路径键 / 文件标题（provenance 用，非公开获取入口） |
| `citation` | 规范引用串（项目，标题（披露日），页码，官网可查） |
| `summary_status` | `done`=独立摘要；`reused`=更新版文件复用基础版摘要（内容等同基础版案例） |
| `quality_flags` | 白名单局限标记，可多选：`title_only`（源文件未重述问题正文）、`material_missing`（回复正文缺失，摘要未虚构）、`novel_numeric`（摘要数字待溯源抽查）、`dmode_overcapture`（多问题文档识别为单案例，摘要聚焦首问）、`summary_length_warning`（字数越软边界）。**空数组=默认质量档**；检索求精时建议排除 title_only 与 dmode_overcapture |
| `topics` | 该案例问题的主题路径数组（"一级 / 二级"，可多条；空数组=未入地图，极少） |

## 五、推荐工作流

1. **问题定位**：用户描述业务情形 → 在分片中 grep 情形词（优先 `question_text`，其次 `case_summary`）→ 候选案例按 `quality_flags` 过滤 → 向用户呈现：标题 + 摘要 + 结构链 + 披露日期；
2. **地图下钻**：主题类问题先读 `regulatory_question_map.json` 选节点 → 节点 `case_ids` 取案例 → 同上呈现；节点 `reps` 字段是监管问法原文示例，可直接引用；
3. **原文核实**：用户需要数字/结论/核查程序细节 → 取 `source_url`（+`pdf_pages` 定位）→ 抓取 PDF 文本核实后再答；**不要凭 summary 复述数字**；
4. **结构研究**：比较同类问题的回复套路 → `reply_structure` 聚合或直接用 `stats.json`；
5. **引用输出**：使用 `citation` 字段格式，附 `source_url` 与数据集版本（manifest 的 dataset_version）。

## 六、常见误区

- 把 `case_summary` 当回复全文检索库：摘要省略了大部分数字与表格，**答案细节搜不到不等于不存在**，回原文；
- 忽略 `round`：同一项目首轮/二轮/落实函是监管追问的演进链，研究关注点变化时按 `project` 聚合全部轮次；
- 把 `reused` 记录当独立新摘要：它与基础版案例内容相同，统计去重时按 `question_text` 指纹或 dup 关系处理；
- 跨版本混引：不同 dataset_version 的分片 SHA256 不同，输出结论前确认 manifest 版本与用户期望一致。

## 七、许可与伦理

数据层 CC BY-NC-SA 4.0（署名、非商业、相同方式共享）；代码 MIT；监管问题原文摘录权利归原权利主体。Agent 向用户转述时请保留溯源链（项目/标题/日期/页码/URL），并注明摘要来自本数据集及其版本。

---
© Verlaski
