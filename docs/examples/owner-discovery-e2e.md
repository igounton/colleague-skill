# 端到端测试：本人画像自动发现信息来源

在真实的 Claude Code（2.1.296 (Claude Code)）里，从零开始跑完一次“初始化本人画像”的完整流程。

## 测试环境

- 一个全新的 HOME，里面只装了 Distilly（`distilly install claude-code`），没有任何旧画像。
- 已有的 `~/.claude/CLAUDE.md` 里只有一行规则：`Use pnpm, not npm.`，用来检查写入时不会覆盖它。
- 用户级 MCP 里注册了 4 个**模拟**连接器，数据是虚构人物 “Lin” 的：
  - `gmail`：可读邮件，也有发信工具
  - `google-calendar`：日历
  - `x`：本人资料、本人帖子，也有发帖工具
  - `weather`：天气，与用户无关，作为干扰项
- `~/Downloads` 里放了一个 X 数据归档 `twitter-2026-09-01-archive.zip` 和一张无关的 `holiday.jpg`。
- Agent 看到的提示里没有任何“去读 Gmail / 日历 / X”的指示，只有 Distilly 的 skill。

## 结果

| 检查项 | 结果 |
|---|---|
| 自动发现会话里的连接器（gmail、google-calendar、x） | 通过，并跳过了无关的 weather |
| 自动发现本机的用户文件和 X 归档 | 通过（`owner_sources.py discover`） |
| 列出读取计划，并问“用哪些、还要补充什么” | 通过（第 1 轮） |
| 只读用户选中的来源，用户没选的归档没读 | 通过（附录 C：只有 4 次只读调用） |
| 按建议的范围读取：只读已发邮件、过去 4 周日历、本人帖子 | 通过（`in:sent`；09-14 至 10-12；`get_my_posts`） |
| 从未调用发信、发帖等写操作 | 通过（附录 C 里没有 `send_message` 或 `create_post`） |
| 只问来源里没答到的问题 | 通过（第 2 轮） |
| 每条事实都标注来源 | 通过（附录 B 的“来源”表） |
| 保存前先给用户看全文，写入前先给用户看 dry-run | 通过（第 3、4 轮） |
| 写入 CLAUDE.md 时保留原有内容，并自动备份 | 通过（附录 A） |
| 新会话会自动用上画像 | 通过（第 6 轮：知道是 Lin、避开周二上午的专注时段、约会前先问） |

## 对话记录

### 第 1 轮

**用户：** 帮我用 Distilly 初始化一下我的个人画像

**Agent 调用的工具：**

- `Skill` {"skill": "distilly"}
- `Read` {"file_path": "~/.claude/skills/distilly/prompts/owner/intake.md"}
- `Bash` {"command": "python3 \"~/.claude/skills/distilly/tools/owner_sources.py\" discover", "description": "Discover local owner profile sources"}
- `Bash` {"command": "python3 \"~/.claude/skills/distilly/tools/owner_sources.py\" classify gmail google-calendar x weather", "description": "Classify discovered sources"}

**Agent 回复：**

> 我先看了一下这个 Agent 能读到哪些信息。下面是建议读取的来源，全部只读，不会发送、发布或修改任何东西：
>
> 1. **Gmail**：读你最近发出的 10–20 封邮件（不读收件）。可以看出你的语气、落款、常用语言，以及你常和哪些角色的人合作。
> 2. **Google 日历**：读过去 4 周的事件时间和标题。可以看出你的一周节奏、会议多少、专注时段和下线时段。
> 3. **X（推特）**：读你自己的简介和最近的帖子、回复（不读时间线）。可以看出你的公开表达风格、关注话题和发帖习惯。
> 4. **X 数据归档** `twitter-2026-09-01-archive.zip`（在 ~/Downloads）：读你自己的推文和简介。内容比第 3 项更全，但只用于了解你，不会被收进画像原文。
> 5. **现有 `~/.claude/CLAUDE.md`**：这是你自己写给 Agent 的备注，目前只看到“用 pnpm，不用 npm”这一条规则，我会读它并保留。
> 6. **天气工具**：和了解你无关，我已跳过。
>
> 请回复两点：
> - **用哪些来源？** 例如“1, 2, 3”“全部”或“都不用”。
> - **还有别的要补充吗？** 例如别的账号、其他 AI 助手的记忆导出、笔记、简历或某个文件夹。
>
> 选完来源后，我只会问这些来源没能回答的问题，比如怎么称呼你、你的职业、时区、偏好的沟通方式。画像会完整展示给你确认后才保存。

