# 必修任务清单与探测方法

> 真机校准：2026-11-09 · 账号课程「大学英语3（普22）」· 教程《新标准大学英语（第二版）综合3》
> 用途：批量作答前，先确认**哪些任务算完成、哪些不用做**，避免把时间花在选修内容上。

## 1. 判定规则

平台在任务行上渲染一个「必修」角标（选修任务没有角标）。判定只看这一处：

| 位置 | 选择器 | 取值 |
|---|---|---|
| 任务名 | `[class*=courses-unit_taskTypeName]` | `Vocabulary` / `Exercise-1` … |
| 必修标记 | `[class*=courses-unit_taskRequireIcon]` | `必修`（无此元素 = 非必修） |
| 完成状态 | `[class*=courses-unit_nodePassStateTip]` | `未开始` / `进行中` / `已完成` |

**不要**用截图或页面文本猜测：同名任务在不同小节重复出现（如 1-2 与 1-4 都有 `Text`），必须按行读角标。

## 2. 2026-11-09 快照（综合3）

| Unit | 任务总数 | 必修 | 必修范围 |
|---|---|---|---|
| 1 | 32 | 6 | 1-2 Vocabulary + Exercise-1~4；1-6 Unit test |
| 2 | 36 | 36 | 全部（2-4 Text 已完成） |
| 3 | 35 | 7 | 3-1 Vocabulary + Exercise-1~5；3-6 Unit test |
| 4 | 36 | 36 | 全部 |
| 5 | 38 | 38 | 全部 |
| 6 | 33 | 6 | 6-2 Vocabulary + Exercise-1~4；6-6 Unit test |
| 7 | 34 | 34 | 全部 |
| 8 | 36 | 8 | 8-2 Vocabulary + Exercise-1~6；8-6 Unit test |
| **合计** | **280** | **171** | 与课程卡片「已学任务 0/171」一致 |

### 过关条件（「学习要求」页）

| Unit | 学习周期 | 单元过关条件（客观题正确率） |
|---|---|---|
| 1 / 3 / 6 / 8 | 2026-08-31 ~ 2026-12-27 | **≥60%** |
| 2 / 4 / 5 / 7 | 无限制 | 0%（只要求完成） |

> 结论：1/3/6/8 是「少而精 + 卡正确率」，不能乱填；2/4/5/7 是「全做」，重点是完成而不是得分。
> 作业页当前为「暂无作业」，任务全部在教程目录内。

## 3. 复扫方法（只读）

在 `resource-detail` 页执行以下函数，返回当前 Unit 每个任务的 `{sec,title,req,state}`：

```js
() => {
  const cs = [...document.querySelectorAll('div[class*=courseDetailNodeCollapseContainer]')];
  const scope = cs.find(c => c.offsetParent !== null) || cs[0] || document.body;
  const seq = [...scope.querySelectorAll('div[class*=ant-collapse-header], div[class*=courses-unit_taskItemContainer]')];
  let sec = ''; const out = [];
  for (const el of seq) {
    if (el.classList.contains('ant-collapse-header')) {
      const t = (el.innerText || '').trim().split('\n')[0];
      if (/^\d+-\d+\s/.test(t)) sec = t;
      continue;
    }
    const title = (el.querySelector('[class*=courses-unit_taskTypeName]')?.innerText || '').trim();
    const req = (el.querySelector('[class*=courses-unit_taskRequireIcon]')?.innerText || '').trim();
    const state = (el.querySelector('[class*=courses-unit_nodePassStateTip]')?.innerText || '').trim();
    if (title) out.push({ sec, title, req, state });
  }
  return out;
}
```

切换 Unit 必须点 `[class*=tabDefaultModeContainer]` 且文本恰为 `Unit N`；用文本点击会命中侧边栏同名节点。

## 4. 验证记录（2026-11-09 · Unit 1）

| 任务 | 题数 | 首次得分 | 修正后 | 答案来源 |
|---|---|---|---|---|
| 1-2 Exercise-2 | 12 空 | 100 | — | 知识库 Task 1 直接命中 |
| 1-2 Exercise-3 | 8 选择 | 100 | — | AI 兜底 |
| 1-2 Exercise-4 | 9 选择 | 88.9（Q8 错） | 100 | AI 兜底 |

- 首次 29 题错 1 题（Exercise-4 Q8「make the best of」应为 B），修正后全部满分。
- **错题定位**：结果页题号元素 `class*=item-wrong`（红色），点开解析可读「正确答案 / 你的答案」。
- **重做会清空该练习的全部答案**，需整页重填；平台按「最高一次成绩」计。
- 结论：知识库命中题可靠；AI 兜底可用于 1/3/6/8 的 60% 过关线，但不要假设全对。

## 5. 全量跑批记录（2026-11-09 · Unit 1/3/6/8）

用户口径：Unit 2/4/5/7 不做；Vocabulary 全部跳过；其余必修**全部提交（含 Unit test）**。

