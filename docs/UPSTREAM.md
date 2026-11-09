# 上游前作调研（2026-11-09）

> 目的：接手本仓库前，先看清 README「致敬」里 5 个前作**现在**做到哪一步、哪些思路值得继续借、哪些与我们的红线冲突。
> 数据来源：各仓库 README 与关键源码（`gh api` 读取）、GitHub 元数据（star / 最近提交），均为 **2026-11-09** 快照。
> 只借鉴思路，不复制代码——其中 3 个是 GPL-3.0。

## 0. 结论先行

| 结论 | 依据 |
|---|---|
| 5 个前作里只有 **UnipusAI 仍在活跃更新**（2026-10-21），且已转向 Rust + HTTP 直连 | 仓库 pushedAt |
| **AutoUnipus 已停更**（2025-11-30，作者明示不再更新） | README |
| 我们的定位（浏览器语义操作、不逆向接口、默认不提交）与前作**不冲突**：它们解决「快」，我们解决「稳、可审计、可回滚」 | 对比第 3 节 |
| 可补的短板：题库 Provider 链 + 本地缓存、错题重做上限、冷却递增、讨论题/音视频题 | chaoxing / UnipusAI |
| **不要学**：协议逆向、第三方题库传题、默认自动提交 | 与 SKILL.md 红线 1/7 冲突 |

## 1. 项目对照表

