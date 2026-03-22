---
name: media-tools
description: "自媒体执行工具集（小红书首发，X/抖音/TikTok 陆续接入）。CDP 脚本自动化发布/搜索/数据/评论 + 运营 SOP + NLM 知识引擎集成。"
---

# media-tools：全平台创作者运营 Skill

> 当前激活：小红书
> 规划接入：X、抖音、TikTok

融合两套能力：
- **自媒体执行（工具层）**：Python CDP 脚本（发布、搜索、数据看板、评论、多账号）
- **运营 SOP** + 风控 + 爆款复刻流程

## 职责边界

**media-tools = 自媒体执行（工具层），wemedia = 自媒体运营（策略层）**

| 归 media-tools（自媒体执行） | 归 wemedia（自媒体运营） |
|---------------|-----------|
| CDP 脚本执行（publish_pipeline / cdp_publish）| 选题发现与评估 |
| 竞品数据采集（search-feeds）| 内容队列维护（HOT/EVERGREEN/SERIES）|
| 数据看板（content-data）| Publishability Gate |
| 评论回复 | Constitution-First 内容创作 |
| 登录管理 | 正文/标题/标签生成 |
| NLM 知识采集管道（media-research notebook）| 配图生成指令 |
| 平台规则驱动 | 平台内容适配（格式/结构）|

**main 的角色**：编排中心，不直接执行 CDP 脚本。main 从 wemedia 获取内容 + 执行指令，调用 media-tools 完成发布。

**禁止**：wemedia agent 不得直接调用 `python3 scripts/publish_pipeline.py`（那是 main 的职责）。

## Required Read

按需读取：
- `references/xhs-publish-flows.md` — 发布流程分支
- `references/xhs-comment-ops.md` — 评论风控 SOP
- `references/xhs-runtime-rules.md` — 运行规则
- `references/xhs-viral-copy-flow.md` — 爆款复刻
- `references/xhs-eval-patterns.md` — JS 提取模板（browser fallback 时用）
- `examples/reply-examples.md` — 评论回复范例

## 环境

- **Python 脚本目录**：`~/.openclaw/skills/media-tools/scripts/`
- **CDP 端口**：`9223`（与 OpenClaw 浏览器 18800 不冲突）
- **Chrome Profile**：`~/Library/Application Support/XiaohongshuProfiles/`
- **临时文件**：`/tmp/xhs_publish/`
- **依赖**：`requests` + `websockets`（已安装）

## 文件路径约定（与 wemedia 的接口）

| 用途 | 路径 | 写入者 | 读者 |
|------|------|--------|------|
| 内容草稿 | `~/.openclaw/workspace/agents/wemedia/drafts/{A\|B\|C}/{标识}.txt` | wemedia agent | main / media-tools |
| 配图 | `~/.openclaw/workspace/agents/wemedia/drafts/generated/{A\|B\|C}/{标识}_sq.jpg` | main / nano-banana | main / media-tools |
| 发布命令 | `cd ~/.openclaw/skills/media-tools && python3 scripts/publish_pipeline.py ...` | main 调用 | main 执行 |
| 竞品数据 | `/tmp/xhs_publish/search_{keyword}_{timestamp}.json` | media-tools | main 分析 |

> **main 是唯一读 wemedia 草稿的人**，media-tools 不直接读草稿（由 main 传递参数）。

## Hard Rules

1. **⛔ 未经晨星确认，绝不调用发布命令**
2. 登录操作必须有窗口（不能 headless），扫码由晨星完成
3. 评论回复遵循 `xhs-comment-ops.md` 风控（单次 1 条，间隔 8-15s）
4. source 删除前必须二次确认 ID 类型（notebook ID ≠ source ID）
5. 竞品扫描限频：每周最多 2 次，避免风控
6. 脚本失败不阻断流水线：Warning → 降级 → 继续
7. **发布脚本（publish_pipeline.py）是 main 的专属工具，wemedia spawn 的子 agent 不得调用**

## 路由规则

根据用户意图判断调用什么：

### 发布
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/publish_pipeline.py \
  --title "标题" --content "正文" \
  --image-urls "https://..." "https://..." \
  --headless
```
- 加 `--preview` 只填表不发布（预览模式）
- 不加 `--preview` 默认填表后自动发布
- 视频发布加 `--video /path/to/video.mp4`

### 搜索竞品
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py --headless search-feeds --keyword "关键词"
```
> 注意：`--headless` 是全局参数，必须放在子命令前面。
- 支持 `--sort-by 最热|最新|综合`
- 返回结构化 JSON

### 笔记详情 + 评论
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py --headless get-feed-detail \
  --feed-id <id> --xsec-token <token> \
  --load-all-comments --limit 20
```

### 数据看板
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py --headless content-data --csv-file /tmp/xhs_data.csv
```

### 评论检查
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py --headless get-notification-mentions
```

### 评论回复
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py --headless respond-comment \
  --feed-id <id> --xsec-token <token> \
  --content "回复内容" --comment-author "用户名"
```
- ⚠️ 必须遵循 `references/xhs-comment-ops.md` 的风控规则
- 单次只回复 1 条，间隔 8-15 秒

