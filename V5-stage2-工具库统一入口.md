# V5 Stage 2 · 工具库统一入口 + dry-run 联调(约 1h)

> 前置:Stage 1 已通过 review,AGENTS.md 注册完成,SKILL.md + yaml SSOT 就位。本 Stage 只做一件事——把 V4 已经实装的 fetcher / parser / knowledge / feishu_client 用一个**统一入口 `finreport_agent/tools.py`** 暴露出来,供 SKILL 里 bash 调用。

## 背景必读
1. `memory/finreport/财报研究Agent-V5-OpenClaw原生版.md` 第 1 节文件清单
2. `skills/FINREPORT_RESEARCH_SKILL.md` Step 1 / 2 / 5 — 看 bash 调用格式
3. `finreport_agent/fetcher.py`、`parser.py`、`knowledge.py`、`feishu_client.py` — 了解现有函数签名

## 禁动清单
同 Stage 1 + 新增:
- `config/finreport.yaml` / `config/finreport_companies.yaml` — Stage 1 已定稿,不再改
- `skills/FINREPORT_RESEARCH_SKILL.md` — Stage 1 已定稿,不再改
- `AGENTS.md` — 已注册,本 Stage 完全不动

## 任务清单

### T1. 写 finreport_agent/tools.py(30 分钟)

统一入口,用 argparse 暴露 4 个子命令:`fetch / parse / deliver / lookup`。每个子命令都返回 JSON 到 stdout(Claude 读 stdout 消费)。

```python
# finreport_agent/tools.py
"""V5 统一工具入口 · 被 SKILL.md 里 bash 调用,不是人类 CLI。"""
import argparse, json, sys, traceback
from pathlib import Path

from finreport_agent.config_loader import load_env, load_yaml_config
from finreport_agent import fetcher, parser as pdf_parser, knowledge, feishu_client

def _out(obj):
    """统一输出,Claude 会从 stdout 解析 JSON。"""
    print(json.dumps(obj, ensure_ascii=False, default=str))

def _err(msg, code=1):
    _out({"ok": False, "error": msg})
    sys.exit(code)

def cmd_lookup(args):
    """Step -1 复用守卫:查 kb_index.json。"""
    hit = knowledge.lookup(args.ticker, args.period, args.report_type or "finreport_standard")
    _out({"ok": True, "hit": hit})   # hit 为 None 或 {local_path, doc_url, delivered_at, age_hours}

def cmd_fetch(args):
    load_env()
    cfg = load_yaml_config("config/finreport.yaml")
    companies = load_yaml_config("config/finreport_companies.yaml")["companies"]
    if args.ticker not in companies:
        _err(f"{args.ticker} not in whitelist")
    meta = fetcher.fetch_filing(
        ticker=args.ticker,
        period=args.period,
        cfg=cfg,
        company=companies[args.ticker],
    )
    _out({"ok": True, **meta})   # {local_path, sha256, size, source_url, form_type, downloaded_at}

def cmd_parse(args):
    result = pdf_parser.parse_filing(args.file)
    _out({"ok": True, **result})

def cmd_deliver(args):
    load_env()
    cfg = load_yaml_config("config/finreport.yaml")
    # 硬断言
    expected = "YOUR_FEISHU_FOLDER_TOKEN"
    actual = cfg["feishu"]["target_folder_token"]
    if actual != expected:
        _err(f"target_folder_token mismatch: {actual} != {expected}")

    report_md = Path(args.report_md).read_text(encoding="utf-8")
    card_json = json.loads(Path(args.card_json).read_text(encoding="utf-8"))

    if args.dry_run:
        _out({"ok": True, "dry_run": True,
              "would_write_folder": expected,
              "report_chars": len(report_md),
              "card_blocks": len(card_json.get("elements", []))})
        return

    result = feishu_client.write_report(
        folder_token=expected,
        ticker=args.ticker,
        period=args.period,
        report_md=report_md,
        card_json=card_json,
    )
    # 写后 metadata 双查
    verified = feishu_client.verify_doc(result["doc_token"], expected)
    if not verified:
        _err("post-write metadata verification failed")

    # append kb_index
    knowledge.append_entry(
        ticker=args.ticker, period=args.period,
        report_type="finreport_standard",
        local_path=str(args.report_md),
        doc_url=result["doc_url"],
        doc_token=result["doc_token"],
    )
    _out({"ok": True, **result})

def main():
    ap = argparse.ArgumentParser()
    sub = ap.add_subparsers(dest="cmd", required=True)

    p = sub.add_parser("lookup"); p.add_argument("--ticker", required=True); p.add_argument("--period", required=True); p.add_argument("--report-type")
    p.set_defaults(func=cmd_lookup)

    p = sub.add_parser("fetch"); p.add_argument("--ticker", required=True); p.add_argument("--period", required=True)
    p.set_defaults(func=cmd_fetch)

    p = sub.add_parser("parse"); p.add_argument("--file", required=True)
    p.set_defaults(func=cmd_parse)

    p = sub.add_parser("deliver")
    p.add_argument("--ticker", required=True); p.add_argument("--period", required=True)
    p.add_argument("--report-md", required=True); p.add_argument("--card-json", required=True)
    p.add_argument("--dry-run", action="store_true")
    p.set_defaults(func=cmd_deliver)

    args = ap.parse_args()
    try:
        args.func(args)
    except Exception as e:
        _err(f"{type(e).__name__}: {e}\n{traceback.format_exc()}")

if __name__ == "__main__":
    main()
```

