# 一个完整任务的数据变化

> 场景：`TC-05.2 断电注入后数据一致性校验`，走 `testcase_v1` 六个阶段。
> 中途两次失败 —— 一次环境抖动（原地重跑）、一次脚本缺陷（回退重做）。

数据不是编的：下面每一个表快照都由真实 SQL 在 DESIGN.md 的 DDL 上跑出来，
表结构、约束、索引与文档完全一致。

## 先看全景

一次任务在库里只会发生四种变化：**建一行、改一行、标一行失效、追加一条事件**。没有第五种。

| 步 | 发生了什么 | `status` | `current_stage` | run 累计 | artifact 累计 |
|---|---|---|---|---|---|
| S0 | 部署：4 个 actor + 1 份 workflow 快照 | — | — | 0 | 0 |
| S1 | 建单，首阶段是 gate，开闸等人 | `open` → `active` | `clarify` | 0 | 0 |
| S2 | 人审通过，派发第一个 agent | `active` | `gen_case` | 1 | 0 |
| S3 | 用例生成成功 | `active` | `gen_script` | 1 | 1 |
| S4 | 派发脚本生成 | `active` | `gen_script` | 2 | 1 |
| S5 | 脚本 v1 交付，开闸 | `active` | `review_script` | 2 | 2 |
| S6 | 人审打回，回退 | `active` | `gen_script` | 2 | 2 |
| S7 | 回退重做，带 feedback | `active` | `gen_script` | 3 | 2 |
| S8 | 脚本 v2 通过，进入运行 | `active` | `run` | 4 | 3 |
| S9 | 运行失败（env），原地重跑 | `active` | `run` | 5 | 3 |
| S10 | 运行失败（script_bug），回退 | `active` | `gen_script` | 5 | 4 |
| S11 | 第三次重做，带运行日志 | `active` | `gen_script` | 6 | 4 |
| S12 | 脚本 v3 通过，运行成功 | `active` | `review_result` | 7 | 6 |
| S13 | 结果审通过 | `done` | `review_result` | 7 | 6 |

两个数字值得注意。

**`status` 全程只变了两次**（`open` → `active` → `done`）。进度全在 `current_stage` 上，
它走过 6 个阶段、回退 2 次。两套机制各管一件事，所以从没出现「两个字段互相矛盾」。

**7 次 run 只覆盖 3 个阶段。** `run` 是「一次尝试」，不是「一个阶段」——
这是它能同时装下 agent 执行、job 执行、环境重跑、回退重做的原因。

下面 S0–S15 逐步展开，每一步给出这一轮写了哪几张表、写成了什么状态。

---


### S0 · 初始化：四个 actor（1 人 + 3 agent），一份 workflow 快照

> 部署时执行一次，不随任务变化。


actor（4 行，人与 agent 同表）：

| id | kind | name | runtime_id | capabilities | max_parallel |
|---|---|---|---|---|---|
| 1 | `human` | `zhaoqi` |  | `[]` | 1 |
| 2 | `agent` | `agent-case` | `lab-01` | `["testcase"]` | 1 |
| 3 | `agent` | `agent-script` | `lab-01` | `["testcase","script","python"]` | 2 |
| 4 | `agent` | `agent-runner` | `lab-02` | `["script","lab"]` | 1 |


workflow（1 行，不可变快照）：

| id | version | spec_hash |
|---|---|---|
| `testcase_v1` | 1 | sha256:c8819e93e1b1 |

`spec` 存的是 6 个阶段的表 + `on_failure` 六条路由 + `max_stage_attempts: 3`。
YAML 后续改动只会插新版本，这一行永不被覆盖 —— 在途 issue 钉的就是它。

### S1 · 建单：issue 落到首阶段，首个阶段是 gate，直接开闸

> 人 POST /api/issues，`workflow_id=testcase_v1`。stage_driver 入口一：issue 创建。


issue（1 行）：

| id | status | current_stage | awaiting_gate | stage_attempts | version |
|---|---|---|---|---|---|
| 10 | `active` | `clarify` | `clarify` | `{}` | 1 |


