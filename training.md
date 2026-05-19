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

## 2026-05-11 — apply_chat_template 的 tools 渲染由 chat_template 决定

**类型**: 知识

`apply_chat_template(messages, tools=[...])` 把 tools 转 JSON schema 注入 Jinja
渲染上下文，**渲染位置完全由 tokenizer 的 `chat_template` 模板字符串决定**，
不是 transformers 硬编码。模板里有 `{% for tool in tools %}` 才渲染，没写就
忽略。排查训练/推理格式不一致先 `print(tok.chat_template)` 对比两侧模板。
（这正是 SFT 后推理乱码的深层排查点。）

## 2026-05-11 — verl 计数单位是 prompt（query），不是 query×n

**类型**: 知识

verl 全栈以 prompt（query 条数）为基本单位，rollout 数 `n` 是独立维度。
`train_prompt_mini_bsz=48` + `rollout.n=8` → 实际 384 条 trajectory；
`concurrent_per_server=32` 是 32 个 query 在跑，in-flight sequence 是 32×n。
算显存/生成量记得乘 `actor_rollout_ref.rollout.n`。

## 2026-05-11 — verl step 单位：rollout step vs trainer step

**类型**: 知识

`rollout.total_rollout_steps` 按 **rollout step** 计（rollout 端产出多少 step
才停，异步训练 rollout/train 解耦故用它做终止信号）；`trainer.save_freq` /
`test_freq` 按 **trainer step（actor update step）** 计。两者单位不同，跑短
实验调小 total_rollout_steps。

## 2026-05-11 — verl SFT 数据准备

**类型**: 知识

verl SFT 默认读 parquet（一行一样本），推荐 messages 格式（覆盖多轮/tool
call）：`{"messages":[...]}`，assistant 的 tool_calls 单独字段。loss mask 由
chat_template 的 generation 标记 + dataset 配置决定，只在 assistant 段算 loss。
比 RL 数据准备简单：一份 parquet + 一段 chat_template 渲染。

---

## 2026-05-11 — 多轮 prefix 只训 last round 的危害

**类型**: 知识

tool_call+response 数据合训、只对最后一轮算 loss，能跑通但有害：
① Exposure bias 放大——训练时 prefix 全是 gold tool_call/tool_response，推理时
prefix 是模型自己生成的（可能格式/参数错），模型没见过"脏 prefix 还能正确收尾"，
表现为中间 tool_call 出错后末轮答崩、多调几次工具就崩；② 中间轮决策不学——
tool_call 样本只学"发起调用"不学"看到返回该干嘛"，response 样本只学"看返回给
答"不学"何时停止再调"。缓解需让中间轮也进 loss 或加 self-generated prefix 数据。

## 2026-05-11 — Qwen3 chat_template 结构

**类型**: 知识

Qwen2.5/Qwen3 标准模板三段：① system 块（`{%- if tools %}` 时把用户 system
内容 + `# Tools` + `<tools>` 循环拼进去）；② messages 循环渲染
user/assistant(+tool_calls)/tool；③ `add_generation_prompt` 时追加
`<|im_start|>assistant\n`。tools **只在模板有 tools 分支时**才进 system——
训练/推理 tools 不一致会改变整个 system 块，是格式错位根源。
