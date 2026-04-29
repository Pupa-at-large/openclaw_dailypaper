# Stage 1 · H1 脚手架（1 小时）

> 这是 V4-Final 分阶段实施的第一阶段，单独喂给腾讯云 Claude。
> 本阶段结束后**必须停下**，等用户人工 review 后才进入 Stage 2。

---

## 【最高优先级】开始前必读

### 你是谁
你运行在腾讯云 OpenClaw Agent 中，`/root/.openclaw/workspace/` 是现有"论文助手"的家目录。本次任务是在**不破坏论文助手任何功能**的前提下追加一个"财报研究助手"能力模块。

### 安全铁律（5 条，任何一条违反立即停止）
1. **严禁将 secrets（飞书 app_id/app_secret/verify_token、SEC 邮箱）以任何形式写入 workspace 任何文件**。secrets 只存在于 `/root/.openclaw/secrets/finreport.env`，你不得读取、打印、复制此文件内容。
2. **严禁修改禁动清单中的任何文件**（见下表）。
3. 凭证只能通过 `config_loader.load_env()` 走环境变量加载。
4. 所有新建代码/配置不得出现硬编码 secrets。
5. 本 Stage 结束必须跑 sha256 基线校验，有 FAIL 立即停止并报告。

### 禁动清单
| 类别 | 禁动内容 |
|---|---|
| 根级 MD | `AGENTS.md` `SOUL.md` `IDENTITY.md` `CONFIG.md` `TOOLS.md` `HEARTBEAT.md` `USER.md` `MEMORY.md` |
| config/ | `companies.yaml` `notion.yaml` `search-strategy.md` |
| skills/ | `PAPER_ANALYSIS_SKILL.md` `notion-api-skill/` `openclaw-workspace/` |
| memory/ | `papers/` `analysis/` `logs/` 以及所有 `2026-*.md` |
| reports/ | 现有所有论文报告文件 |
| shared/ | `papers/` `templates/` |
| users/ | 整个目录 |
| docs/ | 现有所有文件 |
| .openclaw/ | 整个隐藏目录 |

### 开始前必须按顺序读的现有文件
```
1. /root/.openclaw/workspace/AGENTS.md
2. /root/.openclaw/workspace/SOUL.md
3. /root/.openclaw/workspace/IDENTITY.md
4. /root/.openclaw/workspace/CONFIG.md
5. /root/.openclaw/workspace/TOOLS.md
6. /root/.openclaw/workspace/skills/PAPER_ANALYSIS_SKILL.md
7. /root/.openclaw/workspace/docs/delivery-format.md
8. /root/.openclaw/workspace/docs/tencent-docs-sop.md
9. /root/.openclaw/workspace/docs/reporting-protocol.md
10. /root/.openclaw/workspace/docs/verification-protocol.md
11. /root/.openclaw/workspace/config/companies.yaml（只读）
```
读完后告诉用户：论文助手的飞书写入方式、IM 风格、`reports/` 命名规范。**本文档与现有文件冲突时以现有文件为准**。

### 前置条件检查
用户应该已经跑过 `finreport-prep.sh`。开始前先确认：
```bash
test -f /root/.openclaw/secrets/finreport.env || echo "FAIL: 用户未完成前置准备"
test -f /root/.openclaw/secrets/finreport_baseline.sha256 || echo "FAIL: sha256 基线不存在"
git -C /root/.openclaw/workspace tag -l baseline-before-finreport | grep -q . || echo "FAIL: git tag 不存在"
```
任一 FAIL → 停止，告诉用户先跑 `finreport-prep.sh`。

---

## H1 具体任务

### 任务 1 · 安装 Python 依赖
```bash
pip3 install requests pdfplumber camelot-py[cv] lark-oapi pydantic pyyaml python-dotenv
```

### 任务 2 · 创建 `config/finreport.yaml`（不含 secrets）