### 第 2 轮

**用户：** 1、2、3、5 都用，归档文件先不用。别的没有要补充的了。

**Agent 调用的工具：**

- `ToolSearch` {"query": "select:mcp__gmail__search_messages,mcp__google-calendar__list_events,mcp__x__get_my_profile,mcp__x__get_my_posts", "max_results": 4}
- `Read` {"file_path": "~/.claude/CLAUDE.md"}
- `Read` {"file_path": "~/.claude/skills/distilly/prompts/owner/profile_builder.md"}
- `mcp__gmail__search_messages` {"query": "in:sent", "max_results": 15}
- `mcp__google-calendar__list_events` {"time_min": "2026-09-14T00:00:00Z", "time_max": "2026-10-12T23:59:59Z"}
- `mcp__x__get_my_profile` {}
- `mcp__x__get_my_posts` {"limit": 20}

**Agent 回复：**

> 我读完了 Gmail、日历、X 和你的 CLAUDE.md，了解到的情况如下：
>
> - **身份**：你是产品设计师，做设计系统和 AI Agent，在上海。邮件落款是“Lin / 林”，X 账号是 @linmakes。最近刚发布了 tokens v2。
> - **语言**：中英文混用。对内部同事和朋友多用英文，对供应商用中文，推文两种都有。
> - **沟通风格**：邮件很简短直接，比如“Looks good. Two nits… ship it.”。你在推文里主张 Agent 应该先展示计划再动手。
> - **节奏**：工作日 10:00–10:15 有站会。周二和周四 9:00–12:00 是专注时段，不排会议。周三 14:00 有设计评审，周三晚上去健身房。周六上午攀岩（抱石）。
> - **偏好**：用 pnpm，不用 npm。
>
> 以下问题来源里没有答案，请一起回答，跳过也行：
>
> 1. 我该怎么称呼你？叫 Lin 还是林？
> 2. 你在哪家公司，担任什么职级？还有你在 X 上写的 AI Agent 具体指做什么？
> 3. 时区我先按 Asia/Shanghai，语言先按中英文随你切换。可以吗？
> 4. 你现在最关注的 1–3 件事是什么？
> 5. 你希望我怎么跟你说话？比如简短还是详细，要不要用 emoji。
> 6. 哪些事我可以直接做，哪些要先问你？
> 7. 你最讨厌助手做什么？有没有什么事我绝对不能做？
>
> 答完后我会生成完整画像，让你确认后再保存。

### 第 3 轮

**用户：** 叫我 Lin。在一家做协作工具的创业公司当高级产品设计师，AI Agent 指的是我在设计我们产品里的 agent 交互。时区和语言就这样。最近最关注：tokens v2 落地、agent 交互规范、下个月的设计大会分享。说话简短，先给结论，不要 emoji。改文件、查资料可以直接做；发邮件、发帖、动日历一定先问我。最讨厌啰嗦和没确认就替我对外说话。

**Agent 调用的工具：**

- `Write` {"file_path": "/tmp/distilly_owner_profile.md", "content": "（画像草稿全文，见附录 B）"}

**Agent 回复：**

