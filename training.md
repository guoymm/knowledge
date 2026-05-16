# Training — 模型训练 / SFT / 推理

## 2026-05-16 — 训练/推理 thinking 模式不匹配导致输出乱码

**类型**: 问题

现象：SFT 后模型推理输出乱码（`he\n\n0.0.0.00...` 之类）。
根因链：① 训练用 `--add_non_thinking_prefix true`（Qwen3 训练时禁用 thinking），
推理侧未关 thinking，模型吐 `<think>...</think>` 原文且未被 strip；
② 更深层是 chat template / serving 配置与训练不一致。
解决：推理参数加 `enable_thinking=False`，并在客户端用
`re.sub(r'<think>.*?</think>\s*', '', raw, flags=re.DOTALL)` 兜底 strip。
排查顺序：先离线 vLLM + `apply_chat_template(enable_thinking=False)` 验证模型
本身是否正常，再查 PSM serving 的 chat_template 配置。

---

## 2026-05-16 — `--add_non_thinking_prefix true` 的机制

**类型**: 知识

ms-swift 对 Qwen3 thinking 模型，该 flag 在 token 层面给每个 assistant turn 加
空 think block（`<think>\n\n</think>`），训练成非思考模式。它不写在 messages
文本里——所以训练数据 jsonl 里看不到 think token。推理时必须对应地在生成参数
层关闭 thinking，而不是改 system prompt 文本。

---

## 2026-05-16 — `--loss_scale last_round+ignore_empty_think`

**类型**: 知识

SFT 只对最后一个 assistant turn 计算 loss，前面的 tool_call / tool_response
轮不计 loss。多轮工具调用数据据此构造：中间轮是上下文，只有最终回答（或拒答
tag / 最终 tool_call）是训练 target。

---

## 2026-05-15 — GRPO 最简数据格式

**类型**: 知识

复用 tako_ls rubric 打分做 GRPO 时，训练数据只需 `query` + `rubric`，不需要
参考答案。reward 在 rollout 后由 scorer（Phase1 comprehensiveness + Phase2
general）实时算。RL plugin 里 tool_response 要按最新日期重新取，system prompt
里的日期字段也要随之变；`get_user_feature` 的 tool_call/tool_response 固定写
死在 messages 中。

---

## 2026-05-15 — swift rollout 动态权重 & steps_per_generation

**类型**: 知识

ms-swift Megatron GRPO：`swift rollout --model <base>` 起的 rollout server 会
在训练中动态更新权重（`POST /update_flattened_params/`），不是固定 base。
`steps_per_generation=1` 表示每次 rollout 后立即用新权重，即 on-policy 程度
最高。`agent_template: hermes` 指 rollout 时用 Hermes 风格的 tool-call 模板。

---

## 2026-05-16 — swift `max_model_len - num_tokens < max_tokens` 警告

**类型**: 知识

含义：输入 prompt 占了 `num_tokens`，`max_model_len`（如 24576）减去它后
剩余空间 < 配置的 `max_tokens`（生成上限），swift 自动下调本次 `max_tokens`。
`num_tokens` 是 rollout 前的完整输入长度。频繁出现说明输入过长，应缩短 prompt
或调大 max_model_len。

---