event（2 条）：

| id | type | run_id | payload |
|---|---|---|---|
| 1 | `issue_created` |  | `{"workflow_id": "testcase_v1", "workflow_version": 1, "first_stage": "clarify"}` |
| 2 | `gate_opened` |  | `{"stage": "clarify", "prompt": "与需求方确认验收标准"}` |

注意：`awaiting_gate='clarify'`，但 `status` 是 `active` 不是「待审」—— gate 等人是正常工作状态，不是异常。**`run` 表此刻是空的**，人审不产生 run。

### S2 · gate 通过：清闸，派发第一个 agent run

> 人点 approve。注意两件事在同一事务里：清 `awaiting_gate`、建 run。


issue（推进后）：

| status | current_stage | awaiting_gate | version |
|---|---|---|---|
| `active` | `gen_case` |  | 2 |


run（第 1 条）：

| id | stage | kind | actor_id | attempt | status | wall_clock_s |
|---|---|---|---|---|---|---|
| 1 | `gen_case` | `agent` | 2 | 1 | `queued` | 0 |

`run.route` 落的是**决策留痕**，不是结果：候选两个、为什么排除 4、为什么选 2。
agent 一多，「为什么派给它」是唯一无法从别的字段反推的问题。

### S3 · 一次 agent run 的生命周期：认领 → 事件 → 交付

> daemon 从 SSE 收到 job，CAS 认领，跑完上报。


run（认领 → 运行 → 成功）：

| id | status | claim_token | heartbeat_at | tokens_in | tokens_out | wall_clock_s |
|---|---|---|---|---|---|---|
| 1 | `succeeded` | `c-7f3a` | 1761667680000 | 48210 | 6310 | 312 |


artifact（第 1 条）：

| id | stage | version | kind | content_ref | content_hash | produced_by | stale |
|---|---|---|---|---|---|---|---|
| 1 | `gen_case` | 1 | `testcase` | `artifacts/10/gen_case/v1/testcase.md` | sha256:9ba19826294d | 1 | 0 |

一次 `complete` 写了四张表：`run` 置 succeeded、`artifact` 插入、`event` 追加、`issue` 推进。
在 SQLite 里这是一个 `BEGIN IMMEDIATE` 包住的四次写，**不可能只写一半**。

### S4 · 派发 gen_script：亲和优先，选回同一台机器

> requires=[script,python]，只有 actor 3 满足。


run（第 2 条）：

| id | stage | kind | actor_id | attempt | status |
|---|---|---|---|---|---|
| 2 | `gen_script` | `agent` | 3 | 1 | `queued` |

`run.inputs`（首次执行，`feedback` 为空）：

```json
{
  "base": {
    "artifact_id": 1,
    "hash": "sha256:9ba19826294d",
    "ref": "artifacts/10/gen_case/v1/testcase.md"
  },
  "feedback": null,
  "constraints": [
    "脚本入口固定 tests/case_10.py",
    "不得硬编码物理 LBA 偏移"
  ]
}
```

`inputs.base` 指向上游 artifact 的 id 与 hash。hash 的作用是**发现上游被换过**：
agent 干到一半上游出了 v2，它能从 hash 不一致察觉自己读的不是当前版本。

### S5 · 脚本交付：推进到 review_script，开闸等人

> 这是第一版脚本，**它有缺陷**，后面会被打回。


issue（等人审）：

| status | current_stage | awaiting_gate | gate_opened_at | version |
|---|---|---|---|---|
| `active` | `review_script` | `review_script` | 1761668400000 | 4 |


artifact（2 条）：

| id | stage | version | kind | content_hash | stale |
|---|---|---|---|---|---|
| 1 | `gen_case` | 1 | `testcase` | sha256:9ba19826294d | 0 |
| 2 | `gen_script` | 1 | `script` | sha256:122da8dcb0d4 | 0 |

### S6 · 人审打回：回退到 gen_script，下游失效