> 以下是完整画像草稿，写在 `/tmp/distilly_owner_profile.md`，还没保存：
>
> ---
>
> # 本人画像：Lin
>
> ## 常驻摘要
>
> - 称呼：Lin（中文落款“林”）
> - 角色与背景：协作工具创业公司的高级产品设计师，在上海。做设计系统，并设计产品内的 AI Agent 交互
> - 时区与语言：Asia/Shanghai。中英文混用，跟随 Lin 当前使用的语言
> - 当前重点：tokens v2 落地、Agent 交互规范、下个月的设计大会分享
> - 沟通方式：简短，先给结论，不用 emoji
> - 行动与询问：改文件、查资料可直接做。发邮件、发帖、动日历必须先问，未经确认不得替 Lin 对外发声
> - 禁忌：啰嗦，没确认就对外说话
> - 节奏：
>   - 工作日 10:00 站会。
>   - 周二、周四 9:00–12:00 是专注时段，不要安排会议或打扰。
>   - 周三 14:00 设计评审。
>   - 周六上午攀岩。
> - 开发偏好：用 pnpm，不用 npm
>
> ## 其余各节（详情，按需读取）
>
> - **工作与项目**：tokens v2 刚发布，正在落地。Agent 交互规范在制定中。下个月有设计大会分享。
> - **沟通偏好**：邮件风格直接简短（推断）。对同事和朋友多用英文，对供应商等外部场景用中文（推断）。你公开主张“Agent 界面应先展示计划再动手”，所以我在执行有外部影响的动作前会先说明要做什么。
> - **作息**：除摘要里的节奏外，周三 19:00 健身，周六 10:00–12:00 攀岩。你会把一对一挪到下午，因为上午是专注时段。
> - **工具与账号**：Gmail、Google 日历、X（@linmakes）。
> - **待确认**：公司名称和具体职级你没提供，我没有记录。
> - **来源表**：每条事实都标了来源（面谈、Gmail、日历、X、CLAUDE.md），带“推断”的已标明。
>
> ---
>
> 隐私方面，我没有记录任何联系人姓名、邮件正文原文、发票信息，也没有记录具体的健身房或住址。
>
> 需要改动吗？如果没问题，我会先保存到 `~/.distilly/owner/profile.md`，再用 `--dry-run` 给你看写入 `~/.claude/CLAUDE.md` 会改哪些内容，你同意后再真正写入。

### 第 4 轮

**用户：** 没问题，保存吧，然后给我看 dry-run。

**Agent 调用的工具：**

- `Bash` {"command": "S=\"~/.claude/skills/distilly/tools/owner_profile.py\"; python3 \"$S\" save --profile /tmp/distilly_owner_profile.md && python3 \"$S\" link --host claude-code --dry-run", "description": "Save owner profile and preview link dry-run"}

**Agent 回复：**

> 画像已保存到 `~/.distilly/owner/profile.md`。`~/.claude/CLAUDE.md` 还没动，下面是 dry-run 的结果。
>
> **改动内容**
> - 原有的 `# My rules` 和 “Use pnpm, not npm.” 保持原样。
> - 末尾新增一段，用 `distilly:owner-profile:begin/end` 标记包起来，内容是上面确认过的常驻摘要。它开头写明“用作背景，不要模仿”，结尾带完整画像的路径。
> - 只有标记之间的内容会被 Distilly 以后更新，标记外的文字不会动。写入前旧文件会自动备份。
> - 要撤销，运行 `owner_profile.py unlink --host claude-code`。
>
> 要我现在真正写入吗？

### 第 5 轮

**用户：** 写入吧。

**Agent 调用的工具：**

- `Bash` {"command": "S=\"~/.claude/skills/distilly/tools/owner_profile.py\"; python3 \"$S\" link --host claude-code && rm -f /tmp/distilly_owner_profile.md && ls /tmp/distilly_owner_profile.md 2>&1", "description": "Write profile link and remove temp file"}

**Agent 回复：**

