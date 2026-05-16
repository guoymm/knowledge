# Pipeline — 问答卡精修 pipeline / 数据生成

## 2026-05-16 — 不拒答原则

**类型**: 知识

pipeline 不因非生服意图早退。`predict_ls_intent` 仅设 `is_ls` 标志并把
`rejected` 重置为 False，所有 query 都走完整精修。只有两种情况真正拒答提前
返回：LS refine 返回 `poi_unavailable`；General refine 返回 `insufficient_info`。

---

## 2026-05-16 — is_ls 路由用同一 client 换 prompt

**类型**: 经验

按 is_ls 走两条精修路径，不需要两个 client，只是 `LsCardRefineGenerator`
内部按 `is_ls` 选 `LS_CARD_REFINE_PROMPT_LONG` 或 `GENERAL_CARD_REFINE_PROMPT`。
结论：路由差异能用 prompt 切换解决就不要复制类，减少维护面。

---

## 2026-04-16 — tool_call_final vs tool_call_new

**类型**: 知识

`tool_call_new`：ToolGenerator 重新生成的 tool call（格式化字符串）。
`tool_call_final`：最终采用、写入 SFT target 的那一个。非生服或不开
tool_call_judge 时 `tool_call_final = tool_call_new`；开 judge 时取
Planning rubric 评分通过的那一轮。SFT `_to_sft_record` 读 `tool_call_final`。

---

## 2026-04-16 — add_ref_poi：多 tool_response 堆叠时引用全局递增

**类型**: 知识

多个 `<tool_response></tool_response>` 堆叠时，`[^N]` 编号必须跨块全局连续，
不是每块从 1 重数。所有 ref 元素（不只 tiktok_places）统一递增计数，
tiktok_places 只是其中一部分。`ref_offset` 从 0 开始才能让首条是 `[^1]`。

---

## 2026-05-16 — RL 中 is_ls 判定

**类型**: 知识

RL plugin 里 `_detect_is_ls`：看模型生成的 tool_call 是否含 `tiktok_places`。
期望模型调 `combine_search` 且 `sources` 覆盖 web/tiktok/lemon8/tiktok_places
四项；只有三项（缺 tiktok_places）视为非生服。reward 按命中工具/源比例算。

---

## 2026-04-16 — SFT 脚本一一对应 & 复用基类

**类型**: 经验

多个数据生成脚本（hdfs_data / tako_data / us_data）逻辑高度重合：`add_ref_poi`、
`_build_item`、`_get_system_prompt`、四文件输出（rejected/success/tool_call/
debug）全部下沉到 `SFTGeneratorBase`，子类只重写差异部分。输入输出文件名严格
一一对应。结论：新增数据源先看 SFTGeneratorBase 能否复用，不要整段复制。

---

## 2026-04-24 — 测试时不要 resume，每次重跑

**类型**: 经验

评估/pipeline 测试反复要求"不要 resume，每次都重跑"+ `set_cache=False
get_cache=False`。原因：resume / 旧缓存会读到上一版 prompt 的脏结果，掩盖改动
效果。结论：迭代 prompt 时必须清缓存全量重跑才能对比真实改进。

---

## 2026-05-16 — 训练数据 `<tools>` 内容来源

**类型**: 知识

SFT system prompt 的 `<tools>...</tools>` 来自
`SFTGeneratorBase.__init__` 里 `tool_client.get_all_tools()` 填入
`TRAINING_SYSTEM_PROMPT` 的 `{{tool_descs}}`。`prepare_training_data.py` 只是
原样转抄源文件 messages，不重建 system prompt——所以训练数据里的工具集取决于
当初生成源数据时的工具环境，与推理时 `model.py` 注入的工具集可能不一致。

---
