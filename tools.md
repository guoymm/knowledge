# Tools — 工具用法（lark-cli / git / CLI / Claude Code）

## 2026-05-16 — 跨 session 总结靠 transcript 文件

**类型**: 知识

Claude Code 每个 session 的完整对话落盘在
`/root/.claude/projects/-/<session-id>.jsonl`，每行一条消息。模型本身看不到
其他 session 上下文，但可扫描这些 jsonl 实现跨 session 归档。按文件 mtime
筛日期，jsonl 太大时先 grep 报错/关键词定位行号再读片段。

---

## 2026-05-16 — lark-cli docs 操作要点

**类型**: 经验

`docs +fetch --doc <url>`（必须带 `--doc`，URL 或 token 都行）；更新优先用
局部模式（`insert_before/after`、`replace_range` + `--selection-by-title`
或 `--selection-with-ellipsis "开头...结尾"`），慎用 `overwrite`（会清空重写，
丢图片/评论）。wiki 链接要先 `wiki spaces get_node` 取真实 obj_token。
表格/画板等 token 内容无法读出原样写回，定位时避开。

---

## 2026-04-26 — lark-cli 认证

**类型**: 经验

`--as user` 访问个人资源（云盘/日历/文档），`--as bot` 只能访问应用自身资源。
权限不足时按身份处理：user 用 `lark-cli auth login --scope "<missing>"`
增量授权（多次累积）；bot 把错误里的 console_url 给用户去后台开 scope，
禁止对 bot 跑 auth login。

---

## 2026-05-16 — Edit 工具精确替换

**类型**: 经验

Edit 的 old_string 必须与文件逐字匹配（含缩进），且唯一；改前必须先 Read。
大范围结构调整用 Write 重写整文件更稳。提交 git 前用
`git diff --cached --stat` 自检改动范围，避免误提交。

---

## 2026-05-08 — git submodule 拉取 & 内网 github mirror

**类型**: 经验

`.gitmodules` 只是声明，子模块代码要 `git submodule update --init --recursive`。
环境出网不通时走字节内网 mirror（一次全局生效）：
`git config --global url."http://go.bytedance.net/github_proxy/".insteadOf "https://github.com/"`
（同理对 `git@github.com:`）。然后再 submodule update。

## 2026-05-16 — GitHub deploy key 默认只读

**类型**: 问题

`ssh -T git@github.com` 成功但 `git push` 报
`ERROR: The key you are authenticating with has been marked as read only.`。
根因：repo Settings → Deploy keys 添加时**未勾 "Allow write access"**。该选项
创建后**不可改**，只能删掉 key 重加并勾选。或改用账号级 SSH key（默认有写权限）。
排查口诀：ssh -T 通 = 认证 OK；push 被拒看是只读 deploy key 还是账号无权限。

## 2026-05-16 — 重生成 SSH key 不要覆盖原 key

**类型**: 经验

机器上原 `id_rsa` 可能属别的账号或被集群（ssh_master/ssh_worker）在用。
重生成时：① 先 `cp id_rsa id_rsa.bak_<ts>` 备份；② 新 key 用**独立文件名**
（如 `id_ed25519`）不覆盖；③ 在 `~/.ssh/config` 加 `Host github.com` +
`IdentityFile <新key>` + `IdentitiesOnly yes` 只对 github 用新 key，不影响
其他 host。切忌直接 `ssh-keygen -f id_rsa` 覆盖。

## 2026-04-29 — 飞书 docx 表格块不能用 children API

**类型**: 知识

Feishu 表格是带 row/column grid 的特殊 block，`/blocks/{id}/children` 增量
创建 cell 会 `1770001 invalid param` / 9499（table_cell block_type=32 带
inline children）。两条出路：① 用 `/blocks/{id}/descendants` 端点一次 POST
整张带层级的块树（需传 block_id/parent_id）；② 最稳——markdown 阶段就把
`| a | b |` 表格预处理成 heading+bullet，飞书 100% 兼容。优先方案②。

## 2026-04-29 — 飞书 drive 权限 API v1→v2 端点

**类型**: 知识

设公共权限 `PUT /drive/v1/permissions/{t}/public` → `PATCH /drive/v2/.../public`
（v2 改 PATCH 部分更新语义）；转 owner `POST /drive/v1/.../transfer_owner`
→ `POST /drive/v1/.../members/transfer_owner`（少了 `/members`）。
copy_entity 值 `"anyone"` → `"anyone_can_edit"`。400 时看 `field_violations`
逐个对枚举值。
