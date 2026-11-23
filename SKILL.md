# 飞书妙记反馈提取

> 从飞书妙记一键提取 AI 产物（总结/章节/待办/逐字稿），落到任务 notes/ 或追加到已有飞书文档

## Trigger

- `/feishu-minutes {妙记URL}` — 提取妙记内容存到当前任务的 notes/
- `/feishu-minutes {妙记URL} --to {飞书文档URL}` — 提取后追加到目标文档
- 自然语言触发：
  - "根据这个妙记链接整理反馈"
  - "把这个会议纪要的内容拉出来"
  - "从飞书妙记提取总结"
  - "整理一下这场会议的反馈"

## 前置条件

- `lark-cli` 已安装
- 需要的 scope（user identity）：
  - `minutes:minutes:readonly`
  - `minutes:minutes.artifacts:read`
  - `minutes:minutes.transcript:export`
- **首次使用可能要等应用审批**：如果授权时报 "app is pending approval"，提示用户到飞书开放平台管理后台审批对应 scope，然后重新运行 `lark-cli auth login --scope "minutes:minutes:readonly minutes:minutes.artifacts:read minutes:minutes.transcript:export"`

## Pipeline

### Step 1: 解析输入

从用户输入中提取：
- **minute_token**：从 URL `https://*.feishu.cn/minutes/{token}` 中提取 `obcn...` 这段
- **目标文档**（可选）：如果用户给了 `--to {doc_url}` 或对话中提到"追加到 XX 文档"，提取目标 docx token

### Step 2: 拉取妙记内容

**正确命令**（返回 AI 产物）：

```bash
lark-cli vc +notes --minute-tokens "{minute_token}" --as user
```

返回结构：
```json
{
  "data": {
    "notes": [{
      "title": "会议标题",
      "minute_token": "obcn...",
      "artifacts": {
        "summary": "完整 AI 总结（Markdown 格式）",
        "chapters": [
          {"start_ms": "...", "stop_ms": "...", "title": "...", "summary_content": "..."}
        ],
        "todos": [{"content": "..."}],
        "transcript_file": "本地逐字稿文件路径"
      }
    }]
  }
}
```

**⚠️ 陷阱**：不要用 `lark-cli minutes minutes get` — 那个只返回元数据（title/duration/cover），拿不到 AI 产物。必须用 `vc +notes`。

### Step 3: 权限错误处理

| 错误 | 处理 |
|------|------|
| `missing required scope(s): minutes:minutes:readonly` | 后台运行 `lark-cli auth login --scope "minutes:minutes:readonly minutes:minutes.artifacts:read minutes:minutes.transcript:export"`，提取 verification URL 发给用户 |
| `authorization failed: app is pending approval` | 提示用户去飞书开放平台后台审批 scope，审批后重试 |
| `permission denied` 且未覆盖上述 scope | 检查是否完整带了三个 scope |

### Step 4: 组装结构化 Markdown

根据 artifacts 生成标准结构：

```markdown
# {会议标题}

> 来源：{妙记 URL}
> 时长：{duration 分钟}
> 日期：{YYYY-MM-DD}

---

## TL;DR
{从 summary 顶部提炼 3-5 行核心结论}

## 章节要点

### {chapter title} ({start}~{stop})
{chapter summary_content}

...（按 chapters 顺序）

## 完整总结
{artifacts.summary 原文}

## 待办事项
- [ ] {todo 1}
- [ ] {todo 2}
...

## Action Items（提炼版）
{从 summary 和 todos 中提炼出具体可执行的动作，按责任人/方向分类}
```

### Step 5: 落盘

#### 5a: 存到当前任务的 notes/（默认）

1. 识别当前任务目录（通过 `pwd` 或 CLAUDE.md 线索）
2. 如果在 `~/Projects/rf/active/T*/` 下，存到 `notes/{YYYY-MM-DD}_{简化会议名}.md`
3. 如果不在任务目录下，询问用户要存到哪里

#### 5b: 追加到飞书文档（用户指定时）

```bash
lark-cli docs +update \
  --doc "{target_doc_url_or_token}" \
  --mode append \
  --as user \
  --markdown '{结构化 markdown}'
```

