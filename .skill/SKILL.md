---
name: ielts
description: 雅思学习全流程辅助 — 进度概览、真题/模拟题练习(写作/口语/阅读/听力)、碎片记忆练习、YouTube 学习材料(影子跟读 HTML)。用户输入 /ielts 或带子命令时触发。
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash(mkdir *)
  - Bash(ls *)
  - Bash(open *)
  - Bash(date *)
  - Bash(cat *)
  - WebFetch
  - WebSearch
---

# /ielts — 雅思学习辅助

参数: `$ARGUMENTS`

本 skill 帮助用户备考 IELTS,覆盖四大模块: **概览、练习、随机练习、学习材料**。所有用户状态保存在 `~/.claude/skills/ielts/data/`,所有评分标准、模板、题库等参考资料保存在 `~/.claude/skills/ielts/references/`(按需读取,不要一次全读)。

**语言策略**: 中文讲解 + 英文示例/改写。讲解、点评、思路用中文;原文、改写、词汇示例用英文。

**今日日期**: 用 `date +%Y-%m-%d` 获取(不要假设)。

---

## 0. 启动前置检查

在执行任何命令前:

1. 用 `Bash(ls ~/.claude/skills/ielts/data/profile.json)` 检查 profile 是否存在。
2. 如果不存在,且用户命令不是 `setup`,提示: "首次使用请先运行 `/ielts setup` 创建档案",然后停止。
3. 如果存在,Read 它,把字段载入上下文备用。

---

## 1. 参数分发

把 `$ARGUMENTS` 按空格切分。第一个 token 是主命令,第二个是子命令(可选),其余作参数。识别 `--web` 任意位置出现作为标志。

| 第一个 token | 路由到 |
|---|---|
| (空) | §2 概览 |
| `setup` | §3 首次配置 |
| `plan` | §4 学习计划 |
| `progress` | §5 进度详情 |
| `practice` | §6 系统练习 |
| `quick` | §7 随机练习 |
| `materials` | §8 学习材料 |
| `errors` | §9 错题本 |
| `help` 或未识别 | §10 帮助菜单 |

---

## 2. 概览 (默认)

`/ielts` 或 `/ielts --web`

### 终端版输出格式

```
━━━ IELTS Overview · <今日日期> ━━━

📊 档案
   当前 <currentBand> → 目标 <targetBand>   距考试 <N> 天 (<examDate>)
   弱项: <weakAreas>

📅 本周计划 (Week <W>)
   <从 plan.md 抽出本周块,显示未完成项前 5 个>

📈 最近 3 次练习
   <date>  <type>  <scores 简写>
   ...

📌 错题到期 <N> 题   |   记忆卡到期 <M> 张

━━━ 可用命令 ━━━
  /ielts setup           更新档案
  /ielts plan [30d|60d|90d]
  /ielts practice ...    系统练习(真题/模拟)
  /ielts quick ...       碎片练习
  /ielts materials ...   YouTube 学习材料
  /ielts errors          错题本
  /ielts progress        进度详情
  加 --web 任意命令 → 生成 HTML
```

### --web 版本 (生成集成 App)

`--web` 输出的不是单页, 而是一个**集成 App**: `<webOutputDir>/index.html` 是主入口, 含底部 4 个 tab (概览/材料/测验/计划)。详情页 (shadow / summary / quiz) 是独立 HTML 文件, 链接回 `index.html#materials` 或 `index.html#quizzes`。

生成流程:
1. Read `templates/index.html`
2. 替换占位符:
   - `{{currentBand}}` `{{targetBand}}` `{{daysLeft}}` `{{examDate}}` `{{generatedAt}}`
   - `{{weekNum}}` 当前 week (从 plan.md 推断)
   - `{{weeklyTasks}}` HTML `<li>` 列表
   - `{{recentPractice}}` HTML `<tr>` 行
   - `{{errorsDue}}` `{{memoryDue}}` 数字
   - `{{weakAreasTags}}` HTML `<span class="tag">` 列表
   - `{{materialsList}}` 学习材料 tab 内容: `Bash(ls <webOutputDir>/shadow-*.html <webOutputDir>/summary-*.html)` 拿到列表, 渲染成 `.item-card` 卡片, 每个 href 指向对应详情 HTML
   - `{{quizzesList}}` 测验 tab 内容: 同上, `ls <webOutputDir>/quiz-*.html`
   - `{{planContent}}` 计划 tab 内容: 读 `data/plan.md` 转 HTML, 按周分块 (用 `.plan-week` 结构), 当前周加 `.current`