| Unit | 必修 | 已完成 | 未做 | 得分 |
|---|---|---|---|---|
| 1 | 6 | 5 | 1-2 Vocabulary（跳过） | Ex1~4 + Unit test 全 100 |
| 3 | 7 | 6 | 3-1 Vocabulary（跳过） | Ex1~5 + Unit test（45/45）全 100 |
| 6 | 6 | 5 | 6-2 Vocabulary（跳过） | Ex1~4 + Unit test（45/45）全 100 |
| 8 | 8 | 7 | 8-2 Vocabulary（跳过） | Ex1~6 全 100；Unit test 97.8（44/45） |

**本次踩到的坑**

1. **课件域 SSO 失效的新形态**：不是跳到登录页，而是
   `ucontent.unipus.cn/cas-auth/login/...&ticket=ST-...` 直接回 `Forbidden / Login failed out of limit.`。
   `status` 仍是 `loggedIn:true`（uai 域自己的 jwt 还在）。清 ucontent cookie 无效；
   **清 `sso.unipus.cn` 的 TGC + `.unipus.cn` 的 jwt，重登一次**立刻恢复。判据：同账号在普通浏览器能进课件 → 本地登录态脏。
2. **驱动接口只认 query 参数**：`POST /nav`、`/open-course`、`/click-row` 的字段必须放 URL 查询串，放 JSON body 会得到 `q.url === undefined`（只有 `/eval`、`/fill` 从 body 取）。
3. **弹窗会吃掉点击**：课件页首屏常有「本单元学习时间 / 是否必修」确认框，挡住了「下一题」。操作前先 `click 我知道了`。
4. **`click-row` 不识别必修标记**：同名任务（如 `Exercise-1`）先按小节顺序编号，必须先扫一遍 `{title,nth,sec,req}` 再定位，否则会点到 6-1 的选修题。
5. **知识库两处错**：Unit 8 Active reading 2 First reading 第 7 题应为 D（已改）；Unit 8 Unit test 第 45 题应为 B（courageous），第 19 题 `endeavour / endeavours` 两种形式平台都判错（疑似判分 bug，放弃追）。

## 6. 账号内另两本书（2026-11-09 扫描，未作答）

### 6.1 新标准大学英语（第二版）视听说3 — 200 个任务全部必修

| Unit | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 合计 |
|---|---|---|---|---|---|---|---|---|---|
| 必修 | 26 | 25 | 28 | 25 | 23 | 25 | 25 | 23 | **200** |

- 每单元固定结构：Inside view（Conversation 1-1~2-5）→ Outside view（Watching and understanding-1~5）→ Listening in（News report-1/2、Passage 1/2 各 1-2）→ Presentation skills（Learning / Practice）→ Pronunciation（3~6 个语音练习）→ Unit test。
- **时间限制：任务级没有**。「学习要求」页 8 个单元全是「无限制 - 无限制」，无开始/截止日期；单元过关条件 0%（只要求完成）。
- **成绩统计方式：首次成绩**（与综合3 的「最高一次成绩」相反，错一次就定分，重做无效）。
- 实测倒计时：Conversation 1-1（`00:00/01:33`）、Watching and understanding-1（`00:00/00:41`）、News report-1（无时间元素）、Passage 1-1（`00:00/03:30`）、Unit test（`00:16`）——**都是音频时长，没有限时器**。Presentation skills / Pronunciation 未测。
- **不能自动作答复的原因**：`knowledge/` 无这本书；页面只有 `Words & tips`（生词+文化注释），**没有听力原文**；`<video>/<audio>` 也没有 `<track>` 字幕轨。听力题无答案来源，硬做就是瞎猜，且首次成绩不可挽回。

### 6.2 新视野大学英语（第三版）视听说思政数字课程3 — 391 个任务，122 个必修

| Unit | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 合计 |
|---|---|---|---|---|---|---|---|---|---|
| 必修/总数 | 14/52 | 15/50 | 16/49 | 14/47 | 17/48 | 15/46 | 15/49 | 16/50 | **122/391** |

必修集中在：`Sharing · Practice-2~N`、`Listening · Use the skills`、`More practice in listening`（Short conversations / Long conversation / Passages-1/2 / News-1/2）、`Unit test`。
章节名与 `knowledge/新视野大学英语（第三版）/新视野大学英语 视听说教程3.md` 一一对应，有答案可查。

> 用户 2026-11-09 明确：**思政版不用管**，视听说3 暂不做。

## 7. 使用建议

1. 每单元先复扫一遍再排任务：`req === '必修'` 的才进队列。
2. 1/3/6/8：优先保客观题正确率，Unit test 单独处理（先看是否限时/监考）。
3. 2/4/5/7：全部是必修，但大量为 `Pre-reading / Text / Background` 阅读型任务，走 `read`/`cards` 模式而非答题。
4. 教师调整要求后角标会变，本表只作 2026-11-09 的快照，不作为长期依据。