> 人点 reject，`back_to=gen_script`。这一条是回退机制的核心，看它写了哪几张表。


issue（回退后）：

| status | current_stage | awaiting_gate | stage_attempts | version |
|---|---|---|---|---|
| `active` | `gen_script` |  | `{}` | 5 |


artifact（script v1 被标 stale）：

| id | stage | version | stale |
|---|---|---|---|
| 1 | `gen_case` | 1 | 0 |
| 2 | `gen_script` | 1 | 1 |

注意 `gen_script` 自己的 v1 也被标了 `stale=1` —— 回退范围是「目标阶段（含）之后」。
这一步很容易漏：脚本 v2 生成后，v1 的 review 结论已经作废，不能拿它当「已验证」。

### S7 · 回退重做：run.inputs 带失败证据与约束

> 这是全流程最关键的一步 —— 不带 feedback 的原地重跑，agent 会生成几乎一样的脚本。


run（第 3 条）：

| id | stage | attempt | status |
|---|---|---|---|
| 3 | `gen_script` | 2 | `queued` |

`run.inputs`（回退重做，`feedback` 非空）：

```json
{
  "base": {
    "artifact_id": 1,
    "hash": "sha256:9ba19826294d",
    "ref": "artifacts/10/gen_case/v1/testcase.md"
  },
  "feedback": {
    "from_run": 2,
    "error_class": "script_bug",
    "signal": "AssertionError: LBA 0x1a2f expect 0x00 got 0x47 — 该 LBA 已被 FTL 重映射",
    "log_artifact": null,
    "review_note": "断言写死了物理偏移 0x1a2f，FTL 重映射后必然失败"
  },
  "constraints": [
    "只改断言与重试逻辑",
    "不得修改 test vector 与步骤序列",
    "不得修改验收标准"
  ]
}
```

这里出现**两个计数器**，语义不同：

| 计数器 | 此刻的值 | 含义 |
|---|---|---|
| `run.attempt` | 2 | 这个 stage 的第几次执行，含 `env` 重跑 |
| `issue.stage_attempts[gen_script]` | **0** | 这个 stage 第几次**计入**的失败 |

`UNIQUE(issue_id, stage, attempt)` 靠前者，`max_stage_attempts` 闸门靠后者。

注意这里 `stage_attempts` 是 **0**：人审打回只改 `current_stage`、只标 stale，**不计数**。
这是文档 `stage_driver` 穷举表里 gate rejected 那一行的字面行为，后果后面单独说。

### S8 · 第二版脚本通过人审，进入运行阶段

> agent 只改了断言，用例与步骤没动 —— 这是 constraints 起作用的结果。


artifact（3 条，v1 仍 stale）：

| id | stage | version | kind | stale | produced_by |
|---|---|---|---|---|---|
| 1 | `gen_case` | 1 | `testcase` | 0 | 1 |
| 2 | `gen_script` | 1 | `script` | 1 | 2 |
| 3 | `gen_script` | 2 | `script` | 0 | 3 |

v1 没有被删、也没有被覆盖。审计问「当时为什么失败」时，答案就在这一行里。

### S9 · 第一次运行失败：环境问题，原地重跑且不计次

> `error_class=env`。真机抖一下就把回退额度用光，整个工作流会废掉。


run（job 执行两次，attempt 递增但 stage_attempts 不变）：

| id | stage | attempt | status | error_class | wall_clock_s |
|---|---|---|---|---|---|
| 4 | `run` | 1 | `failed` | `env` | 8 |
| 5 | `run` | 2 | `queued` |  | 0 |


issue（stage_attempts 仍为空）：

| current_stage | stage_attempts | status |
|---|---|---|
| `run` | `{}` | `active` |

`env` 不计次，`stage_attempts` 一条没加；但 `run` 行照样新增一条。`run` 数上限（`max_runs`）防的正是
「环境一直不好、无限重跑」—— 它不靠 `stage_attempts`，靠 run 条数。

### S10 · 第二次运行失败：脚本缺陷，回退到 gen_script

