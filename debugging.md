# Debugging — bug 定位 / 排查方法

## 2026-05-16 — 静态方法当裸名调用导致 NameError，错误被吞

**类型**: 问题

现象：`generate_train_split_sft_data.py` 所有 `*_tool_call.jsonl` 输入产出 0 条，无报错。
根因：`_predict_tool` 内把静态方法当裸名调用 `_parse_tool_calls_text(...)`，触发
`NameError`；外层 `_process_item` 的 catch-all 把异常静默吞掉，表现为"无输出"。
解决：改为 `self._parse_tool_calls_text(...)`。Python LEGB：静态/类方法在实例方法
内必须用 `self.`/类名 限定，不能裸名访问。

---

## 2026-05-16 — `r.get('a' or r.get('b'))` 短路陷阱

**类型**: 问题

`content = (r.get('tool_call_final' or r.get('tool_call_new')) or '')` 永远只读
`tool_call_final`。`'tool_call_final'` 是真值字符串，`or` 直接短路返回它，
`r.get('tool_call_new')` 从不执行。正确写法：
`r.get('tool_call_final') or r.get('tool_call_new') or ''`。

---

## 2026-05-16 — GPU 利用率 0% / 训练卡死定位

**类型**: 经验

RL 训练无输出、GPU 0%、又没有 error 时，用 `py-spy dump --pid <pid>` 打印每个
rank 的 Python 栈，能直接看到卡在哪个调用（如等锁、等 RPC、等 rollout health）。
比盲猜日志高效。结论：先 `py-spy dump` 看栈，再决定方向。

---

## 2026-05-16 — 训练时怀疑加锁，先回滚验证

**类型**: 经验

tool_client 多线程调用加锁后卡死。结论：之前不加锁跑得很好就先去掉锁验证，
不要默认"加锁更安全"而过度防御——锁本身可能是死锁源。改动前先确认问题确实
由竞争引起。

---

## 2026-04-24 — `local variable 'exact_fuzzy' referenced before assignment`

**类型**: 问题

`get_exact_fuzzy` 抛异常时 `exact_fuzzy` 未赋值，后续引用即报此错。
解决：异常分支里 `result.setdefault('intent', 'fuzzy')`，给变量兜底默认值。
通用经验：任何"可能抛异常后又被引用"的变量，在 try 前或 except 里给默认值。

---

## 2026-05-16 — 模块从错误路径 import

**类型**: 问题

`cannot import name 'UserFeatureClient' from 'src.user_feature.get_user_feature'`
（路径指向 `/opt/tiger/LLM_training3/tako_ls/...`）。根因：线上训练时 `tako_ls`
被同步到 `LLM_training3`，与本地 `/root/yingmeiguo/tako_ls` 版本不一致，旧版本
没有该类。排查 import 错误先确认 `module.__file__` 实际指向哪个副本。

---

## 2026-05-12 — Ray ActorDiedError 真因在 actor stderr

**类型**: 经验

driver 端看到的 `ActorDiedError` 只是"actor 死了"的传播错误，被截断处不是
根因。真因在那个 actor 自己的 worker log：从报错里取 `actor_id` / `pid` /
节点 ip，去 Ray dashboard → Actors → 该 actor → stderr，或在对应节点
`find /tmp/ray -name "worker-*<pid 后缀>.err"`。排 Ray 故障别只看 driver log。

## 2026-05-11 — bytedray async actor strict check

**类型**: 知识

bytedray 2.10.0.74 在 `ray.remote(Worker)` 时检查到基类有 `async def` 方法
就标记为 async actor，但若执行方法是 sync 就矛盾报错。verl 上游在原生 ray
2.10.x 无此问题，是 bytedray 加的检查。排查：
`grep -n "async def" verl/single_controller/base/worker.py`。

## 2026-05-15 — IPv6 URL 必须方括号 + FastAPI 尾斜杠 301

**类型**: 问题

`http://fdbd:...::8001/health/` 一直 curl 失败两个独立坑：① IPv6 地址在 URL
里必须方括号包裹 `http://[fdbd:...::]:8001/`，否则解析失败；② FastAPI 把
`/health/` 301 重定向到 `/health`，`curl -sf` 默认不跟随重定向返回非零 →
误判服务没起。健康检查 URL 去掉尾斜杠或 curl 加 `-L`。

## 2026-05-15 — vLLM engine subprocess EOFError 真因

**类型**: 经验

`rollout.py EOFError`（multiprocessing pipe）/ starlette traceback 都是
**后果**，真实 crash 在 vLLM worker 子进程，且日志在 EOFError **之前**或
独立 process stream。排查往上翻找 `CUDA out of memory` / 模型加载失败，
不要盯着 EOFError 本身。