### 登录 / 账号管理
```bash
# 首次登录（需要有窗口 Chrome，分两步）
# 步骤 1: 启动独立 Chrome 实例
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --remote-debugging-port=9223 \
  --user-data-dir="$HOME/Library/Application Support/XiaohongshuProfiles/default" \
  --no-first-run --no-default-browser-check \
  "https://creator.xiaohongshu.com" &
# 步骤 2: 晨星扫码登录创作者中心 + 主站（两个域需要分别登录）

# 检查登录状态
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py check-login

# 获取登录二维码（远程场景）
python3 scripts/cdp_publish.py get-login-qrcode
```
> ⚠️ macOS 注意: Chrome 绑定 IPv6，已修改 CDP_HOST 为 `localhost`
> ⚠️ 创作者中心和主站是独立登录态，搜索/互动需要主站登录

### 首页推荐
```bash
cd ~/.openclaw/skills/media-tools
python3 scripts/cdp_publish.py list-feeds --headless
```

### 爆款复刻
参见 `references/xhs-viral-copy-flow.md`，流程：
1. `get-feed-detail` 抓取源笔记
2. 分析爆款因素
3. NLM 生成差异化内容
4. nano-banana 生成封面
5. `publish_pipeline.py` 发布

## NLM 知识引擎集成（方案 C+）

### 知识采集管道

| 来源 | 采集方式 | 频率 | 喂入 NLM |
|------|---------|------|---------|
| X 帖子 | browser 抓取搜索/KOL | 每日 | ✅ |
| 小红书竞品 | `search-feeds` 脚本 | 每周 2 次 | ✅ |
| 技术博客 | `web_fetch` | 热点触发 | ✅ |
| 自家数据 | `content-data` 脚本 | 每日 | ✅ |

> **media-research notebook 是共享资源**：由 media-tools 维护，wemedia 在 Step 2 创作前置链中调用查询。media-tools 负责采集和写入，wemedia 负责查询和消费。

### 采集 → NLM 流程

1. 采集原始数据（脚本/browser/web_fetch）
2. 结构化提取为 markdown
3. 保存到 `/tmp/nlm-sources/`
4. `nlm-gateway.sh source --subcmd add --notebook media-research --path <file>`
5. NLM 查询生成知识简报

### source 容量规划（~50 上限）

| 分类 | 上限 | 滚动策略 |
|------|------|---------|
| X 观点 | 10 | 2 周滚动 |
| 小红书竞品 | 8 | 4 周滚动 |
| 技术博客 | 8 | 2 周滚动 |
| GitHub 趋势 | 4 | 4 周滚动 |
| 自家数据 | 5 | 月压缩 |
| 领域知识（常驻） | 10 | 只增不删 |
| 缓冲 | 5 | — |

### NLM 查询模板

```bash
# 选题差异化分析（wemedia Step 2 用）
bash ~/.openclaw/skills/notebooklm/scripts/nlm-gateway.sh query \
  --agent main --notebook media-research \
  --query "基于竞品数据和行业趋势，{选题}的差异化角度是什么？有哪些爆款因素可以参考？"

# 内容策略建议
bash ~/.openclaw/skills/notebooklm/scripts/nlm-gateway.sh query \
  --agent main --notebook media-research \
  --query "过去 2 周哪类内容互动最高？下一步应加注什么方向？"
```

> **S 级快速路径**：时效优先，S 级跳过 NLM 查询。media-tools 可在晨间扫描时预生成一份 `intel/media-ops/DAILY-KNOWLEDGE-BRIEF.md`，wemedia 直接读文件获取当日热点背景。

## 与自媒体流水线 v1.1 的集成

| Step | 改造 |
|------|------|
| Step 0 | gemini 扫描时可调 `search-feeds` 获取竞品 |
| Step 2 | 新增 NLM 深研环节（M/L 级） |
| Step 3 | wemedia 创作输入包含 NLM 知识简报 |
| Step 6 | 适配输出附带发布命令模板 |
| Step 7 | 晨星确认后 main 调 `publish_pipeline.py` 发布 |
| Step 8 | `content-data` 抓数据，写入日结，喂 NLM |

---

## 与 wemedia Skill 的联动关系

**本质**：media-tools = 自媒体执行（工具层），wemedia = 自媒体运营（策略层）。

**media-tools 提供的共享资源**：
- `media-research` notebook（NLM 知识库，wemedia Step 2 查询用）
- `publish_pipeline.py`（main 调用，wemedia 不得直接调用）
- 竞品数据文件（main 分析后传递结论给 wemedia）
- 配图生成能力（由 main/nano-banana 执行，wemedia 只输出指令）

**media-tools 从 wemedia 接收**：
- 标准交付物（`drafts/{A|B|C}/{标识}.txt`，含标题/正文/标签/配图路径）
- 发布指令（main 转发，media-tools 执行）

**main 的调度职责**：
```
wemedia 交付 → main 读取 → media-tools 执行 → 监控群通知
```

**禁止**：wemedia agent 直接调用 `publish_pipeline.py`。所有发布必须经 main 调度。

参见 `persona.md`（待晨星填写）。默认使用晨星自定义人设，不使用预设模板。
