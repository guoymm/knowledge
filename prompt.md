# Prompt — prompt 工程 / rubric 设计

## 2026-05-16 — 改 prompt 必须查上下游一致性

**类型**: 经验

任何 prompt（生成 / refine / critique）改动，都要顺着依赖图检查上下游：
`REFINE_COMMON_CONSTRAINTS` 是共享约束层，注入所有 refine/card-refine prompt；
改它会影响全部路径。结论：用 prompt-modify skill 走依赖图，保证精简、不矛盾、
不冗余；新约束优先放共享层，单路径的删冗余后再加。

---

## 2026-05-16 — 输出语言应匹配 query 而非原回复

**类型**: 经验

原规则 "same language as original response" 错误：query 是 Tagalog 但原回复
是别的语言时，refine 会沿用错的语言。正确："match query language"。
结论：语言对齐基准永远是用户 query，不是上一版模型输出。

---

## 2026-05-16 — 平台引用：动作指引 OK，来源标注禁止

**类型**: 知识

"on TikTok / search on YouTube / watch on TikTok"（告诉用户去哪找）是允许的
动作指引；"according to TikTok Places / on TikTok Places this POI has..."
（当作信息来源标注）才违规。rubric 描述里要显式区分这两类，否则会过度扣分。

---

## 2026-05-16 — 添加新评估标准的修改位置

**类型**: 知识

| 标准类型 | 文件 | 变量 |
|---|---|---|
| 全局（LS+非LS） | rubric_prompt_v3.py | GLOBAL_PREDEFINED_RUBRICS_V3 |
| 仅生服 | rubric_prompt_v3.py | LS_ONLY_RUBRICS_V3 |
| 仅非生服 | rubric_prompt_v3.py | GENERAL_ONLY_RUBRICS_V3 |
| 生服完整性 | rubric_prompt_v3.py | COMPLETENESS_RUBRICS_LS |
| 非生服完整性 | rubric_prompt_v3.py | COMPLETENESS_RUBRICS_GENERAL |
| 初始生成约束 | general_prompt.py | REFINE_COMMON_CONSTRAINTS |

Phase1 失败触发补全 refine，Phase2 失败触发修复 refine；需指导修复时还要改
rubric_prompt_v2.py 的 COMPLETENESS_REFINE_PROMPT / REFINE_PROMPT_ALL。

---

## 2026-05-16 — POI 必含字段以交通方式为准

**类型**: 知识

精搜/泛搜的 POI 必含字段统一用"交通方式（transport）"，不再要求原始地址。
改标准时 LS_CARD_REFINE_PROMPT_LONG、COMPLETENESS_RUBRICS_LS 等需同步。

---

## 2026-05-08 — DCG rubric 评分改 0-3 严重度

**类型**: 知识

自动评估 rubric 把 severity 从 `["minor","serious"]` 改为 0-3 评分，每个
RubricDef 内写明打分标准。Card-Relevance 特例：地点精搜召回不相关结果直接
DCG=0；泛搜/品牌精搜一个不相关 POI 最高 1 分，≥2 个不相关 POI = 0。

---