3. 如材料/测验为空, 渲染 `.empty` 占位
4. 写入 `<webOutputDir>/index.html`, 返回路径并询问是否 `open`

**重要**: 每次执行 `materials shadow / summary` 或 `quick test --web` 生成新详情页时, **自动重新生成 index.html** 让列表保持同步。详情页文件名规范: `shadow-<videoId>.html`, `summary-<videoId>.html`, `quiz-<YYYYMMDD>-<topic>.html`。

### 自动部署到 GitHub Pages

若 `profile.json` 含 `"deployMethod": "github-pages"`:

每次生成/更新 HTML 后, 在 `webOutputDir` 内执行:

```bash
cd <webOutputDir> && \
git add -A && \
git diff --cached --quiet || \
  (git -c user.email=<email> -c user.name=<deployRepo拥有者> commit -q -m "Update: <生成的文件名 或 "dashboard"> @ <YYYY-MM-DD HH:MM>" && git push -q)
```

- `git diff --cached --quiet` 用于跳过空提交
- email/name 从 git config 或 profile 推断
- 提交信息简洁, 描述生成了什么
- 静默推送 (`-q`), 失败时告知用户但不阻断本地生成

部署后告知用户公开 URL (`profile.deployUrl`),提示 GitHub Pages 重新构建通常 30-60 秒。

如 `deployMethod` 为其他值 (例如 `local-only`) 或缺失,跳过推送步骤,只生成本地文件。

---

## 3. 首次配置 `setup`

如果 `profile.json` 已存在,先 Read,把现有值作为默认,让用户增量更新。

按顺序收集:

1. **当前雅思水平** (`currentBand`): 5.0 / 5.5 / 6.0 / 6.5 / 7.0+ / 未考过
2. **目标分** (`targetBand`): 6.0 / 6.5 / 7.0 / 7.5 / 8.0+
3. **考试日期** (`examDate`): YYYY-MM-DD,允许"未定"
4. **弱项** (`weakAreas`,多选): writing-task1 / writing-task2 / speaking-part1 / speaking-part2 / speaking-part3 / reading / listening
5. **HTML 输出目录** (`webOutputDir`): 默认 `~/.claude/skills/ielts/data/web/`。提示:"如要在手机上看生成的 HTML,建议改成 iCloud Drive 路径,例如 `~/Library/Mobile Documents/com~apple~CloudDocs/ielts-web/`"

用 AskUserQuestion 收集前 4 项,webOutputDir 直接询问是否使用默认。

写入 `~/.claude/skills/ielts/data/profile.json`:

```json
{
  "currentBand": 6.5,
  "targetBand": 7.0,
  "examDate": "2026-08-15",
  "weakAreas": ["writing-task2", "speaking-part2"],
  "webOutputDir": "/Users/jiangvv/.claude/skills/ielts/data/web",
  "createdAt": "<今日日期>",
  "updatedAt": "<今日日期>",
  "history": []
}
```

完成后提示: "档案已创建。运行 `/ielts plan 60d` 生成学习计划,或 `/ielts` 看概览。"

---

## 4. 学习计划 `plan [30d|60d|90d]`

- 无参 → Read `data/plan.md`,显示当前计划。如不存在,提示用户选择天数。
- 带 `30d|60d|90d` → 读 `references/plan-templates.md`,按 profile 的 weakAreas/currentBand/targetBand 定制,生成 `data/plan.md`,按周分块。

计划生成原则:
- 写作每周 2-3 篇 (含 task1 + task2)
- 口语每周 3-4 次 (P1/P2/P3 轮换)
- 阅读每周 3-4 套(可分单题练习)
- 听力每周 3-4 套(配合 materials 视频)
- 每天 15 分钟 quick memory
- 每周一次错题复盘