```yaml
version: "1.0"
agent_name: "finreport"

feishu:
  env_prefix: "FINREPORT_FEISHU_"
  target_folder_token_env: "FINREPORT_FEISHU_TARGET_FOLDER_TOKEN"

sec:
  base_url: "https://www.sec.gov"
  user_agent_template: "Finreport Agent ${FINREPORT_SEC_USER_AGENT_EMAIL}"
  rate_limit_seconds: 0.15
  request_timeout: 60
  max_redirects: 3
  max_file_size_mb: 50
  allowed_content_types:
    - "application/pdf"
    - "text/html"
    - "application/xhtml+xml"

whitelist_domains:
  - "www.sec.gov"
  - "sec.gov"

budgets:
  total_task_timeout_seconds: 600
  fetcher_timeout_seconds: 120
  parser_timeout_seconds: 180
  analyst_timeout_seconds: 240
  max_clarification_questions: 3
  heartbeat_interval_seconds: 60

critic_thresholds:
  gross_margin_min: -0.5
  gross_margin_max: 1.0
  roe_min: -2.0
  roe_max: 2.0
  debt_ratio_min: 0.0
  debt_ratio_max: 1.5

paths:
  kb_index: "memory/finreport/kb_index.json"
  kb_index_lock: "memory/finreport/.kb_index.lock"
  scan_cache: "memory/finreport/scan_cache.json"
  logs_dir: "memory/finreport/logs"
  pdf_root: "shared/finreports"
  reports_dir: "reports"

security:
  secrets_env_file: "/root/.openclaw/secrets/finreport.env"
  baseline_file: "/root/.openclaw/secrets/finreport_baseline.sha256"
  log_email_redact: true
```

### 任务 3 · 创建 `config/finreport_companies.yaml`

```yaml
version: "1.0"
companies:
  - name: "Apple Inc."
    ticker: "AAPL"
    cik: "0000320193"
    industry: "consumer_electronics"
    fiscal_year_end: "09"
    exchange: "NASDAQ"
  - name: "Microsoft Corporation"
    ticker: "MSFT"
    cik: "0000789019"
    industry: "software"
    fiscal_year_end: "06"
    exchange: "NASDAQ"
  - name: "NVIDIA Corporation"
    ticker: "NVDA"
    cik: "0001045810"
    industry: "semiconductor"
    fiscal_year_end: "01"
    exchange: "NASDAQ"
  - name: "Tesla, Inc."
    ticker: "TSLA"
    cik: "0001318605"
    industry: "ev_automotive"
    fiscal_year_end: "12"
    exchange: "NASDAQ"
```

### 任务 4 · 创建 `finreport_agent/` Python 骨架

所有文件先创建为空骨架（只有 `pass` 或 TODO 注释），Stage 2-4 再填充。

```
finreport_agent/
├── __init__.py                  （空）
├── main.py                      飞书 webhook 入口（只实现 hello 回声）
├── orchestrator.py              骨架类
├── knowledge.py                 骨架类（Stage 2 实现）
├── fetcher.py                   骨架类（Stage 2 实现）
├── parser.py                    骨架类（Stage 3 实现）
├── analyst.py                   骨架类（Stage 3 实现）
├── critic.py                    骨架类（Stage 3 实现）
├── feishu_client.py             骨架类（Stage 4 实现）
├── link_validator.py            骨架类（Stage 3 实现）
├── config_loader.py             【本 Stage 实现】
├── prompts/
│   ├── analyst_system.md        （空占位，Stage 3 填）
│   └── orchestrator_system.md   （空占位）
└── tests/
    ├── __init__.py
    └── check_baseline.py        【本 Stage 实现】
```

### 任务 5 · 实现 `config_loader.py`

