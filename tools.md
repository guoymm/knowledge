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
