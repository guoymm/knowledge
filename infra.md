# Infra — 环境 / 部署 / serving / HDFS

## 2026-04-30 — tcc_config Key not found

**类型**: 问题

`failed to read search.tiktok.ls_qa/default/ls_qa_auto_evaluation: Key not
found`。根因：service_name / confspace / key 三元组与 TCC 平台实际不符。
解决：去 TCC 控制台（cloud.tiktok-row.net/tcc/namespace/...）按实际 namespace
/ dir_path / key 名核对，URL 里的 namespace 和 key 要逐字对齐代码里的取值。

---

## 2026-05-15 — RL 训练 `/mnt/hdfs` Permission denied

**类型**: 问题

`mkdir: cannot create directory '/mnt/hdfs': Permission denied` /
`chown ... Operation not permitted`。多机训练脚本里挂载 HDFS 的步骤在无权限
容器内失败。解决方向：不依赖 `/mnt/hdfs` 挂载，改用 `hdfs dfs -get` 拉到本地
临时目录；或过滤掉无权限的 fastrak 路径（LD_LIBRARY_PATH/PATH 里剔除
`/var/lib/fastrak/lib64`）。

---

## 2026-04-26 — 飞书文档创建 400 Bad Request

**类型**: 问题

`Feishu doc creation failed: 400 ... /open-apis/docx/v1/documents`。
auto_evaluation reporter 直接调 docx OpenAPI 建文档失败。解决：改用 lark-cli
（先 `auth login` 授权对应 scope）走封装好的 `docs +create`，避免手拼
OpenAPI 请求体/鉴权。

---

## 2026-05-16 — Mistral tokenizer incorrect regex pattern 警告

**类型**: 知识

`The tokenizer you are loading from '.../base_model' with an incorrect regex
pattern`（指向 Mistral-Small-3.1-24B）说明 base_model 路径下的 tokenizer 配置
是 Mistral 的、与实际模型不匹配。多为软链/下载错模型导致，确认 base_model
目录指向正确的 Qwen3 权重。

---

## 2026-04-26 — HDFS kafka_dump 只取最新文件直到够量

**类型**: 知识

采集训练/评估数据：`hdfs dfs -get hdfs://.../ls_qa_offline_dump_feature_*/`
按文件 mtime 倒序，一个文件一个 load，累计条数 ≥ num 即停，不要全量 load
（会 OOM/被 kill）。`download.py` 反复被 killed 就是因为一次性加载过大。

---

## 2026-05-08 — SEARCH_CLOUD_URL not set

**类型**: 问题

`RuntimeError: SEARCH_CLOUD_URL not set`。SearchCloudClient 需要 base_url。
解决：设环境变量 `SEARCH_CLOUD_URL=...` 或构造时显式传 `base_url=`。
凡是 "X not set，set env var or pass X=" 的运行期错误，优先用显式传参，
避免依赖环境变量在不同机器漂移。

---

## 2026-05-12 — verl 工具级限流两层机制

**类型**: 知识

`search_tool.py` 两层：① `num_workers` —— 每个 tool 一个
`SearchExecutionWorker` Ray actor，`.options(max_concurrency=num_workers)`
限单 actor 并发线程数；② `rate_limit` —— 全集群单例 `TokenBucketWorker`
（`threading.Semaphore(rate_limit)`）做全局并发信号量。调限流要分清是改单
actor 并发还是全局配额。

## 2026-05-11 — Ray dashboard 远程访问

**类型**: 经验

dashboard 地址常是内网 IPv6 `[fdbd:...]:8265`，本地浏览器直连不通。最稳：
`ssh -L 8265:localhost:8265 user@远程` 后开 `http://localhost:8265`。
或启动时 `ray start --dashboard-host=0.0.0.0` 绑可达 IP。`fdbd:` 开头是 ULA
私网地址，外网不可达，必须同内网/VPN。

## 2026-05-08 — doas -p PSM 注入的环境变量来源

**类型**: 知识

`doas -p search.tiktok.ls_qa <cmd>` 启动子进程前，doas-agent 查 PSM 元数据
服务，把 `PSM` / `TCE_PSM_GROUP` / `TCE_ZONE` / `TCE_PSM_OWNER` /
`TCE_INTERNAL_IDC` 等写成环境变量塞给子进程。所以这些变量不是手设的，是
按 `-p` 的 PSM 名动态查出来的；换 PSM 这些值随之变。