> 这次是 `script_bug`，要计入次数并回退。


run（全部 5 条）：

| id | stage | kind | attempt | status | error_class |
|---|---|---|---|---|---|
| 1 | `gen_case` | `agent` | 1 | `succeeded` |  |
| 2 | `gen_script` | `agent` | 1 | `succeeded` |  |
| 3 | `gen_script` | `agent` | 2 | `succeeded` |  |
| 4 | `run` | `job` | 1 | `failed` | `env` |
| 5 | `run` | `job` | 2 | `failed` | `script_bug` |


issue（stage_attempts 首次变成非空）：

| current_stage | stage_attempts | version |
|---|---|---|
| `gen_script` | `{"gen_script": 1}` | 8 |


artifact（4 条，下游全部 stale）：

| id | stage | version | kind | stale |
|---|---|---|---|---|
| 1 | `gen_case` | 1 | `testcase` | 0 |
| 2 | `gen_script` | 1 | `script` | 1 |
| 3 | `gen_script` | 2 | `script` | 1 |
| 4 | `run` | 1 | `run_log` | 1 |

`script v2` 和刚产出的 `run_log v1` 一起被标 stale —— **回退把下游产出全部作废**。
推进时必须重走 `review_script`，不能从 `run` 直接继续。

但这里有个问题：**上一轮「review_script 已通过」这件事没有任何一行承载它**。
gate 不建 run、不产 artifact，通过的结论只存在于 `event(type=gate_approved)` 这一条事件里，
而 stale 传播只作用于 `artifact` 表 —— 它标记不到一条事件上。

### S11 · 第三次重做：这次拿到的是运行日志，不是人审意见

> `feedback` 的来源不同，agent 要读的东西也不同。


run（第 6 条）：

| id | stage | attempt | status |
|---|---|---|---|
| 6 | `gen_script` | 3 | `queued` |

`run.inputs`（这次 `feedback` 来自运行日志，不是人审）：

```json
{
  "base": {
    "artifact_id": 1,
    "hash": "sha256:9ba19826294d"
  },
  "feedback": {
    "from_run": 5,
    "error_class": "script_bug",
    "signal": "AssertionError: expect 0x00 got 0x47 at block 31",
    "log_artifact": 4
  },
  "constraints": [
    "只改断言与重试逻辑",
    "不得修改 test vector 与步骤序列",
    "不得修改验收标准"
  ]
}
```

这一次 `feedback.log_artifact` 指向**运行日志**，上一轮指向的是人写的 review 意见。
设计里 feedback 只有 `signal` 与 `log_artifact` 两个载体，两种来源都得塞进同一个形状。

### S12 · 第三版脚本通过，job 跑通，开结果审 gate

> 这一轮每个 stage 都用了不同次数。


run（7 条，run 阶段 3 次）：

| id | stage | kind | attempt | status | error_class | wall_clock_s |
|---|---|---|---|---|---|---|
| 1 | `gen_case` | `agent` | 1 | `succeeded` |  | 312 |
| 2 | `gen_script` | `agent` | 1 | `succeeded` |  | 734 |
| 3 | `gen_script` | `agent` | 2 | `succeeded` |  | 298 |
| 4 | `run` | `job` | 1 | `failed` | `env` | 8 |
| 5 | `run` | `job` | 2 | `failed` | `script_bug` | 42 |
| 6 | `gen_script` | `agent` | 3 | `succeeded` |  | 271 |
| 7 | `run` | `job` | 3 | `succeeded` |  | 1873 |


artifact（6 条）：

| id | stage | version | kind | stale |
|---|---|---|---|---|
| 1 | `gen_case` | 1 | `testcase` | 0 |
| 2 | `gen_script` | 1 | `script` | 1 |
| 3 | `gen_script` | 2 | `script` | 1 |
| 4 | `run` | 1 | `run_log` | 1 |
| 5 | `gen_script` | 3 | `script` | 0 |
| 6 | `run` | 2 | `run_log` | 0 |

