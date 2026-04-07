# /feishu-minutes — Claude Code Skill

从飞书妙记一键提取 AI 产物（总结/章节/待办/逐字稿），落到任务 notes/ 或追加到已有飞书文档。

## 效果

给一个飞书妙记 URL，自动拉取 AI 生成的结构化内容——总结、章节（带时间戳）、待办、完整逐字稿——然后组装成一份结构化 markdown 反馈笔记。默认存到当前任务目录的 `notes/` 下，按 `{日期}_{会议名}.md` 命名。如果顺手带一个飞书文档 URL，它会把反馈整理成带章节的追加块直接续到目标文档末尾，保留原有内容不覆盖。再配合任务管理 skill，一次会议的反馈可以从会议结束到沉淀到任务上下文，全程不离开终端。

## 安装

1. 复制 `SKILL.md` 到 `~/.claude/skills/feishu-minutes/`
2. 在 `~/.claude/CLAUDE.md` 的 `## Skills` 段添加一行：
   ```
   - `/feishu-minutes [妙记URL] [--to 目标文档URL]` — 飞书妙记反馈提取：拉 AI 产物（总结/章节/待办）→ 结构化 markdown → 落任务 notes/ 或追加到飞书文档
   ```
3. 确保 `lark-cli` 已安装并完成基础认证
4. 首次使用时会引导授权以下 scope：
   - `minutes:minutes:readonly`
   - `minutes:minutes.artifacts:read`
   - `minutes:minutes.transcript:export`

## 使用

**基础用法 — 提取反馈到任务目录**
```
/feishu-minutes https://your-company.feishu.cn/minutes/obcnxxxxxxxxxxxx
```

**提取并追加到已有飞书文档**
```
/feishu-minutes https://your-company.feishu.cn/minutes/obcnxxxxxxxxxxxx --to https://your-company.feishu.cn/docx/XXzddxxxxxxxxxxx
```

**自然语言触发**
```
根据这个妙记链接整理反馈：https://xxx.feishu.cn/minutes/obcnxxx
```

## 流程

1. **解析输入** — 从 URL 提取 `minute_token`，识别可选的目标文档
2. **拉取 AI 产物** — `lark-cli vc +notes --minute-tokens ...`（注意：不是 `minutes minutes get`，那个只返回元数据）
3. **权限引导** — 首次使用时后台起 `auth login` 申请 scope，引导用户完成授权
4. **组装 markdown** — TL;DR + 章节要点（带时间戳）+ 完整总结 + 待办 + Action Items
5. **落盘** — 默认存当前任务 `notes/`；若指定 `--to`，再用 `docs +update --mode append` 追加到目标飞书文档
6. **输出 TL;DR** — 终端输出 3-5 行核心结论和关键 action items

## 踩坑点

- **命令选择**：一定要用 `lark-cli vc +notes --minute-tokens`，`lark-cli minutes minutes get` 只返回元数据（title/duration/cover），拿不到 AI 产物
- **三个 scope 必须齐全**：少一个就会在拉 artifacts 的时候报错
- **首次授权可能要等审批**：如果企业级应用对这几个 scope 有审批流程，授权时会看到 `app is pending approval`，需要先在飞书开放平台后台审批
- **append 不是 overwrite**：追加到已有文档时必须用 append 模式，避免覆盖原有内容

## 配合使用

- **task-inbox** — 落盘到任务目录的 `notes/` 时自动匹配当前 active 任务
- **feishu-doc** — 如果要新建一个飞书文档承载反馈（而不是追加到已有的），配合 feishu-doc 使用
- **lark-vc** — lark-cli 原生技能，底层调用的就是 `vc +notes`

## License

MIT