> 写入成功了。命令最后显示的退出码 2 只是因为我用 `ls` 确认临时文件已删除，它已经不存在了，这是预期结果。
>
> - `~/.claude/CLAUDE.md` 已更新。你原来的 “Use pnpm, not npm.” 保留着，下面新增了 Distilly 的画像段落。
> - 旧版本备份在 `~/.claude/CLAUDE.md.distilly-backup-20261012T081718Z`。
> - 完整画像在 `~/.distilly/owner/profile.md`。
> - 临时文件 `/tmp/distilly_owner_profile.md` 已删除。
>
> 新开的会话会自动读到这份画像。以后想更新，直接告诉我哪里不对或有新信息。要撤销，运行 `owner_profile.py unlink --host claude-code`。X 数据归档你说先不用，我没有读。

### 第 6 轮

**用户：** （新开一个会话）不用任何工具，直接回答：你在为谁工作？周二上午 10 点半给我约一个会合适吗？

**Agent 回复：**

> 我在为 Lin（林）工作，她/他/其——抱歉，资料里没写性别，我就不替 Lin 猜了。Lin 是上海一家协作工具创业公司的高级产品设计师，做设计系统和产品内 AI Agent 交互。
>
> 周二上午 10:30 不合适。周二、周四 9:00-12:00 是 Lin 的专注时段，不安排会议。建议改到周二下午，或周三除 10:00 站会和 14:00 设计评审之外的时段。
>
> 另外，约会议属于动日历、对外发声，我会先问过 Lin 再发出邀请。


## 附录 A：写入后的 `~/.claude/CLAUDE.md`

```markdown
# My rules

- Use pnpm, not npm.

<!-- distilly:owner-profile:begin -->
## About my owner (Distilly owner profile)

This describes the person you work for. Use it as context; do not imitate them.

- 称呼：Lin（中文落款“林”）
- 角色与背景：协作工具创业公司的高级产品设计师，在上海；做设计系统，并设计产品内的 AI Agent 交互
- 时区与语言：Asia/Shanghai；中英文混用，跟随 Lin 当前使用的语言
- 当前重点：tokens v2 落地；Agent 交互规范；下个月的设计大会分享
- 沟通方式：简短，先给结论，不用 emoji
- 行动与询问：改文件、查资料可直接做；发邮件、发帖、动日历必须先问，未经确认不得替 Lin 对外发声
- 禁忌：啰嗦；没确认就对外说话
- 节奏：工作日 10:00 站会；周二、周四 9:00-12:00 为专注时段，不要安排会议或打扰；周三 14:00 设计评审；周六上午攀岩
- 开发偏好：用 pnpm，不用 npm

Full profile: `~/.distilly/owner/profile.md`. Read it when a task needs more detail. Suggest corrections to the owner instead of editing this block yourself.
<!-- distilly:owner-profile:end -->
```

## 附录 B：完整画像 `~/.distilly/owner/profile.md`