**注意**：append 前可以先 `docs +fetch` 确认文档存在且有权限。

### Step 6: 输出 TL;DR

终端输出简要反馈：

```
## 飞书妙记提取：{会议标题}

**时长**：{minutes} 分钟 | **章节数**：{N}

### TL;DR
{3-5 行核心结论}

### 关键 Action Items
{从 todos 和 summary 提炼}

---
已存到：{文件路径}
（可选）已追加到：{飞书文档 URL}
```

## Rules

1. **命令选择**：**必须**用 `lark-cli vc +notes --minute-tokens`，**不要**用 `lark-cli minutes minutes get`（后者只返回元数据）
2. **Scope 齐全**：三个 minutes scope 要一起申请，缺任一都会报错
3. **落盘优先任务目录**：如果当前在任务目录下，默认存 `notes/`；不在任务目录时询问
4. **不删原始数据**：原文 summary 要保留在"完整总结"章节，不要只写提炼版
5. **TL;DR 要有**：章节或 summary 再长，终端输出也必须有 3-5 行 TL;DR，方便用户快速判断
6. **章节时间戳**：保留 chapters 的 start_ms/stop_ms 转成 mm:ss 格式，方便用户跳转
7. **追加模式用 append**：不要用 overwrite，以免覆盖已有内容
8. **首次使用引导**：如果用户没有对应 scope，不要自己反复尝试，直接引导用户完成授权

## Error Handling

| 情况 | 处理 |
|------|------|
| URL 不含 minute_token | 提示用户提供正确的 `/minutes/obcn...` 格式 URL |
| `vc +notes` 报 missing scope | 后台起 `auth login` 引导授权 |
| `app is pending approval` | 提示去飞书开放平台后台审批 scope |
| `vc +notes` 返回空 artifacts | 可能妙记还没生成 AI 产物（会议刚结束），告知用户稍等再试 |
| 目标文档 append 失败（404/权限） | 先 `docs +fetch` 验证文档存在和权限，报错后询问用户是否改落盘到本地 |
| 不在任务目录 | 询问用户想存到哪里 |

## Examples

### 示例 1：基础提取落到任务目录

**用户输入**：
> 根据 https://your-company.feishu.cn/minutes/obcnxxxxxxxxxxxxxxxxxxxx 整理反馈

**执行**：
1. 提取 minute_token = `obcnxxxxxxxxxxxxxxxxxxxx`
2. `lark-cli vc +notes --minute-tokens obcnxxxxxxxxxxxxxxxxxxxx --as user`
3. 拿到 artifacts（summary + chapters + todos）
4. 识别当前任务为 `T20260324-001_龙虾交易大赛增长/`
5. 生成 `notes/2026-04-07_rf小龙虾欢乐群复盘.md`
6. 输出 TL;DR

### 示例 2：提取后追加到目标文档

**用户输入**：
> https://your-company.feishu.cn/minutes/obcnxxxxxxxxxxxxxxxxxxxx 把这个反馈补充到 https://your-company.feishu.cn/docx/xxxxxxxxxxxxxxxxxxxxxxxxxxx 这个文档里

**执行**：
1. 提取两个 URL
2. `vc +notes` 拉取妙记内容
3. 组装为"反馈收集结果"章节（带日期标题）
4. `docs +update --doc xxxxxxxxxxxxxxxxxxxxxxxxxxx --mode append --markdown {...}`
5. 同时存一份到当前任务 `notes/`
6. 输出 TL;DR + 两个链接

### 示例 3：首次使用，scope 未授权

**用户输入**：
> /feishu-minutes https://your-company.feishu.cn/minutes/obcn...

**执行**：
1. 尝试 `vc +notes` → 报 missing scope
2. 后台运行 `lark-cli auth login --scope "minutes:minutes:readonly minutes:minutes.artifacts:read minutes:minutes.transcript:export"`
3. 从输出提取 verification URL，发给用户
4. 用户完成授权后继续 Step 2
5. 如果报 `app is pending approval`，提示用户去飞书开放平台后台审批