### S13 · 结果审通过，任务完成

> 末阶段 approve → `status=done`。


issue（终态）：

| id | status | current_stage | awaiting_gate | stage_attempts | version |
|---|---|---|---|---|---|
| 10 | `done` | `review_result` |  | `{"gen_script": 1}` | 12 |

### S14 · 终态汇总：四张表的全貌

> 一个任务从头走到尾，最终落在库里的是这些。


run（7 行，3 个 stage 共 7 次执行）：

| id | stage | kind | attempt | status | tokens_in | tokens_out | wall_clock_s | error_class |
|---|---|---|---|---|---|---|---|---|
| 1 | `gen_case` | `agent` | 1 | `succeeded` | 48210 | 6310 | 312 |  |
| 2 | `gen_script` | `agent` | 1 | `succeeded` | 96400 | 12800 | 734 |  |
| 3 | `gen_script` | `agent` | 2 | `succeeded` | 41200 | 5300 | 298 |  |
| 4 | `run` | `job` | 1 | `failed` | 0 | 0 | 8 | `env` |
| 5 | `run` | `job` | 2 | `failed` | 0 | 0 | 42 | `script_bug` |
| 6 | `gen_script` | `agent` | 3 | `succeeded` | 38900 | 4900 | 271 |  |
| 7 | `run` | `job` | 3 | `succeeded` | 0 | 0 | 1873 |  |


artifact（6 行，3 行 stale）：

| id | stage | version | kind | stale | produced_by |
|---|---|---|---|---|---|
| 1 | `gen_case` | 1 | `testcase` | 0 | 1 |
| 2 | `gen_script` | 1 | `script` | 1 | 2 |
| 3 | `gen_script` | 2 | `script` | 1 | 3 |
| 4 | `run` | 1 | `run_log` | 1 | 5 |
| 5 | `gen_script` | 3 | `script` | 0 | 6 |
| 6 | `run` | 2 | `run_log` | 0 | 7 |


event（25 条，全流程审计）：

| id | run_id | actor_id | type |
|---|---|---|---|
| 1 |  | 1 | `issue_created` |
| 2 |  |  | `gate_opened` |
| 3 |  | 1 | `gate_approved` |
| 4 | 1 |  | `run_queued` |
| 5 | 1 | 2 | `progress` |
| 6 | 1 | 2 | `note` |
| 7 | 1 | 2 | `artifact` |
| 8 | 1 | 2 | `run_succeeded` |
| 9 | 2 |  | `run_queued` |
| 10 | 2 | 3 | `run_succeeded` |
| 11 |  |  | `gate_opened` |
| 12 |  | 1 | `gate_rejected` |
| 13 | 3 |  | `run_queued` |
| 14 | 3 | 3 | `run_succeeded` |
| 15 |  | 1 | `gate_approved` |
| 16 | 4 |  | `run_queued` |
| 17 | 4 | 4 | `run_failed` |
| 18 | 5 |  | `run_queued` |
| 19 | 5 | 4 | `run_failed` |
| 20 | 6 |  | `run_queued` |
| 21 | 6 | 3 | `run_succeeded` |
| 22 |  | 1 | `gate_approved` |
| 23 | 7 |  | `run_queued` |
| 24 | 7 | 4 | `run_succeeded` |
| 25 |  | 1 | `gate_approved` |

预算累计（对 run 表求和，不存冗余字段）：token **254,020** / 2,000,000，墙钟 **3,538s** / 86,400s，run 数 **7** / 20。

实际执行 3,538s（59 分钟），而日历时间跨了 1 小时 49 分钟 —— 差的是人审与排队。
预算只统计 `run` 表，所以**等人不花预算**；否则一张审三天的单子会被判超支。

其中 `job` 三次合计 1,923s、0 token，占总墙钟 54% —— 
只按 token 计费会完全漏掉它，这正是「按 kind 计费」的原因。

### S15 · 附录：路由没有候选时不建 run，issue 不转失败