---

## 5. 进度详情 `progress`

Read profile.json 的 `history`,按 type(writing-task2/speaking-part2 等)分组,显示分数趋势:

```
Writing Task 2 (最近 5 次)
   2026-04-30  TR 6.0 CC 6.0 LR 6.5 GRA 6.0   → 6.0
   2026-05-07  TR 6.5 CC 6.0 LR 6.5 GRA 6.5   → 6.5
   ...
   趋势: ↑ 0.5  弱项: TR
```

也支持 `progress writing` / `progress speaking` 等过滤。

---

## 6. 系统练习 `practice`

子分发:

| 命令 | 分支 |
|---|---|
| `practice` 无参 | 显示菜单 |
| `practice real [<pdf>] [<module>] [<item>]` | §6.1 真题模式 |
| `practice mock <module> [<sub>]` | §6.2 模拟题模式 |
| `practice grade <task1\|task2\|speaking>` | §6.3 直接批改 |

### 6.1 真题模式 `practice real`

- 无 pdf 参数 → `Bash(ls ~/.claude/skills/ielts/data/cambridge/)`,列出所有 PDF。若空,提示"请把剑桥真题 PDF 放到 `~/.claude/skills/ielts/data/cambridge/`"并停止。
- 用户选定 PDF 后,询问 module (writing/speaking/reading/listening) 和题号(如 Test 1 Reading Passage 1)。
- 用 `Read(file_path=<pdf>, pages="<range>")` 解析对应页(写作 Task 通常在固定页,阅读 Passage 1/2/3 也是)。如不知页号,先 Read 前 5 页找目录。
- 进入对应模块的练习流程(批改/对答)。

### 6.2 模拟题模式 `practice mock`

| 子模块 | 行为 |
|---|---|
| `mock writing task1` | Read `references/writing-task1-playbook.md`,生成一道仿真题(线/柱/饼/表/流程/地图/书信 随机或按用户指定)。用户提交作文后按 §6.4 批改 |
| `mock writing task2` | Read `references/writing-task2-playbook.md`,生成一道题。同上批改 |
| `mock speaking part1` | Read `references/speaking-topics-2026Q2.md`,抽 3-4 个 P1 题。用户文字回答 → §6.4 批改(口语版) |
| `mock speaking part2` | 抽一道 P2 题卡,显示 cue card + 1 分钟准备提示。回答后批改 |
| `mock speaking part3` | 给 4-5 道 P3 拓展题,逐题回答批改 |
| `mock reading [topic]` | 按 `references/reading-strategies.md` 生成 1 段 700-900 词仿真阅读 + 13 道题(题型混合)。用户答完对答案 + 解析 + 错题入 `data/errors.json` |
| `mock listening [section]` | 脚本模式:展示听力原文(无音频)+ 10 道题。同上 |

### 6.3 直接批改 `practice grade`

用户已经写好作文/口语回答,直接粘贴。skill 询问是 task1/task2/speaking part 几,然后按 §6.4 批改,不出新题。

### 6.4 批改输出格式 (写作/口语共用)

```markdown
【总分预估】 6.5 (TR 7 / CC 6 / LR 6.5 / GRA 6.5)
                  ↑ 写作四项: Task Response / Coherence & Cohesion / Lexical Resource / Grammatical Range & Accuracy
                  口语四项: Fluency & Coherence / Lexical Resource / Grammatical Range / Pronunciation (建议项,不打分)

【逐项点评】
- **TR (Task Response)**: 1-3 条问题,引用原文行号或片段
- **CC (Coherence & Cohesion)**: ...
- **LR (Lexical Resource)**: ...
- **GRA (Grammatical Range & Accuracy)**: ...

【原文标注】
> 用户原文,**粗体标问题词**,~~删除线标多余~~,[斜体备选]

【7+ 改写版本】
整段重写(保持原意,提升结构 + 地道度)

【3-5 个必记表达】
1. `ephemeral` (adj.) 短暂的 — Social media trends are often ephemeral.
2. `bridge the gap between X and Y` — 弥合分歧
3. ...
→ 询问用户:"是否将这些加入 memory-deck?"如同意,追加到 `data/memory-deck.json`

【下一步建议】
1-2 条针对此次最该练的点
```

