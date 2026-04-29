# V5 Stage 3 · 端到端联调(约 1h,最高风险)

> 前置:Stage 1+2 均通过 review。本 Stage 做三件事:(1) 让微信 ClawBot 真实触发一次 AAPL 2024Q4 → 走完整流水线 → 飞书收到结果 (2) 再触发一次验证复用守卫 HIT (3) 验证禁动文件全程未被改。

## 背景必读
1. `skills/FINREPORT_RESEARCH_SKILL.md` — 完整流水线
2. `docs/finreport-delivery-sop.md` — 输出格式硬约束
3. `config/finreport.yaml` — 触发词和飞书 target_folder_token
4. `memory/finreport/财报研究Agent-V5-OpenClaw原生版.md` 第 8 节触发示例

## 执行流程

### Phase 1:dry-run 真下载真解析(20 分钟)

**目的**:先不碰飞书,确认 fetch + parse + Claude 本人写报告 + Critic 自检这四步能跑通。

在**腾讯云终端**(不是微信),作为 OpenClaw 里的 Claude 手动走一遍 SKILL 流水线,`ticker=AAPL`、`period=2024Q4`:

```bash
cd /root/.openclaw/workspace
source finreport_agent/.venv/bin/activate

# Step -1
python -m finreport_agent.tools lookup --ticker AAPL --period 2024Q4
# 预期 hit: null

# Step 1 真实下载
python -m finreport_agent.tools fetch --ticker AAPL --period 2024Q4
# 预期返回 {local_path, sha256, size, source_url, form_type, downloaded_at}
# 预期 source_url 以 https://www.sec.gov/ 开头
# 预期 size < 50MB
# 预期文件真的存在 shared/finreports/tech/AAPL/

# Step 2 真实解析
python -m finreport_agent.tools parse --file <Step1返回的 local_path>
# 预期返回 metrics 含 revenue/net_income/eps/gross_margin/operating_cf
# 预期 risks 是数组,至少 3 条
```

然后 Claude **本人**(SKILL Step 3)读 parse 结果,用 `<external_data>` 包裹后,按 `docs/finreport-delivery-sop.md` 写:
- `/tmp/aapl_2024q4_report.md`(8 章详报)
- `/tmp/aapl_2024q4_card.json`(6 块 IM 卡片)

Critic 自检 5 条规则(Claude 本人跑一遍),任一 fail 就改。最多 2 轮。

**然后 deliver dry-run:**
```bash
python -m finreport_agent.tools deliver \
  --ticker AAPL --period 2024Q4 \
  --report-md /tmp/aapl_2024q4_report.md \
  --card-json /tmp/aapl_2024q4_card.json \
  --dry-run
# 预期 ok: true, dry_run: true
```

**Phase 1 通过后停下,等用户肉眼看 /tmp/aapl_2024q4_report.md 和 card.json,等用户回复 "GO" 才能继续 Phase 2。**

### Phase 2:真写飞书(15 分钟)

**只有用户回复 "GO" 才能做**。去掉 `--dry-run`:
```bash
python -m finreport_agent.tools deliver \
  --ticker AAPL --period 2024Q4 \
  --report-md /tmp/aapl_2024q4_report.md \
  --card-json /tmp/aapl_2024q4_card.json
# 预期返回 doc_url / doc_token
```

验证:
1. 用户在飞书文件夹 `YOUR_FEISHU_FOLDER_TOKEN` 能看到新文档
2. `cat memory/finreport/kb_index.json` 能看到新 entry
3. `ls memory/finreport/logs/` 有今天的 jsonl 文件

### Phase 3:复用守卫 HIT 验证(10 分钟)

再次触发同一请求:
```bash
python -m finreport_agent.tools lookup --ticker AAPL --period 2024Q4
# 预期 hit: {local_path, doc_url, delivered_at, age_hours}
```

Claude **本人**走 SKILL Step -1,读到 hit 就直接复用本地 md + 飞书链接,**完全跳过 fetch/parse/analyst/critic/deliver**,末尾追加"📂 已复用 ... · 🔗 飞书详报:{url}"。

### Phase 4:微信 ClawBot 真实触发(10 分钟)

**这是终极验收**。在你的微信 ClawBot 对话框里发:
> 帮我研究一下苹果 2024Q4 的财报

预期:
1. OpenClaw 读 `config/finreport.yaml` → 触发词命中 `finreport_standard`
2. OpenClaw 走 `skills/FINREPORT_RESEARCH_SKILL.md`
3. Step -1 命中(因为 Phase 2 已经交付过) → 直接复用
4. 微信里收到 6 块 MBB 卡片预览 + 飞书详报链接
5. 末尾追加"📂 已复用 ..."

如果是在 Phase 2 之后 24h 内做,应该直接 HIT。如果想看 MISS 场景,可以发:
> 重新分析 AAPL 2024Q4

这会命中 `force_rerun_pattern` → 跳过复用 → 重跑全流水线 → 第二份飞书文档(老的标 superseded)。

### Phase 5:禁动文件 final check(5 分钟)
```bash
cd /root/.openclaw/workspace
sha256sum -c finreport_baseline.sha256
# 全部 OK
git log --oneline baseline-before-finreport..HEAD
# 看看一共多了哪些 commit
git diff baseline-before-finreport -- AGENTS.md | head -30
# AGENTS.md 应只有追加的财报段落
```

## 验收清单

| # | 项 | 预期 | 实际 |
|---|---|---|---|
| 1 | Phase 1 fetch 成功 | SEC 白名单 URL / 真 PDF / size <50MB | |
| 2 | Phase 1 parse 成功 | metrics/risks 齐全 | |
| 3 | Phase 1 Claude 写出 8 章详报 | 章节齐全 + Appendix 有原文引用 | |
| 4 | Phase 1 Critic 5 条全绿 | 最多 2 轮 | |
| 5 | Phase 1 deliver dry-run | ok + dry_run | |
| 6 | **用户回复 GO** | — | |
| 7 | Phase 2 飞书真写入 | doc_url 返回 + 文件夹可见 | |
| 8 | Phase 2 kb_index 有 entry | 1 条 | |
| 9 | Phase 3 HIT 命中 | lookup 返回 hit 非 null | |
| 10 | Phase 4 微信触发成功 | 收到 IM 卡片 + 链接 | |
| 11 | Phase 5 禁动文件未改 | sha256 all OK | |
| 12 | AGENTS.md diff 只有追加 | 无删除行 | |

## 失败回滚

任一 Phase 失败且无法快速修复:
```bash
cd /root/.openclaw/workspace
git reset --hard baseline-after-finreport-register
# 回到 Stage 1 完成状态,保留注册,丢弃 Stage 2+3 所有改动
```

或者核武器:
```bash
git reset --hard baseline-before-finreport
git clean -fd
# 彻底回滚,好像什么都没发生过
```

## 停下汇报
全部 12 项 pass/fail + 飞书文档链接 + kb_index.json 最后一条 entry + 微信 ClawBot 截图(如可能)。