> `lab-02` 上的 daemon 掉线。这是「待派发」与「失败」必须分开的原因。

`run` 表里 issue 11 **一行也没有** —— 没有 run 就没有心跳、没有超时，
也就没有被误判成 failed 的可能。


issue（仍是 active，stage 停在 run）：

| id | status | current_stage | awaiting_gate |
|---|---|---|---|
| 11 | `active` | `run` |  |

「待派发」是派生的，不需要新字段：

```sql
SELECT id FROM issue
 WHERE status = 'active'
   AND current_stage IS NOT NULL
   AND id NOT IN (SELECT issue_id FROM run
                  WHERE status IN ('queued','claimed','running'));
```


这条查询此刻命中：

| id |
|---|
| 11 |

命中 issue 11，不命中 issue 10（它已 done）。daemon 上线后跑一次这个查询，逐个补派。
如果把「没人接」也写成 `failed`，运维看到的就是一堆假失败 —— 而它其实会自愈。

---

## 对照 DESIGN.md：这套设计撑不撑得住这条流程

### 结论

**撑得住。** 这条流程的每一个中间状态都能在六张表里找到落点，没有一处需要「再多存点什么」才能表达。
6 个阶段、7 次执行、2 次回退、2 类失败，全部落在既有字段上。

但推演过程中有 **9 处文档没写死**，其中 **3 处不同的人会写出不同实现**。下面按「会不会导致实现分歧」排序。

### 一、撑住了的部分，以及证据

| 设计断言 | 这次推演里的证据 |
|---|---|
| issue / run / gate 三概念分离 | S1 时 `run` 表 **0 行**；四次人审累计等待 28 分钟，没有任何一行需要心跳或超时 |
| `status` 只管「卡没卡」，`current_stage` 管进度 | `status` 全程变 2 次，`current_stage` 变 7 次，两者从未打架 |
| issue 与 run 分离，重试不覆盖历史 | 3 个阶段共 7 条 run，失败记录（4、5）与成功记录（7）并存 |
| `UNIQUE(issue_id, stage, attempt)` | `gen_script` 的 1/2/3 与 `run` 的 1/2/3 同时存在，跨阶段同号不撞 |
| 产物版本化 + 可失效 | 6 行 artifact、3 行 stale，旧版本一条没删 |
| 事件是唯一事实流 | 25 条事件覆盖全部状态迁移，没有一条需要额外补写 |
| 回退必须携带失败证据 | S7 的证据来自人审、S11 来自运行日志，同一个 `feedback` 形状都装得下 |
| `env` 不计次 | `run` 阶段执行 3 次，`stage_attempts` 只记了 `gen_script: 1` |
| 按 kind 计费 | `job` 三次合计 1,923s、0 token，占总墙钟 **54%** |
| 预算不存冗余字段 | 对 `run` 表求和得出 254,020 token / 3,538s，与逐行相加一致 |
| 无候选时不建 run | S15 里 issue 11 的 `run` 表 0 行，`status` 仍是 `active`，一条派生查询就能找出「待派发」 |

其中第 1 条和第 2 条是整份设计的地基，这次都验证到了：**gate 期间 `run` 表是空的**，
所以「人审三天被杀掉」这类问题在这个模型里根本不存在。

### 二、必须补的 3 处（会导致实现分歧）

#### 1 · `artifact.version` 没有生成规则

表上只有 `UNIQUE(issue_id, stage, version)`，文档没写 `version` 从哪来。
两个人会写出两种：`COUNT(*)+1`（删过行就撞）与 `MAX(version)+1`。
而且它必须和插入在**同一个 `BEGIN IMMEDIATE` 事务**里算，否则两个 daemon 同时交付时
会读到同一个 `MAX`，第二条插入直接 500 —— 而这类冲突只在并发时出现，本地测不出来。

→ 建议在 §4.4 补一句：`version = MAX(version)+1`，与插入同事务计算。

#### 2 · `run.inputs.base` 是单数，容不下两个上游