批改完成后:
1. 把作文/口语原文 + 反馈写入 `data/writing/<YYYY-MM-DD>-task{1|2}.md` 或 `data/speaking/<YYYY-MM-DD>-part{1|2|3}.md`
2. 更新 `profile.json` 的 `history` 数组,追加 `{date, type, scores}`
3. 错题/疑问入 `errors.json`(可选)

读 `references/band-descriptors-writing.md` 或 `band-descriptors-speaking.md` 作为打分依据。读 `references/high-band-vocab.md` 作为改写词汇库。

---

## 7. 随机练习 `quick`

子分发:

| 命令 | 行为 |
|---|---|
| `quick` 无参 | 显示菜单 |
| `quick test [module]` | §7.1 出 3-5 道小题 |
| `quick memory [type]` | §7.2 记忆卡复习 |
| `quick add` | §7.3 添加记忆内容 |

### 7.1 `quick test`

5-10 分钟可完成的小测验。`module` 可选 reading/listening/writing-sent(单句改写)/speaking-mini(30 秒答题)。无 module 则随机一种。

- reading: 出 1 段 200 词短文 + 3 道题
- listening: 出 1 段对话脚本 + 3 道题
- writing-sent: 给 3 个中文短句让用户翻译,或给 3 个低分句让用户升级
- speaking-mini: 出 3 个 P1 题让用户 30 秒内文字作答

判分简化版:不打四项分,只标对错 + 1-2 条点评。错题入 errors.json。

加 `--web` → 生成 `templates/quiz.html` 的 HTML 页(单选/填空交互),保存到 webOutputDir。

### 7.2 `quick memory [type]`

Read `data/memory-deck.json`,筛 `nextReviewAt <= 今日`,按 type 过滤(可选),随机抽 5-10 张。

每张卡片:
- 显示 `front`(中文/题干)
- 等用户输入
- 显示 `back`(英文/答案) + 用户对错自评(对/错)
- 对 → `interval` 翻倍,`nextReviewAt = 今日 + 新 interval`;错 → `interval = 1`,`nextReviewAt = 今日 + 1`
- `reviewCount += 1`,回写 deck

新用户的 deck 为空时,从 `references/memory-deck-seed.md` 导入。

### 7.3 `quick add`

用户粘贴内容,skill 智能拆成卡片:
- 单词列表 → vocab 卡片
- 句型 → sentence 卡片
- 短语/搭配 → phrase 卡片
- 语法规则 → grammar 卡片

追加到 `data/memory-deck.json`,新卡片 `interval=1, nextReviewAt=明天`。

---

## 8. 学习材料 `materials`

子分发:

| 命令 | 行为 |
|---|---|
| `materials` 无参 | 显示菜单 + 读 `references/youtube-channels.md` 展示推荐频道 |
| `materials find <topic>` | §8.1 找视频 |
| `materials shadow <url>` | §8.2 影子跟读 HTML |
| `materials summary <url>` | §8.3 知识点总结 HTML |
| `materials videos` | §8.4 列已生成的 HTML 页 |

### 8.1 `materials find <topic>`

用 WebSearch 查询: `<topic> site:youtube.com IELTS OR "English learning"`。从结果筛选已知英语学习频道(参考 `references/youtube-channels.md`)优先,返回 5-10 个候选:

```
1. BBC Learning English · 6 Minute English: <title>
   https://www.youtube.com/watch?v=...
   时长 ~6 分钟 · 难度 B1-B2

2. ...
```

询问用户选哪个 → 进入 shadow 或 summary。

### 8.2 `materials shadow <youtube-url>`