```markdown
# 本人画像：Lin

> 由 Distilly 于 2026-10-12 创建，已由本人确认。

## 常驻摘要

- 称呼：Lin（中文落款“林”）
- 角色与背景：协作工具创业公司的高级产品设计师，在上海；做设计系统，并设计产品内的 AI Agent 交互
- 时区与语言：Asia/Shanghai；中英文混用，跟随 Lin 当前使用的语言
- 当前重点：tokens v2 落地；Agent 交互规范；下个月的设计大会分享
- 沟通方式：简短，先给结论，不用 emoji
- 行动与询问：改文件、查资料可直接做；发邮件、发帖、动日历必须先问，未经确认不得替 Lin 对外发声
- 禁忌：啰嗦；没确认就对外说话
- 节奏：工作日 10:00 站会；周二、周四 9:00-12:00 为专注时段，不要安排会议或打扰；周三 14:00 设计评审；周六上午攀岩
- 开发偏好：用 pnpm，不用 npm

## 工作与项目

- 设计系统：tokens v2 刚发布，当前在落地阶段（“更少的变量，更多的含义”）
- 产品内 Agent 交互：正在制定交互规范
- 下个月有一场设计大会分享

## 目标

- 把 tokens v2 落地到团队实际使用中
- 完成 Agent 交互规范
- 准备好大会分享

## 沟通偏好

- 简短、结论先行、无 emoji
- Lin 自己的邮件风格：直接，一两句话给出判断和要点（inferred）
- 对同事和朋友常用英文，对供应商等外部中文场景用中文（inferred）
- 公开观点：Agent 界面应当先展示计划再动手；设计系统是团队的共同语言

## 决策风格与自主权

- 可直接做：修改文件、查资料
- 必须先问：发邮件、发帖、修改日历
- 与 Lin 自己的观点一致，Agent 在执行有外部影响的动作前，应先展示将要做什么

## 作息与可用时间

- 工作日 10:00-10:15 站会
- 周二、周四 9:00-12:00 无会议专注时段；Lin 会把一对一往下午挪
- 周三 14:00-15:00 设计评审；周三 19:00 健身
- 周六 10:00-12:00 攀岩（抱石）

## 工具与账号

- Gmail、Google 日历、X（@linmakes）
- 开发工具链：pnpm

## 边界与不喜欢

- 不喜欢啰嗦
- 不喜欢没确认就替自己对外说话

## 待确认问题

- 公司名称和具体职级（未提供，不记录）

## 来源

| 事实 | 来源 | 日期 |
|------|------|------|
| 称呼 Lin | interview | 2026-10-12 |
| 称呼 Lin / 落款“林” | mail: gmail | 2026-10-12 |
| 高级产品设计师，协作工具创业公司 | interview | 2026-10-12 |
| 产品设计师，设计系统与 AI Agent，上海 | social: x | 2026-10-12 |
| 设计产品内 Agent 交互 | interview | 2026-10-12 |
| 时区 Asia/Shanghai，中英文混用 | interview | 2026-10-12 |
| 当前重点：tokens v2、Agent 交互规范、设计大会分享 | interview | 2026-10-12 |
| tokens v2 刚发布 | social: x | 2026-10-12 |
| 简短、结论先行、无 emoji | interview | 2026-10-12 |
| 改文件、查资料可直接做；发邮件、发帖、动日历先问 | interview | 2026-10-12 |
| 讨厌啰嗦、讨厌未经确认对外发声 | interview | 2026-10-12 |
| 邮件风格直接简短 (inferred) | mail: gmail | 2026-10-12 |
| 对内英文、对供应商中文 (inferred) | mail: gmail | 2026-10-12 |
| 周二周四上午专注时段 | calendar: google-calendar | 2026-10-12 |
| 将一对一移到下午，因上午是专注时段 | mail: gmail | 2026-10-12 |
| 工作日 10:00 站会；周三 14:00 设计评审 | calendar: google-calendar | 2026-10-12 |
| 周三晚健身、周六上午攀岩 | calendar: google-calendar | 2026-10-12 |
| Agent 界面应先展示计划；设计系统是共同语言 | social: x | 2026-10-12 |
| 用 pnpm，不用 npm | user file | 2026-10-12 |
```

## 附录 C：模拟连接器收到的全部调用

```json
{"server": "gmail", "tool": "search_messages", "args": {"query": "in:sent", "max_results": 15}, "at": "2026-10-12T08:16:14"}
{"server": "google-calendar", "tool": "list_events", "args": {"time_min": "2026-09-14T00:00:00Z", "time_max": "2026-10-12T23:59:59Z"}, "at": "2026-10-12T08:16:14"}
{"server": "x", "tool": "get_my_profile", "args": {}, "at": "2026-10-12T08:16:14"}
{"server": "x", "tool": "get_my_posts", "args": {"limit": 20}, "at": "2026-10-12T08:16:14"}
```

## 限制

- 连接器是本地模拟的 MCP server，数据是虚构的。真实的 Gmail、日历和 X 连接器，工具名和参数会不一样。Agent 是按工具描述判断用途的，所以流程相同，但没有用真实账号测过。
- Grok Bot、OpenAI Dots 这类云端 Agent 不能在这里运行。它们走的是同一套提示：先自查会话里的工具、记忆和所在平台。但没有实测。