```python
"""
安全加载 secrets。严禁读取/打印/复制 secrets 内容。
"""
import os
import subprocess
from pathlib import Path
from dotenv import load_dotenv

SECRETS_PATH = Path("/root/.openclaw/secrets/finreport.env")
WORKSPACE = Path("/root/.openclaw/workspace")

REQUIRED_VARS = [
    "FINREPORT_FEISHU_APP_ID",
    "FINREPORT_FEISHU_APP_SECRET",
    "FINREPORT_FEISHU_VERIFY_TOKEN",
    "FINREPORT_FEISHU_TARGET_FOLDER_TOKEN",
    "FINREPORT_SEC_USER_AGENT_EMAIL",
]

def load_env():
    if not SECRETS_PATH.exists():
        raise RuntimeError(f"secrets file not found: {SECRETS_PATH}")
    mode = oct(SECRETS_PATH.stat().st_mode)[-3:]
    if mode != "600":
        raise RuntimeError(f"secrets file must be chmod 600, got {mode}")
    load_dotenv(SECRETS_PATH, override=False)
    missing = [v for v in REQUIRED_VARS if not os.environ.get(v)]
    if missing:
        raise RuntimeError(f"missing env vars: {missing}")
    _self_check_no_secret_leak()

def _self_check_no_secret_leak():
    """启动自检：workspace 中不得出现 app_secret"""
    secret = os.environ.get("FINREPORT_FEISHU_APP_SECRET", "")
    if not secret or secret.startswith("TODO"):
        return
    result = subprocess.run(
        ["grep", "-rq", secret, str(WORKSPACE)],
        capture_output=True
    )
    if result.returncode == 0:
        raise RuntimeError("FATAL: app_secret leaked into workspace!")

def redact_email(s: str) -> str:
    """日志写入前 redact 邮箱"""
    import re
    return re.sub(r'[\w.+-]+@[\w-]+\.[\w.-]+', '***@***.***', s)
```

### 任务 6 · 实现 `main.py`（飞书 hello 回声）

实现一个最小的飞书 webhook 监听器，只做 "hello" → 回声 "hello from finreport agent"。参考现有论文助手在 `docs/tencent-docs-sop.md` 和 `AGENTS.md` 中的飞书接入方式（读完前置文件后你应该知道怎么做）。

关键约束：
- 启动时先调 `config_loader.load_env()`
- 所有日志调用 `config_loader.redact_email()` 后再写入
- 不持久化任何 secrets 到磁盘

### 任务 7 · 实现 `tests/check_baseline.py`

```python
"""禁动文件 sha256 基线校验。每个 Stage 结束必跑。"""
import subprocess
import sys

BASELINE = "/root/.openclaw/secrets/finreport_baseline.sha256"
WORKSPACE = "/root/.openclaw/workspace"

def main():
    result = subprocess.run(
        ["sha256sum", "-c", BASELINE, "--quiet"],
        cwd=WORKSPACE,
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        print("FAIL: 禁动文件基线校验失败")
        print(result.stdout)
        print(result.stderr)
        sys.exit(1)
    print("OK: 所有禁动文件 sha256 一致")

if __name__ == "__main__":
    main()
```

---

## H1 验收标准（全部 ✅ 才能进入 Stage 2）

- [ ] 任务 1-7 全部完成
- [ ] `config/finreport.yaml` 不含任何 secrets 明文（跑 `grep -E "app_id|app_secret|verify_token" config/finreport.yaml` 无输出）
- [ ] `finreport_agent/` 目录骨架完整
- [ ] `python3 -c "from finreport_agent.config_loader import load_env; load_env()"` 成功加载
- [ ] `python3 finreport_agent/tests/check_baseline.py` 输出 OK
- [ ] 在飞书群发 "hello" → 收到 "hello from finreport agent"
- [ ] `git -C /root/.openclaw/workspace status` 的改动清单**不包含禁动清单里的任何文件**

---

## H1 结束后

**立即停下**，向用户汇报：
1. 任务完成情况
2. 验收结果（每一项 ✅ 或 ❌）
3. 遇到的问题或与本文档的 delta
4. 等待用户指令进入 Stage 2

**不要主动开始 Stage 2**。
