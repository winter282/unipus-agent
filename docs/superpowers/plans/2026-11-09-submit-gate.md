# 提交闸门 + 附带修复 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 默认只填不提交，主人在核对审核单后通过 `MODE=submit` 触发受闸门保护的自动提交；同时修复 MCP 服务端 2 个 P0 bug 与 5 项工程缺陷。

**Architecture:** runner.py 从单阶段改为 `MODE=fill|submit` 两阶段状态机；提交判定抽成纯函数 `review_gate()`；MCP 服务端对齐 `serve.mjs` 的 v2 脚本。

**Tech Stack:** Node 22 + `@modelcontextprotocol/sdk` + `playwright-core`；Python 3.10+（仅标准库）。

**Spec:** `docs/superpowers/specs/2026-11-09-submit-gate-design.md`

**执行记录（2026-11-09 复核）:** 代码已落地；静态检查、`review_gate` 6 用例、`kb-search` stdio 冒烟、`build_queue`（119 tasks）均通过。`_selftest/` 按计划删除，`dryrun` 未复跑；真实 U校园提交未验收。

## Global Constraints

- 不新增第三方依赖
- 不修改 `mcp/answers/`、`mcp/ledger/` 的数据格式
- `MODE` 未识别必须报错退出，不猜
- 所有提交路径必须经过 `review_gate`；`unipus_submit` 行为不变
- 临时测试文件统一放 `mcp/_selftest/`，收尾删除

## Review Focus

- 旧 state 文件里已有 `submitted` 的任务，`MODE=submit` 不得重复提交
- 页面已提交（无提交按钮）时闸门必须拒绝
- `_review.json` 不存在或条目缺失时必须跳过而非崩溃
- essay 答案的空白差异不得误判为不一致
- 真实 U校园会话无法本地验证的部分，必须标注而非假装测过

---

### Task 1: 修复 `unipus_kb_search` 的 `readFileSync` 未导入

**Files:**
- Modify: `mcp/server.js:423-437`
- Test: `mcp/_selftest/kb-search.test.mjs`

**Interfaces:**
- Produces: `unipus_kb_search({query, book?, maxLines?}) -> {total, hits[]}` 可正常返回

- [x] **Step 1: 确认测试当前失败** — `node _selftest/kb-search.test.mjs` → `readFileSync is not defined`
- [x] **Step 2: 把 `readFileSync` 加入该工具的动态导入**
- [x] **Step 3: 重跑测试** → `RESULT: PASS`

### Task 2: MCP 服务端对齐 v2 脚本

**Files:**
- Modify: `mcp/server.js:314,335,341,375`
- Test: `mcp/_selftest/dryrun.test.mjs`

**Interfaces:**
- Consumes: `scripts/extract_questions_v2.js`、`scripts/fill_answers_v2.js`（payload `{index, answer, dryRun}`，返回含 `kind`）

- [x] **Step 1: 把 `unipus_extract` / `unipus_fill` / `unipus_submit` 三处 `scriptFn` 换成 `_v2`**
- [ ] **Step 2: 夹具页测试通过，且返回含 `choiceCount`**（未复跑：测试文件已按收尾要求删除；代码层已确认三处 `scriptFn` 指向 v2）

### Task 3: `review_gate()` 纯函数 + 单测

**Files:**
- Modify: `mcp/runner.py`（新增函数）
- Test: `mcp/_selftest/test_review_gate.py`

**Interfaces:**
- Produces: `review_gate(answers: list[str], current: list[dict], has_submit: bool) -> tuple[bool, list[str]]`
  - `current[i]` 形如 `{"index":0,"type":"blank","value":"the"}`

- [x] **Step 1: 写 5 个用例（全绿 / 值不符 / 空值 / 题数不符 / 无提交按钮）**
- [x] **Step 2: 跑测试确认失败（函数不存在）**
- [x] **Step 3: 实现 `review_gate`**
- [x] **Step 4: 跑测试通过**

### Task 4: runner.py 两阶段改造

**Files:**
- Modify: `mcp/runner.py:16-24, 189-253, 255-264, 333-403`
- Test: `mcp/_selftest/test_queue.py`（`build_queue` 行为不变）

**Interfaces:**
- Consumes: `review_gate`
- Produces: `MODE` / `DO_SUBMIT`（默认 0）/ `DETAIL_URL`；`_review.json`、`_review.md`

- [x] **Step 1: `DO_SUBMIT` 默认改 `"0"`；新增 `MODE`（默认 `fill`）、`DETAIL_URL`**
- [x] **Step 2: fill 阶段去掉提交分支，写审核单；state 记 `filled`**
- [x] **Step 3: 新增 `submit_reviewed()`，逐条过闸门**
- [x] **Step 4: `__main__` 按 MODE 分派；未知 MODE 报错退出**
- [x] **Step 5: `python -m py_compile` + 单测通过**

### Task 5: 工程缺陷修复

**Files:**
- Modify: `tools/pipeline.sh:17`、`tools/verify_kb.py:71`、`mcp/.gitignore`

- [x] **Step 1: `NODE_BIN` 默认改为 `node`**
- [x] **Step 2: 占位符正则排除罗马数字与参考文献；对「缺已转录单元头」给出可操作提示**
- [x] **Step 3: `_review.json` / `_review.md` 加入 `.gitignore`**
- [x] **Step 4: 重跑 `verify_kb.py`，确认误报消失**

### Task 6: 文档纠错

**Files:**
- Modify: `README.md`、`mcp/README.md`、`references/submit-policy.md`、`SKILL.md`

- [x] **Step 1: 工具数 13→15（含 `unipus_click`），说明两条运行路径**
- [x] **Step 2: 覆盖度措辞改为「32 本已转录 / 200 条目索引」**
- [x] **Step 3: `submit-policy.md` 补两阶段工作流**
- [x] **Step 4: `SKILL.md` 红线 1 补「自动执行器同样适用」**

### Task 7: 全量验证与清理

- [x] `node _selftest/kb-search.test.mjs` → PASS（2026-11-09 复跑：total=67 / 3 hits）
- [ ] `node _selftest/dryrun.test.mjs` → PASS（未复跑：需夹具页；原文件已删除）
- [x] `python _selftest/test_review_gate.py` → PASS（2026-11-09 复跑：6 用例全过）
- [x] `python -m py_compile mcp/runner.py tools/verify_kb.py` → 0 error
- [x] `node --check` 全部 JS → 0 error
- [x] `python tools/verify_kb.py` → 0 误报
- [x] 删除 `mcp/_selftest/`，保留 design/plan 文档