1. 解析 URL 提取 video ID (`v=XXXXX` 或 `youtu.be/XXXXX`)。
2. 拉字幕: WebFetch `https://youtubetotranscript.com/transcript?v=<videoId>` 提取 prompt: "Extract the full English transcript with timestamps if available, otherwise just the text."
3. 字幕拉取失败时,告知用户并问: "请把字幕粘贴给我,我继续生成跟读页",拿到后继续。
4. 把字幕按句子(. ! ? 切分)分段,标记长难词(GRE/雅思 7+ 级)。
5. Read `templates/shadow-reading.html`,替换占位符:
   - `{{videoId}}` 视频 ID
   - `{{videoTitle}}` 标题
   - `{{sentences}}` 句子 JSON 数组 `[{en, hardWords:[{w,zh}]}]`
6. 写到 `<webOutputDir>/shadow-<videoId>.html`,返回路径,问是否 `open`
7. **重新生成 index.html** 让材料 tab 出现这个新链接

### 8.3 `materials summary <youtube-url>`

同 8.2 步骤 1-3 拉字幕,然后:

4. 提取以下四类知识点(LLM 分析):
   - **词汇 (Vocabulary)**: 10-15 个 7+ 级或地道用法
   - **搭配 (Collocations)**: 5-10 个高频搭配
   - **句型 (Sentence Patterns)**: 3-5 个值得套用的句式
   - **语法 (Grammar)**: 2-3 个语法点(虚拟语气/倒装/复杂从句等)
5. Read `templates/video-summary.html`,替换占位符 `{{videoId}}` `{{videoTitle}}` `{{vocab}}` `{{collocations}}` `{{patterns}}` `{{grammar}}`(后四个是 HTML 列表片段)。
6. 写到 `<webOutputDir>/summary-<videoId>.html`,返回路径,问是否 `open`,询问是否把词汇 + 搭配加入 memory-deck
7. **重新生成 index.html** 同步材料列表

### 8.4 `materials videos`

`Bash(ls <webOutputDir>/*.html)`,列出已生成的 shadow / summary 页面,显示文件名 + 修改时间。

---

## 9. 错题本 `errors`

| 命令 | 行为 |
|---|---|
| `errors` 或 `errors review` | 显示到期错题(`nextReviewAt <= 今日`),按艾宾浩斯逐题复习 |
| `errors add` | 手动添加(询问题型/题干/正确答案/错误原因) |
| `errors list [module]` | 列出全部错题 |
| `errors stats` | 按题型/话题统计错误分布 |

错题 schema 在 `data/errors.json`:

```json
[
  {
    "id": "rd-20260521-001",
    "type": "reading",
    "subType": "T-F-NG",
    "topic": "environment",
    "question": "...",
    "userAnswer": "T",
    "correctAnswer": "NG",
    "reason": "原文未提及具体数据",
    "addedAt": "2026-05-21",
    "nextReviewAt": "2026-05-22",
    "interval": 1,
    "reviewCount": 0
  }
]
```

复习时若再错 → interval 重置为 1;答对 → interval 翻倍 (1→2→4→7→15→30 天)。

---

## 10. 帮助菜单 `help`

显示 §1 路由表的精简版 + 一段 quick start:

```
未配置档案 → /ielts setup
看进度     → /ielts
做整段练习 → /ielts practice mock writing task2
做碎片练习 → /ielts quick memory
找学习视频 → /ielts materials find IELTS speaking part 2
所有命令加 --web 可生成 HTML 页
```

---

## 通用规则

- **不要一次读完所有 references/templates**。只在路由到对应分支时读必要文件。
- **日期**: 用 `date +%Y-%m-%d` 获取今日日期,不要假设。
- **状态文件不存在**: 写入前用 `mkdir -p` 确保父目录在。
- **profile.json 缺字段**: 旧档案缺新字段时,以默认值补齐并写回。
- **HTML 模板渲染**: 用 Read 加载模板,字符串替换 `{{占位符}}`,Write 输出文件。不需要 Jinja。
- **AskUserQuestion**: 在 setup、选 PDF、选视频候选、确认入 deck 等场景使用,提升交互体验。
- **批改是核心价值**: 任何练习的反馈必须给出可操作的下一步,而不只是分数。
