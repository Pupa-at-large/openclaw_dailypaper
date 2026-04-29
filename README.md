# openclaw_dailypaper
AI research agent on OpenClaw (Tencent Cloud). Natural language via WeChat → auto-downloads SEC 10-K/10-Q → parses financials → generates MBB-style Chinese reports → delivers to Feishu IM + docs. Covers academic papers &amp; earnings research. Multi-agent (V4) to OpenClaw-native skill (V5).

# 龙虾 OpenClaw · 论文 & 财报研究助手

基于 [OpenClaw](https://openclaw.ai)（腾讯云）构建的 AI 研究 Agent，覆盖**学术论文分析**与**上市公司财报研究**两条主线，微信 ClawBot 自然语言触发，飞书 IM + 云文档双路交付。

---

## 功能概览

### 📄 论文助手（Paper Assistant）
- 批量文献摄入与去重（本地知识库 + manifest lookup）
- 自动生成结构化分析报告，推送飞书云文档
- 多轮对话追问，支持跨文献横向对比

### 📊 财报研究助手（Finreport Agent）
- 自然语言触发：`"帮我研究苹果 2024Q4 的财报"` → 全自动交付
- 从 SEC EDGAR 下载原始 10-K/10-Q（白名单硬校验，防 SSRF）
- pdfplumber + camelot 解析三大报表 → 结构化 JSON
- Claude 本人按 MBB 金字塔范式撰写 8 章中文详报 + 6 块 IM 简报
- Critic 5 条规则硬 gate（来源合法性 / 数字自洽 / 引用完整性 / 合理范围 / 链接可点击）
- 每个数据点带可追溯的 SEC 原文 URL（`#page=N`），链接失活自动回退本地存证

---

## 架构

```
微信 ClawBot（自然语言入口）
        ↓
OpenClaw（Claude · 读 AGENTS.md + SOUL.md）
        ↓ 命中触发词
FINREPORT_RESEARCH_SKILL 流水线
  Step -1  复用守卫（查 kb_index.json，命中直接返回）
  Step 0   Knowledge 去重 + ticker/period 解析
  Step 1   Fetcher → SEC EDGAR 下载（白名单 + Content-Type + 50MB 限制）
  Step 2   Parser  → PDF/HTML → 结构化 JSON
  Step 3   Analyst → Claude 本人写 MBB 8 章详报 + 6 块 IM 卡片
  Step 4   Critic  → 5 条规则自检，不过不交付（最多重写 2 轮）
  Step 5   Deliver → 飞书云文档写入（folder_token 硬断言 + metadata 双查）
  Step 6   Persist → kb_index.json 追加 + 日志落盘
        ↓
飞书 IM 简报（≤350 字）+ 飞书云文档详报（8 章）
```

### 版本演进

| 版本 | 架构模式 | 特点 |
|---|---|---|
| **V4** | 多 Agent 协作（Orchestrator + 4 子 Agent + Critic） | 每 Agent 独立进程，webhook 常驻，分 4 Stage 实施 |
| **V5** | OpenClaw 原生 Skill | Claude 本人主控，Python 仅作工具库，与论文助手同构，无独立进程 |

---

## 文件结构

```
.
├── 财报研究Agent-V4-Final.md          # V4 完整实施文档（含代码）
├── 财报研究Agent-V5-OpenClaw原生版.md  # V5 架构设计
├── 财报研究Agent-架构设计V2~V4.md      # 架构演进过程
├── V4-stage1-H1-scaffolding.md        # V4 Stage1：脚手架
├── V4-stage2-H2-fetch.md              # V4 Stage2：Fetcher + Knowledge
├── V4-stage3-H3-analysis.md           # V4 Stage3：Parser + Analyst + Critic
├── V4-stage4-H4-delivery.md           # V4 Stage4：飞书写入 + 端到端
├── V5-stage1-注册与骨架.md             # V5 Stage1：AGENTS.md 注册
├── V5-stage2-工具库统一入口.md         # V5 Stage2：tools.py 统一入口
├── V5-stage3-端到端联调.md             # V5 Stage3：真实触发验证
└── finreport-prep.sh                  # 环境初始化脚本（secrets 需手动填写）
```

---

## 快速开始

### 前置要求

- 腾讯云 OpenClaw 实例，工作目录 `/root/.openclaw/workspace/`
- 飞书机器人（app_id / app_secret / 云文档写权限）
- Python 3.10+

### 1. 初始化环境

```bash
bash finreport-prep.sh
```

脚本会创建目录骨架并生成 secrets 模板文件，**然后手动编辑**：

```bash
vim /root/.openclaw/secrets/finreport.env
```

填写以下变量：

```env
FINREPORT_FEISHU_APP_ID=your_feishu_app_id
FINREPORT_FEISHU_APP_SECRET=your_feishu_app_secret
FINREPORT_FEISHU_TARGET_FOLDER_TOKEN=your_folder_token
FINREPORT_SEC_USER_AGENT_EMAIL=your-email@example.com
```

> ⚠️ secrets 文件权限必须为 `chmod 600`，且**绝不能进入 workspace 目录**。

### 2. 安装依赖

```bash
pip install requests pdfplumber camelot-py lark-oapi pydantic pyyaml python-dotenv
```

### 3. 按 Stage 实施

参考各 Stage 文档逐步推进，每个 Stage 结束后运行基线校验：

```bash
sha256sum -c /root/.openclaw/secrets/finreport_baseline.sha256
```

### 4. 触发使用

在微信 ClawBot 中直接说：

```
帮我研究苹果 2024Q4 的财报
AAPL 2024Q4 财报
重新分析 MSFT 2024Q4
```

---

## 安全设计

- **secrets 外置**：所有凭证存于 `/root/.openclaw/secrets/`，chmod 600，禁止进入 workspace
- **下载白名单**：Fetcher 只接受 `sec.gov`，每次重定向后重新校验域名
- **防 prompt 注入**：外部财报数据全部用 `<external_data>` 标签包裹后喂给 Analyst
- **Python 工具不调 LLM**：分析逻辑全部由 Claude 主推理链完成，防二次注入
- **禁动文件基线**：`sha256sum` 校验论文助手核心文件，Stage 间强制比对

---

## 支持的公司（MVP）

| 公司 | Ticker | 赛道 |
|---|---|---|
| Apple Inc. | AAPL | consumer_electronics |
| Microsoft | MSFT | software |
| NVIDIA | NVDA | semiconductor |
| Tesla | TSLA | ev_automotive |

新增公司请编辑 `config/finreport_companies.yaml`。

---

## License

MIT