| 项目 | 路线 | 核心能力 | 默认提交 | 许可 | 最近提交 | 对本项目 |
|---|---|---|---|---|---|---|
| [chaoxing](https://github.com/Samueli924/chaoxing) | Python + HTTP | 超星任务点、题库 Provider 链、错题重做、通知 | 是 | GPL-3.0 | 2026-09-28 | ⭐ Provider 链 / 缓存 / 重做上限 |
| [AutoUnipus](https://github.com/CXRunfree/AutoUnipus) | Python + Playwright | U校园自动/辅助双模式、仅单选 | 自动模式是 | BSD-3-Clause | 2025-11-30（停更） | 双模式与我们的 fill/submit 同构 |
| [UnipusAI](https://github.com/Zzj-klwgxdz/UnipusAI) | Rust + HTTP | 全题型 LLM 作答、讨论区、语音转写、SQLite 归档、TUI | 是 | GPL-3.0 | 2026-10-21 | ⭐ 冷却、讨论题、转写、归档 |
| [UnipusAIAutoPlayer](https://github.com/uxudjs/UnipusAIAutoPlayer) | 油猴脚本 | 目录遍历、时长均分、倒计时、视频倍速 | 不涉及答题 | GPL-3.0 | 2026-10-01 | ⭐ 时长策略 |
| [NealJun/unipus](https://github.com/NealJun/unipus) | 油猴脚本 | 显示答案、单元测试、自动答题、刷时长 | — | README 标 GPL-3.0（无 LICENSE 文件） | 2021-06-09（停更） | 仅历史参考 |

## 2. 逐个看

### 2.1 chaoxing（借鉴最多）

- **架构**：`api/answer.py` 定义抽象基类 `Tiku`，实现 `TikuFallback` 链式回退，以及言溪 / Go / Like / Adapter / AI / SiliconFlow / Manual 多个 Provider；`CacheDAO` 把命中结果落到 `cache.json`。
- **错题处理**：`work_redo_enabled` + `work_max_retries`（默认 3）——答错后把错误反馈给 AI 重做，直到全对或达上限。这解释了本项目「不满分自动反省、封顶 3 次」的来源。
- **其他**：验证码模块、字体反爬解码、通知模块、`jobs=4` 并发、视频倍速上限 2。
- **可借鉴**：Provider 抽象 + fallback 链 + 本地缓存；重做上限做成配置常量。
- **不能照搬**：第三方题库会把题目发出去，违反本项目红线 7（题目不出境）；并发多账号也不在自用范围内。

### 2.2 AutoUnipus

- **双模式**：Automode 自动识别必修题并提交；Assistmode 进入题目页后按键自动选中、**不提交**。
- **实现**：Python + Playwright，`account.json` 明文存账号密码；答案通过题号（qid）向外部服务查询（`res/fetcher.py`）；仅支持单选题；验证码人工输入；触发安全验证由用户手动完成。
- **可借鉴**：「自动 / 辅助」双模式与本项目 `MODE=fill|submit` 是同构设计，说明两阶段是社区验证过的方向。
- **不能照搬**：明文凭据、外部题库、默认提交。

### 2.3 UnipusAI（当前最活跃）

- **路线**：Rust + reqwest，不启浏览器；题目内容 AES-128-ECB 解密，登录账密 AES-128-CBC 加密，annotator token 本地签 HS256 JWT。
- **能力**：OpenAI 兼容 LLM 作答（选择/填空/简答/下拉）、讨论题自动发言、单词卡自动完成、Whisper 本地转写音视频、SQLite 按账号归档、ratatui TUI、CLI 子命令。
- **风控**：提交命中「操作过于频繁」后等待递增冷却（180s 起，最多 5 次）再重试。
- **可借鉴**：冷却参数、讨论题/音视频题的任务类型划分、SQLite 归档（对应我们的 ledger/answers）。
- **不能照搬**：协议逆向与 JWT 自签，正是本项目明确不做的部分。

### 2.4 UnipusAIAutoPlayer

- **形态**：Tampermonkey 用户脚本，作用域 `ucontent.unipus.cn`（`uai.unipus.cn` 首页不显示悬浮球）。
- **能力**：自动遍历 Unit / Section / Micro，总课时均分到每个子项，每页倒计时，可暂停继续并热改配置，自动关闭「我知道了」，视频 1-4 倍速。
- **可借鉴**：时长「总量均分 + 每步倒计时」比本项目「每节固定 10 分钟」更贴近真实学习节奏，可评估作为可选策略。

### 2.5 NealJun/unipus

- 2021 年停更的油猴脚本，能力为显示答案、单元测试、自动答题、刷时长。
- 只作历史参考；GPL-3.0（仓库无 LICENSE 文件，以 README 声明为准）。

## 3. 与本项目的边界

| 维度 | 前作常见做法 | 本项目 |
|---|---|---|
| 交互层 | HTTP 协议直连 / 油猴注入 | 真实浏览器语义操作，复用登录态 |
| 答案来源 | 第三方题库 / 远程答案服务 | 本地 Markdown 知识库 grep，题目不出境 |
| 提交 | 默认自动提交 | 默认只填不交，`MODE=fill` → 人工核对 → `MODE=submit` 过闸门 |
| 凭据 | 明文 config / account.json | 只在会话内使用，不落盘 |
| 验证码 | 部分尝试识别 | 一律交人工 |
| 失败处理 | 无限重试 / 无上限 | 明确上限（重做 3 次、冷却 ≥180s） |

## 4. 缺口与可执行下一步

1. **Provider 链**：在 `knowledge/` 之上加一层「本地 KB → 缓存 → AI 兜底」的 provider 抽象，并记录每题命中来源（chaoxing 思路）。
2. **命中率门槛**：把「知识库命中率」做成提交前置条件（当前闸门只比对页面值，不区分答案来源）。
3. **错题重做上限**：确认 `runner.py` 的重做路径有硬上限 3 次，并写入 `references/submit-policy.md`。
4. **冷却递增**：文档已写「≥180s」，代码尚无递增重试；若实现，参数对齐 UnipusAI（180s 起、最多 5 次）。
5. **任务类型补齐**：讨论题（UnipusAI `api/bbs.rs` 思路）、音视频转写（`transcribe.rs` 思路）目前归人工（pitfalls #28/#29），可作为后续 spec。
6. **文档同步**：新增借鉴点时，在 `docs/superpowers/specs/` 写清「借鉴什么、为什么不照搬」，避免 README 的「致敬」变成空话。

## 5. 维护提醒

- star 数与提交时间会变，引用时必须带日期；本文件按季度复核即可。
- 许可证：chaoxing / UnipusAI / UnipusAIAutoPlayer 为 GPL-3.0，AutoUnipus 为 BSD-3-Clause，NealJun/unipus 无 LICENSE 文件。**只借鉴设计，不复制代码**；若未来要复用代码，先确认 GPL 传染性与本项目 MIT 的兼容问题。