**注意**:上面的 `fetcher.fetch_filing` / `pdf_parser.parse_filing` / `knowledge.lookup` / `knowledge.append_entry` / `feishu_client.write_report` / `feishu_client.verify_doc` 是**预期的函数签名**。如果 V4 已有代码的签名不一致,请:
1. 优先改 tools.py 适配现有函数(最小侵入)
2. 如现有函数语义不符(比如 fetcher 没有白名单断言),补打补丁,并在验收里标注

### T2. 对齐现有模块的函数签名(20 分钟)

逐个检查:

**fetcher.py** 必须有:
- `fetch_filing(ticker, period, cfg, company) -> dict`
- 内部断言:白名单域名 / Content-Type / 50MB / allow_redirects=False / SEC User-Agent
- 如缺失,补上

**parser.py** 必须有:
- `parse_filing(file_path) -> dict`
- 返回 `{ticker, period, metrics, segment_revenue, risks, raw_excerpts}`
- 有 `signal.alarm(180)` 超时保护

**knowledge.py** 必须有:
- `lookup(ticker, period, report_type) -> Optional[dict]`
- `append_entry(**kwargs) -> None`
- 使用 `fcntl.flock` 保护 `memory/finreport/kb_index.json`

**feishu_client.py** 必须有:
- `write_report(folder_token, ticker, period, report_md, card_json) -> dict`
- `verify_doc(doc_token, expected_folder) -> bool`
- 内部断言 folder_token 匹配

缺啥补啥,**但不要大改**。这个 Stage 的主目标是接上统一入口,不是重构。

### T3. dry-run 联调测试(10 分钟)

准备一份假的 report_md 和 card_json 测试 deliver 的 dry-run 路径:
```bash
cd /root/.openclaw/workspace
source finreport_agent/.venv/bin/activate

# 造假数据
echo "# AAPL 2024Q4 测试详报" > /tmp/fake_report.md
echo '{"elements":[{"tag":"div","text":"test"}]}' > /tmp/fake_card.json

# 4 个子命令烟雾测试
python -m finreport_agent.tools lookup --ticker AAPL --period 2024Q4
# → 预期 {"ok":true,"hit":null}(第一次查,未命中)

python -m finreport_agent.tools deliver \
  --ticker AAPL --period 2024Q4 \
  --report-md /tmp/fake_report.md \
  --card-json /tmp/fake_card.json \
  --dry-run
# → 预期 {"ok":true,"dry_run":true,"would_write_folder":"YOUR_FEISHU_FOLDER_TOKEN",...}
```

fetch 和 parse 的真实测试留给 Stage 3(因为会下真 PDF,放到 Stage 3 端到端跑更省时间)。

### T4. Git commit(5 分钟)
```bash
git add finreport_agent/tools.py finreport_agent/*.py
git commit -m "finreport V5 Stage 2: tools.py unified entry"
```

## 验收清单

| # | 项 | 验证 | 预期 |
|---|---|---|---|
| 1 | tools.py 存在 | `ls finreport_agent/tools.py` | 存在 |
| 2 | 4 子命令可见 | `python -m finreport_agent.tools --help` | 列出 fetch/parse/deliver/lookup |
| 3 | lookup 烟雾 | 见 T3 | 返回 `hit: null` |
| 4 | deliver dry-run | 见 T3 | `ok: true, dry_run: true` |
| 5 | target_folder_token 硬断言 | 手动把 yaml 里的 token 临时改错一个字符后再跑 deliver dry-run | 返回 error 含 "mismatch"(测完记得改回来) |
| 6 | fetcher 白名单断言存在 | `grep -n "allowed_domains\|_assert_whitelist" finreport_agent/fetcher.py` | 至少 1 处命中 |
| 7 | fetcher allow_redirects=False | `grep -n "allow_redirects" finreport_agent/fetcher.py` | 有 `False` |
| 8 | fetcher User-Agent 含 email 变量 | `grep -n "SEC_USER_AGENT_EMAIL" finreport_agent/fetcher.py` | 有命中 |
| 9 | knowledge fcntl.flock | `grep -n "fcntl\|flock" finreport_agent/knowledge.py` | 有命中 |
| 10 | 禁动文件未改 | `sha256sum -c finreport_baseline.sha256 \| grep -v OK` | 空 |

## 停下等 review
报告:
1. 10 项验收 pass/fail
2. `python -m finreport_agent.tools --help` 完整输出
3. lookup 和 deliver dry-run 的实际 stdout
4. 如果改动了 fetcher/parser 等现有模块,列 diff