S7 的回退里，`gen_script` 同时依赖两个产物：`testcase v1`（上游，没变）和 `script v1`（自己上一版，要被替换）。
文档的 `base` 是一个对象，只能指一个 artifact。推演里我只放了 `testcase`，
**上一版脚本没有引用位置** —— agent 只能自己扫 `artifact` 表找「该 stage 最新一条」。

这等于把契约的一部分留给了 agent 的自觉。换一个 agent 实现就可能读错版本。

→ 建议 `base` 改成数组，或加一个 `replaces: {artifact_id}`。

#### 3 · gate 的结论没有存放位置

S8 之后 `review_script` 已经通过，但这件事**只存在于 `event(type=gate_approved)`**。
S10 回退时那句 `UPDATE artifact SET stale = 1` 标不到它 —— 事件是 append-only，不能改也不该改。

后果：没有任何一条 SQL 能回答「当前阶段是否已被审过」，看板要回答这个问题只能回放事件流。
本设计里 `artifact.kind` 有 `review_note`，但 gate 既不建 run、也不产 artifact，这个 kind 目前没有生产者。

→ 建议二选一：让 gate 产生 `artifact(kind='review_note', produced_by=NULL)`，给 stale 传播一个对象；
或在 issue 上加 `gate_passed` JSON，由 `stage_driver` 维护。

### 三、建议补的 4 处（容易各写一套）

| 位置 | 缺什么 | 推演里的表现 |
|---|---|---|
| §11 的 gate rejected 行 | 没有计数字段 | S6 人审打回后 `stage_attempts` 仍是 `{}`。人可以无限打回，而 agent 失败 3 次就 blocked —— 不对称 |
| §11 的 gate approved 行 | 只写了「非末阶段」 | 末阶段 approve 该写什么没定义。我按 §3.1 让 `current_stage` 停在 `review_result`，另一个人很可能写成 `__done__` |
| §5.5 stale 传播 | 没写「只对已存在行生效」 | 若实现成「推进时按阶段标 stale」，新生成的 v3 会被自己标掉，之后每次推进都判 stale —— 死循环 |
| §5.3 的 feedback 结构 | 只有 `signal` / `log_artifact` | 人打回时写的意见无处可放。推演里我加了 `review_note` 字段，**文档里没有这个字段** |

第一条值得单独说：`stage_attempts` 的语义是「**计入的失败**次数」，不是「执行次数」。
S6 打回后它是 0，S10 失败后才变 1，而 `gen_script` 实际跑了 3 次。三个数字互不相同，都需要明确。

### 四、可选补的 2 处

| 位置 | 情况 |
|---|---|
| `event.type` 没有枚举约束 | 25 条事件里出现了 **11 种** type，表上没有 `CHECK`，前端只能靠字符串匹配 |
| `run.stage = '__single__'` | 如果谁把某个 stage 命名为 `__single__`，语义会撞车。加一条启动期校验即可 |

### 五、一条文档没明说、但结论正确的推断

预算按 `run` 表求和，意味着**只统计执行时间**。这次实际执行 3,538s（59 分钟），
而日历时间跨了 1 小时 49 分钟 —— 差的那 50 分钟是人审与排队。也就是说**等人不花预算**。

这一条文档没写，但方向是对的：否则一张审三天的单子会被直接判超支，
而超时的原因跟 agent 毫无关系。

---

## 小结

| 维度 | 判定 |
|---|---|
| 能否完整表达这条流程 | 能。所有中间状态都有落点，无需额外字段 |
| 字段是否够用 | 够，但 `inputs` 与 gate 结论各缺一块 |
| 约束是否可靠 | 三个关键约束（UNIQUE、复合外键、CHECK）实测都生效 |
| 需要补写的地方 | 3 处必补、4 处建议、2 处可选 |

真正会让人写错的是第二节那三条：版本号怎么生成、上游产物怎么引用、人审结论存在哪。
它们不是设计错误，是设计**没写到那么细**——而这三处恰好都在「两个组件交接」的位置上，
最容易各写一套然后对不上。
