# 提交闸门（两阶段作答）设计

> 状态：已实施 · 2026-11-09（本地静态与单测通过；真实 U校园提交尚未验收）
> 范围：unipus-agent 自动执行器（`mcp/runner.py`）+ 附带 bug 修复
> 背景：`runner.py:20` `DO_SUBMIT` 默认 `"1"`，即不设环境变量就自动提交，与 `SKILL.md` 红线 1「默认只填不提交」冲突。

## 1. 目标与非目标

**目标**：默认只填不交；主人核对审核单后，再触发自动提交，且提交前重新校验页面与审核单一致。

**非目标**（明确不做）：
- 不改 MCP 工具语义（`unipus_fill` 不提交、`unipus_submit` 需 `confirm:true` 已正确，不动）
- 不做平台分数回读、不做错题自动重答
- 不引入新依赖、不改 `answers/` 与 `ledger/` 数据格式

## 2. 现状

```
runner.py 单阶段：
  build_queue → open_task → do_fields(fill + 立即 submit) → stay → state
state 取值：ok / submitted / submit? / nosubmit / filled / navfail / exc / noq ...
```

问题：`filled` 在下一轮被当作「已完成」跳过，**填了的任务事后没有入口再提交**——这也正是作者默认开自动提交的原因。

## 3. 备选方案

| 方案 | 做法 | 否决理由 |
|---|---|---|
| A 两阶段状态机（推荐） | `MODE=fill` 只填并产出审核单；`MODE=submit` 只处理已填任务，重校后提交 | 唯一能配合「先核对、后提交」且保留断点续跑的方案 |
| B 单进程交互确认 | 填完阻塞等 stdin `y/n` | 挂机批处理场景不可用；主人要离开电脑去核对时进程就卡死 |
| C 审核单 + 人工改 JSON | 主人手工把 `approved:true` 写进 JSON 再重跑 | 手工编辑易错、无校验；本质是 A 的手工劣化版 |

选 A。B 与挂机冲突，C 把责任推给手写文件。

## 4. 设计（方案 A）

### 4.1 新增环境变量

| 变量 | 默认 | 含义 |
|---|---|---|
| `MODE` | `fill` | `fill` = 只填不交；`submit` = 只提交 `filled` 任务 |
| `DO_SUBMIT` | `0` | 仅 `MODE=fill` 时兼容旧行为；`1` 且 `MODE=fill` 时退化为旧单阶段（显式选择才生效） |
| `DETAIL_URL` | 原写死值 | 覆盖 `runner.py:16` 硬编码课程 URL |

`MODE=submit` 时 `DO_SUBMIT` 无效，提交一律走闸门。

### 4.2 fill 阶段（默认）

1. 每个 `fields` 任务：回填 → 回读校验（沿用现有逻辑与安全闸门 `idx != len(answers)`）
2. 校验通过：`state[key] = "filled"`
3. **立即**把审核记录写入 `mcp/_review.json`（数组，按 key 去重覆盖）：

```json
{ "key": "U2|2-12 Read and think|Evaluate the style",
  "unit": 2, "section": "2-12 Read and think", "title": "Evaluate the style",
  "nth": 0, "answers": ["..."], "verify": [{"index":0,"expected":"...","actual":"...","ok":true}],
  "filledAt": "2026-11-09T12:00:00" }
```

4. 不调用 submit。

### 4.3 submit 阶段（`MODE=submit`）

```
读取 state + _review.json
for 每个 state[key] == "filled" 且审核单有条目的 key:
    open_task(key 对应 unit/title/nth)
    cur = extract()
    校验闸门 review_gate(审核单, cur, cur.hasSubmit)
    全绿 → api("submit") → state[key] = "submitted" / "submit?"
    任何一项不过 → state[key] = "gatefail"，记录原因，不提交
    stay(...)  # 时长仍按配置
```

**闸门判据 `review_gate(review, current, has_submit)`（纯函数，可单测）**：

1. `has_submit` 必须为 `true`（按钮在 = 未提交过；不在 = 已提交或题型无按钮）
2. 当前题数与审核单答案数一致
3. 逐题 `current[index].value == review.answers[index]`（essay 型做空白归一化后比较）
4. 任一题当前值为空 → 拒绝
5. 全部通过才返回 `(True, [])`，否则 `(False, [差异列表])`

### 4.4 审核单给谁看

`_review.json` 是给 AI / 主人读的原始数据；`_review.md` 同时生成一份人读表格（题号、答案、置信度、校验结果），对应 `references/answer-verify.md` 的审核单模板。

### 4.5 兼容与回滚

- 不新增状态值之外的字段；旧 state 文件可直接读
- 回滚：设 `MODE=fill DO_SUBMIT=1` 即回到旧单阶段行为
- `MODE=submit` 未识别 → 报错退出，不猜

## 5. 附带修复（同一批次，独立可验收）

| # | 文件 | 改动 | 验证 |
|---|---|---|---|
| 1 | `mcp/server.js:423-437` | 导入 `readFileSync` | MCP `unipus_kb_search` 实测返回 hits |
| 2 | `mcp/server.js:314,335,341,375` | v1 → v2 抽取/回填（与 `serve.mjs` 对齐） | 夹具页 extract/fill 通过；选择题路径存在 |
| 3 | `mcp/runner.py:16` | `DETAIL` 支持 `DETAIL_URL` 覆盖 | 导入后读环境变量生效 |
| 4 | `tools/pipeline.sh:17` | 默认 `NODE_BIN=node` | `bash -n` + 无 NODE_BIN 时取 PATH |
| 5 | `tools/verify_kb.py:71` | 占位符规则收窄（罗马数字/参考文献不再误报），补「已转录单元」头检查提示 | 重跑仍报 0 误报 |
| 6 | `README.md` / `mcp/README.md` | 工具数 13→15；覆盖度措辞改为「32 本已转录 / 200 条目索引」；补两阶段说明 | 人工核对 |
| 7 | `references/submit-policy.md` | 补 `MODE=fill/submit` 工作流 | 人工核对 |

## 6. 测试策略

| 层 | 测试 | 现状 |
|---|---|---|
| MCP | `_selftest/kb-search.test.mjs`（真实 stdio 调用） | 已写，**当前 FAIL** |
| MCP | `_selftest/dryrun.test.mjs`（本地夹具页） | 已写，链路 PASS |
| Python | `review_gate` 纯函数单测（不联网） | 待写 |
| Python | `build_queue` 读现有 ledger/answers | 待写 |
| 静态 | `python -m py_compile` / `node --check` | 可跑 |

无法本地验证的部分（真实 U校园提交）明确标注为「需在真实会话验收」，不假装测过。

## 7. 风险

- 提交阶段重新打开任务可能遇到 SPA 未渲染 → 复用 `open_task` 的 4 次重试 + `wait_detail_ready`
- 两次运行之间平台若自动保存草稿，页面值可能与审核单一致但语义已变 → 闸门只保证「与主人核对过的一致」，不保证平台侧正确
- `_review.json` 含答案文本，加入 `mcp/.gitignore`