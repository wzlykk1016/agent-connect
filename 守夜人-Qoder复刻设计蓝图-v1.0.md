# 守夜人 · Qoder 复刻设计蓝图

> 版本：v1.0　｜　日期：2026-11-17　｜　状态：设计蓝图，未开工
> 目标环境：Qoder（IDE / Quest Mode / CLI），赛场仅允许使用 Qoder 侧 AI 能力
> 源系统：`C:\个人\mcp\守夜人\`（watch.py 52 KB / watchctl.py 23 KB / watchdesk.py 17 KB / agent_auditor.py 17 KB）
> 本文所有「原系统行为」均取自源码与实测记录，关键处标注 `文件:行号`；所有「Qoder 能力」标注了验证状态，未验证的一律写「待核实」，不做推测性断言。

---

## 0. 阅读指北

| 读者 | 先读 |
|---|---|
| 评委 / 非技术 | §1 执行摘要、§4.1 分层图、§9.4 演示脚本 |
| 开发者 | §5 数据契约、§6 模块详设、§7 Qoder 适配层 |
| 运维 | §8 部署运行、§10 风险、附录 A 常量表 |
| 审核 / 安全 | §2.2 设计原则、§6.5–6.8 三道闸门、§10 红线 |

**符号约定**：`[源码]` = 原系统已有实现；`[Qoder]` = Qoder 平台能力；`[待核实]` = 文档未定论，必须在本机 `--help` 或实测确认；`[新增]` = 复刻时新增的适配代码。

---

## 1. 执行摘要

**守夜人是什么**：一个**无人值守的软件任务调度器**。它自己不写代码、不判断方案好坏，只做一件事——每隔 N 分钟醒来，读一份「交接说明」，判断「现在该谁上场」，该 AI 执行端上场就拉起它，跑的过程中盯着有没有越界 / 丢回退点 / 卡死，跑完只验收机器能判的几件事，然后回到等待。它把「AI 编程」从「人坐在旁边一步步点」变成「人下班前下一单，第二天看回执」。

**为什么要复刻**：原系统绑定 Claude Code CLI（`claude.cmd -p`）。赛场规则只允许 Qoder 侧的 AI，因此必须把「执行端」换成 Qoder，同时保留原系统最值钱的三样东西：

1. **状态词即调度表**——一个字段同时编码「谁该动」，不需要额外的 owner 字段；
2. **三道机械闸门**——开工前、跑动中、收网后，全部可机器判定，不靠 AI 自觉；
3. **判停线**——同一任务收网后绝不自唤醒，没人下新单就永远不烧钱。

**复刻的边界**：控制面（状态机、锁、调度）用纯 Python 标准库原样重建；执行面通过一个**适配器**对接 Qoder；展示面用 Qoder Sites 做只读看板。**AI 调用点收敛为一处**（`agent_adapter`），其余 95% 的代码不含任何 AI。

**一句话架构**：

```
Windows 计划任务 / cron
        │ 每 N 分钟
        ▼
watch.py:tick ──读通道──> 判断状态词 ──拉起──> agent_adapter ──> qodercli -p
     │                                              │
     ├─ 过程看护（越界/丢回退点/卡死）               └─ 原始输出逐行落盘
     ├─ 收网机械验收（回执/范围/代码/回退点）
     └─ 写状态机 + 逐轮日志
```

---

## 2. 原系统解剖

### 2.1 一句话定位

> 智力全在 if-else 与阈值：它不判断任务对不对，只判断「现在有没有该醒的活」。它不是验收者，是「该谁上场」的判定者。
> —— `watch.py:1-14` 模块 docstring

### 2.2 十条设计原则（复刻必须保留）

| # | 原则 | 具体表现 | 为什么不能丢 |
|---|---|---|---|
| P1 | **状态词即调度** | 通道文件第一个合法 `状态` 值决定 owner：`待执行/待修正→Claude`、`待审核→Codex`、`待放行/待裁决→人`、`排队/空闲/闭环→无人` | 一个字段同时表达「谁该动」，减少同步面 |
| P2 | **机械可判** | 收网只判四件事：回执 mtime、范围、代码正则、回退点祖先关系 | AI 判 AI 会互相包庇；机械判定可复现 |
| P3 | **单轮心跳** | 一次运行 = 一轮，跑完退出，没有 `while`；单实例锁防重叠 | 崩溃只损失一轮，没有长驻进程的状态腐坏 |
| P4 | **异常不重试** | 未捕获异常 → ALERT + 停机；异常停机不自动重跑 | 自动重试会把一次错误放大成 N 次烧钱 |
| P5 | **不自动回滚** | 中止只 kill，回滚由人决定 | 回滚不可逆，且工作区可能是半成品 |
| P6 | **开工前先要退路** | 工作区必须干净、回退点必须可追溯 | 没有可信回退点就没有安全网 |
| P7 | **等谁只报警** | owner 是人/Codex 时只发 ALERT，每 30 分钟重报 | 喊一次就静默一整天是 2026-09-19 的事故形状 |
| P8 | **控制面与项目面分离** | 状态机/锁留在 `mcp\守夜人\`；逐轮日志/原始输出留在项目内 `日志\` | 项目可迁移，控制面不污染仓库 |
| P9 | **唯一真源** | `可改` 字段是范围闸门的唯一真源；通道文件是任务唯一入口 | 两套实现必然漂移 |
| P10 | **缩权执行** | 执行端只给白名单工具，禁止 bypass 权限 | 无人值守时，权限就是最后一道墙 |

### 2.3 运行时拓扑（原系统）

```
┌──────────────────────── 控制面 C:\个人\mcp\守夜人\ ────────────────────────┐
│  _watch_projects.json   项目注册表（active + 4 个项目）                     │
│  _watch_state.json      状态机（原子写）                                    │
│  _watch.lock            单实例锁                                           │
│  watch.py               一轮调度器（cron 型）                               │
│  watchctl.py            控制层（启停/频率/状态/日志/中止/下单/放行）         │
│  watchdesk.py           tkinter 控制台窗口                                  │
│  agent_auditor.py       34 条正则闸门（命令 18 + 代码 12 + 范围 1 组）       │
│  守夜人控制台.bat         %~dp0 启动器（纯 ASCII）                          │
└───────────────────────────────────────────────────────────────────────────┘
        │ 读/写                         │ 拉起                    │ 读
        ▼                               ▼                         ▼
┌────────────────┐        ┌────────────────────────┐   ┌──────────────────┐
│ 项目仓库        │        │ runner 子进程           │   │ 个人 MCP          │
│  记忆库\给Claude│        │  watch.py --run-child   │   │  watch_status    │
│  记忆库\给Codex │        │    └─ claude.cmd -p ... │   │  (只读状态机)     │
│  日志\守夜人-*  │        └────────────────────────┘   └──────────────────┘
│  日志\runs\*.log│
└────────────────┘
        ▲
        │ 每 1 分钟
┌────────────────────────┐
│ 计划任务 GMYH-Watcher   │  pythonw.exe watch.py
│ (PT1M / 电池条件已关)    │
└────────────────────────┘
```

### 2.4 一轮的完整决策链（`watch.py:707-829`）

按顺序，任一步命中即 return：

1. **取锁**：`take_lock()` 拿不到直接退出（防止上一轮未结束）；陈旧锁超过 `2×TASK_TIMEOUT` 自动清。
2. **日切**：`day` 变了 → `wakes=0, alerted=False`。
3. **廉价短路**：`cheap_skip()` 三条件同时成立 → 直接退出，零子进程、不写盘、不写日志。
4. **有 PID 且存活** → `watch_during_run()`：越界 / 丢回退点 / 卡死，三查一，命中即 kill。
5. **有 PID 但已消失** → 判定收尾：
   - runlog 没有 `# 退出码` 行 → **异常停机**（被中断），不按验收结果处理；
   - 退出码非 0 → **异常停机**（FAILED）；
   - 退出码 0 → `machine_check()` → **已收网** PASS/FAIL。
6. **前置文档闸门**：项目根、双通道、日志目录、git 仓库、日志目录被 git 忽略 —— 缺一个就停机叫人。
7. **读通道 + 状态机**：
   - 无合法状态值 → 心跳；
   - owner 是人/Codex → 只报警（30 分钟重报）；
   - 非唤醒态 → 记 `idle_clean + chan_hash`，心跳；
   - 唤醒态 → 继续。
8. **判停线**：同一任务指纹且 phase ∈ {已收网, 异常停机, 已中止} → 只报警，不唤醒。
9. **熔断闸**：`fails >= 3` → 停机叫人。
10. **预算闸**：`wakes >= 12/天` → 停机叫人。
11. **工作区脏**：有任何未提交改动 → 拒绝开工（没有可信回退点）。
12. **开工**：记录 `task / declared / head_before`，清等待计时，`start_claude()`；命令过不了出口闸门就不算开工、不占配额；成功才 `wakes+1`。

---

## 3. 复刻目标与边界

### 3.1 必须保留的性质（验收时逐条打勾）

| 性质 | 可观测判据 |
|---|---|
| 无人值守 | 计划任务连续跑 24 小时，无人工干预，日志连续 |
| 幂等 | 同一任务指纹重复出现 N 次，只执行 0 次（收网后）或 1 次（未收网） |
| 可回退 | 每轮开工前记录 `head_before`，收网时验证它仍在 HEAD 祖先链 |
| 可审计 | 每轮有日志、每次执行有原始输出、每个判定有依据 |
| 不越界 | 执行端动「可改」之外的文件 → 运行中即 kill |
| 不自动烧钱 | 无新任务时零唤醒；异常后不自愈式重试 |
| 可移植 | 项目根可换，控制面不写进项目仓库 |

### 3.2 可以替换的实现

| 原实现 | 复刻替换 | 替换理由 |
|---|---|---|
| `claude.cmd -p` | `qodercli -p`（或 Qoder Quest 手动模式） | 赛场只允许 Qoder |
| Claude Code 权限参数 | Qoder CLI 工具白名单 + hooks | 参数模型不同，必须重新表达 |
| MCP `programmer/auditor` | 可原样挂到 Qoder（支持 MCP） | 减少重写 |
| tkinter 控制台 | 保留桌面控制台 + Qoder Sites 只读看板 | 比赛要求可公开展示 |
| Windows 计划任务 | Windows 计划任务（主）/ cron（备） | 跨平台 |

### 3.3 明确不复刻（红线）

1. **不做自动合并 / 自动放行**：放行永远是人或审核侧的动作。
2. **不做自动回滚**：中止只 kill 进程树，回滚留给人。
3. **不做任务内容理解**：守夜人不解析任务正文，只解析状态词与可改清单。
4. **不做本地模型裁决**：硬编码闸门是唯一可信基线；任何语义模型只能「升级为需人看」，不能降级、不能直接 BLOCK。
5. **不用** `--yolo` **/ 等价 bypass 权限**：无人值守 + 无权限墙 = 事故。

---

## 4. 总体架构

### 4.1 分层图（复刻版）

```
┌──────────────────────────── 展示层 ────────────────────────────┐
│  watchdesk.py（桌面控制台）      Qoder Sites 只读看板（新增）      │
│  启用/禁用/频率/立即扫/中止      状态、最近日志、告警、任务历史     │
│  下单/历史/一键放行              脱敏快照，不暴露本机路径与令牌      │
└──────────────────────────────┬─────────────────────────────────┘
                               │ import / HTTP（只读）
┌──────────────────────────────▼─────────────────────────────────┐
│                         控制层 watchctl.py                       │
│  task_state() / enable() / disable() / run_now() / set_interval()│
│  abort_run() / summary() / preflight() / create_task() / release()│
└──────────────────────────────┬─────────────────────────────────┘
                               │ import（只用解析器） / 读写文件
┌──────────────────────────────▼─────────────────────────────────┐
│                    调度层 watch.py（一轮即退出）                  │
│  tick() 决策链 → 闸门 → 拉起 → 看护 → 收网 → 写状态              │
└───────┬──────────────────────┬──────────────────────┬───────────┘
        │ import               │ spawn                │ 读/写
┌───────▼─────────┐   ┌────────▼─────────┐   ┌────────▼─────────┐
│ 闸门模块         │   │ runner 子进程     │   │ 项目面            │
│ agent_auditor.py │   │ watch.py         │   │ 通道 in/out       │
│ 18+12+1 组正则   │   │  --run-child     │   │ 日志/守夜人-*.md  │
└─────────────────┘   │   └─ agent_adapter│   │ 日志/runs/*.log  │
                      │        └─ qodercli│   └──────────────────┘
                      └───────────────────┘
                               │
                      ┌────────▼─────────┐
                      │ Qoder 侧          │
                      │ CLI / Quest / MCP │
                      │ hooks（工具边界）  │
                      └──────────────────┘
```

### 4.2 模块清单与依赖矩阵

| 模块 | 职责 | 输入 | 输出 | 依赖 | 失败模式 |
|---|---|---|---|---|---|
| `watch.py` | 一轮调度 | 注册表、状态机、通道、git | 状态机、日志、runner | agent_auditor、git、计划任务 | 异常 → ALERT 停机 |
| `agent_auditor.py` | 机械闸门 | 命令串 / 代码文本 / 两份文件清单 | ALLOW/WARN/BLOCK | 无 | 纯函数，异常向上抛 |
| `agent_adapter.py`[新增] | 执行端适配 | 提示词、权限档 | 命令行 + 环境变量 | qodercli | CLI 不在 → 拒绝开工 |
| `watchctl.py` | 控制层 | 注册表、状态机、任务库 | schtasks 命令、通道文件、git commit | watch（解析器）、PowerShell | 返回 (rc, out)，不抛 |
| `watchdesk.py` | 桌面 UI | watchctl | 用户动作 | tkinter | 崩溃写 `_watchdesk_error.log` |
| `plan-task` | 心跳触发 | — | 拉起 watch.py | OS 调度器 | 电池条件 → 永不跑 |
| `qodercli` | 执行端 | 提示词 | stdout/退出码 | Qoder 账号/模型 | 未登录、限流、模型不可用 |
| MCP 服务（可选） | 扩展工具 | JSON-RPC | 工具结果 | Qoder MCP 配置 | 工具不可用 → 执行端报错 |

**依赖方向（禁止反向）**：

```
watchdesk → watchctl → watch → agent_auditor
                     ↘ agent_adapter → qodercli
watch 不 import watchctl；watchctl 只 import watch 的纯解析函数
```

> 原系统正是这样：`watchctl._verify_channel()` 直接 `import watch` 复用解析器，保证「下单」写出的通道块一定能被「调度」读懂（`watchctl.py:414-428`）。这是防两套实现漂移的核心。

### 4.3 关键时序

**正常一发：**

```
计划任务 ──> tick ──读状态──> 待执行
                ├─ precheck 通过
                ├─ 判停线：指纹新
                ├─ 工作区干净
                ├─ 记 head_before / declared
                └─ 拉起 runner（detached） ──> tick 退出
runner ──> qodercli -p "读通道…执行…写回执"
                └─ stdout 逐行落盘
N 分钟后 tick ──> PID 存活 ──> watch_during_run（三查）
...agent 退出...
tick ──> PID 消失 + 退出码 0 ──> machine_check
                ├─ 回执 mtime > started
                ├─ 范围 ALLOW
                ├─ 代码无 BLOCK
                └─ head_before 仍是祖先
        ──> 已收网 PASS ──> 日志「等人工复核」
```

**返工：**

```
通道状态改为「待修正」──> 任务指纹变化
tick ──> 唤醒态 ──> 判停线不命中（指纹不同）
     ──> 开工 ──> build_prompt("待修正") 指向「返工单」
```

**异常停机：**

```
runner 被杀 / 超时 / 启动失败
tick ──> PID 消失
     ├─ runlog 无退出码 → INTERRUPTED
     └─ 退出码非 0     → FAILED
     ──> fails+1 ──> 熔断计数
     ──> 同一指纹不再唤醒（判停线）
```

**等人：**

```
状态=待放行/待裁决 ──> owner=人
tick ──> waiting_alert：立即报一次，之后每 30 分钟重报
     ──> 不拉任何 agent、不耗预算、不占唤醒次数
```

### 4.4 控制面 vs 项目面（必须照抄的分界）

| 内容 | 位置 | 是否进项目 git | 理由 |
|---|---|---|---|
| `_watch_state.json` | 控制面 | 否 | 守夜人自己的状态，不是项目产物 |
| `_watch.lock` | 控制面 | 否 | 单实例锁 |
| `_watch_projects.json` | 控制面 | 否 | 多项目注册表 |
| 逐轮日志 `守夜人-YYYY-MM-DD.md` | 项目内 `日志\` | **必须 gitignore** | 人读审计，但不能被 `git add -A` 带进仓库 |
| `日志\runs\*.log` | 项目内 `日志\runs\` | **必须 gitignore** | 原始输出，可能含密钥/路径 |
| 通道 in/out | 项目 `记忆库\` | 是 | 任务与回执要随项目版本走 |
| 交付物 | 项目工作区 | 由任务决定 | 审核放行后才提交 |

> 原系统的前置闸门会用 `git check-ignore` 机械校验日志目录被忽略（`watch.py:523-556`），不是靠「我记得加了 .gitignore」。

---

## 5. 数据契约

> 这一节是复刻的「接口规范」。任何实现都必须逐字满足，否则模块之间会各说各话。

### 5.1 通道文件（唯一任务入口）

#### 5.1.1 位置与方向

| 文件 | 方向 | 写入者 | 读取者 |
|---|---|---|---|
| `记忆库/给Claude的交接说明.md`（复刻后可改名 `给执行端的交接说明.md`） | Codex/人 → 执行端 | 审核侧、控制台「下单」 | `watch.py` |
| `记忆库/给Codex的交接说明.md` | 执行端 → 审核侧 | 执行端（覆盖写） | 人、审核侧 |

**约定**：任务文件每次**直接覆盖，不追加**；历史任务靠 git 历史翻。

#### 5.1.2 最小块格式

```markdown
## 方向确认 00X（标题）

- **范围:** 这一发要做什么（一句话）
- **边界:** 不许做什么（含「`server/` 一律不动」这类禁止项）
- **可改:** `path/a.py`, `path/b.js`
- **状态:** 待执行

### 验收（机械可判，逐条自己跑一遍）

1. …
2. …

### 交付后

把回执覆盖写入 `记忆库/给Codex的交接说明.md`。
```

#### 5.1.3 解析规则（必须逐字实现）

1. **字段提取**：`field(line, name)` 的正则等价于

   `\*\*<name>[:：]?\*\*\s*[:：]?\s*(.+)`
    （`watch.py:292-295`）。
2. **状态扫描**：从文件第一行开始，找**第一个**满足「`状态` 字段值截断到 `（`、`(`、空白 后 ∈ 合法词汇表」的行（`watch.py:297-309`）。
   - 带后缀也认：`排队（003 收口后执行）` → `排队`。
   - 非法值（如「随便写写」）**不**算状态，继续往下找。
3. **块边界**：状态行往上找到最近的 `##`  作为块首，往下找到下一个 `##`  作为块尾；找不到就退回整份（`watch.py:311-324`）。
4. **范围/边界/可改**：从状态行往上最多 15 行，分别取**最近一次**出现的 `范围` / `边界` / `可改`（`watch.py:353-368`）。
5. **可改清单**（范围闸门唯一真源，`watch.py:370-402`）：
   - 有 `可改` 字段 → **只认反引号里的路径**，不加后缀白名单、不跑兜底正则；
   - 没有 `可改` → 退回从 `范围 + 边界` 抠反引号路径，再补 `public/...` 正则，且只保留后缀在白名单里的项；
   - 跳过含 `*` 的项、以 `/api` 开头的项；
   - 去重、排序。
   - **反例教训**：边界里的「`server/` 一律不动」是禁止项，不能被当成声明范围，否则越界写反而被放行。

#### 5.1.4 状态词汇表与 owner

| 状态 | owner | 是否唤醒 | 行为 |
|---|---|---|---|
| `待执行` | 执行端 | 是 | 正常开工 |
| `待修正` | 执行端 | 是 | 返工；提示词改指「返工单」 |
| `待审核` | 审核侧 | 否 | 只报警；阶段 2 未接线 |
| `待放行` | 人 | 否 | 只报警，30 分钟重报 |
| `待裁决` | 人 | 否 | 同上 |
| `排队` | 无 | 否 | 心跳 |
| `空闲` | 无 | 否 | 心跳 |
| `闭环` | 无 | 否 | 心跳 |
| `待补充` | 无 | 否 | 控制台下单未填「可改」时的占位 |
| `已中止` / `已放行` / `已闭环` | **非唤醒态** | 否 | 用于收网后清场（建议值） |

> 词汇表来自 `watch.py:75-90`。复刻时若改词，必须同步改三处：`OWNER`、`WAKE_STATES`、控制台下拉与模板。

#### 5.1.5 任务指纹（判停线的核心）

```python
section_text = "\n".join(lines[start:end]).strip()
task_fingerprint = sha1(section_text.encode("utf-8")).hexdigest()[:12]
```

- **只算被解析到的那一节**，不算整个文件（`watch.py:326-351`）。
- 首尾 `strip()`：下发下一发时「空行 + `## 标题`」不会改变上一节指纹。
- 节内任何改动（改状态、改一个字）都会变指纹 → 判停线失效。这是 P2 修复后仍存在的边界，靠「收网后把状态改成非活动值」的卫生习惯兜底。

### 5.2 状态机文件 `_watch_state.json`

原子写：先写 `.tmp` 再 `os.replace`（`watch.py:198-207`）。

| 字段 | 类型 | 写入者 | 语义 |
|---|---|---|---|
| `day` | `YYYY-MM-DD` | tick | 日切；变了就重置 `wakes` / `alerted` |
| `wakes` | int | tick | 当日真实唤醒次数（过闸才算） |
| `alerted` | bool | 兼容旧逻辑 | 日常告警当日抑制；等人告警不受它抑制 |
| `task` | 12 位 hex | tick | 当前/最近任务指纹 |
| `declared` | list[str] | tick | 开工时解析出的「可改」清单 |
| `head_before` | 40 位 hex | tick | 开工前 HEAD，回退点 |
| `touched_seen` | list[str] | tick / 看护 | 本轮见过的改动文件快照 |
| `idle_clean` | bool | tick / mark_active | 上一轮是否「真没事」；任何实活/告警清零 |
| `chan_hash` | 12 位 hex | tick | 全文件指纹，仅用于廉价短路 |
| `pid` | int/null | tick | runner PID |
| `started_ts` | float | tick | 开工时间戳（秒） |
| `runlog` | str | tick | 原始输出路径 |
| `phase` | str | tick | `已开工` / `已收网` / `异常停机` / `已中止` |
| `verdict` | str | tick | `PASS` / `FAIL` / `FAILED` / `INTERRUPTED` / `ABORTED` |
| `fails` | int | tick / abort | 连续失败计数；成功收网清零 |
| `warns` | int | tick | 本次收网的 WARN 条数 |
| `wait_state` | str/null | tick | 当前等待的 key（状态词或「判停线·指纹」） |
| `wait_since` | float | tick | 等待起点 |
| `wait_alert_ts` | float | tick | 上次重报时间 |
| `last_seen_ts` | float | 看护 | 最近一次存活心跳 |
| `abort_why` / `abort_ts` | str/float | abort | 中止原因与时间（可选） |
| `local_verdict` | str | 升级模块（未开工） | 语义初审结果，**不写通道** |

**恢复语义**：文件损坏/读不到 → `{}`；`day` 为空 → 视为新一天，`wakes=0`。这有「重复唤醒」风险，原系统用原子写规避断电损坏，但没有状态备份。

### 5.3 项目注册表 `_watch_projects.json`

```json
{
  "active": "python学习",
  "projects": [
    {
      "name": "python学习",
      "root": "C:\\个人\\python学习",
      "channel_in": "记忆库\\给Claude的交接说明.md",
      "channel_out": "记忆库\\给Codex的交接说明.md",
      "log_dir": "日志"
    }
  ]
}
```

- 读不到 / 格式坏 / active 不存在 → 退回内置默认（原系统是「星图」，`watch.py:130-140`）。
- 复刻建议：默认项目改成一次「演示项目」，避免开箱即指向真实仓库。
- `apply_project()` 会把通道一级目录、日志目录、`_tmp/` 加入 `IGNORE`，避免守夜人自己的产物触发「工作区脏」（`watch.py:143-163`）。

### 5.4 日志与 runlog

**逐轮日志**：`<项目>/日志/守夜人-YYYY-MM-DD.md`，追加写，每行：

```
- [2026-11-17 20:00:03] 检测到「待执行」任务 b2189850d738，声明文件 [...], 回退点 34357ee
- [2026-11-17 20:01:03] 心跳：任务还在跑（PID 1234，已 60 秒）｜改动 2 个文件 / 范围闸门 ALLOW｜HEAD 34357ee
- [2026-11-17 20:02:03] **ALERT** 当前状态「待放行」：该人处理…已等 0 分钟…
```

**runlog**：`<项目>/日志/runs/<task>-YYYYMMDD-HHMMSS.log`，结构：

```
# 2026-11-17 20:00:05 拉起 Claude
# 命令行: <cli> -p <提示词> <权限参数>
# 实时流（边跑边落盘；stderr 已合并进正文）

<agent stdout 逐行>
...
# 退出码 0（2026-11-17 20:12:00）
```

- `# 退出码` **行必须最后写，且被杀时不写**——`parse_exit_code()` 靠它区分「跑完」与「被杀」（`watch.py:440-449`）。
- 逐行 flush：中断损失从「全丢」压到「丢最后几秒」（`watch.py:646-704`）。

### 5.5 任务库 `relay.db`（可选模块）

原系统的控制台「下单/历史」直连项目内 SQLite（`<项目>/data/relay.db`），DDL：

```sql
CREATE TABLE tasks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  created_at TEXT, instruction TEXT, files_json TEXT, acceptance_json TEXT,
  status TEXT, finished_at TEXT, result TEXT
);
CREATE TABLE events (
  id INTEGER PRIMARY KEY AUTOINCREMENT, task_id INTEGER, "at" TEXT,
  kind TEXT, detail TEXT
);
```

- 写库 → 写通道文件 → **用** `watch.parse_channel` **回读校验** → 失败则还原原文（`watchctl.py:430-481`）。
- 复刻时可以砍掉这个模块：第一版用「手写通道文件」即可；控制台下单是增强项。

### 5.6 契约变更规则

| 变更 | 必须同步改 |
|---|---|
| 新增状态词 | `OWNER`、`WAKE_STATES`、通道模板、控制台下拉、自检断言 |
| 改字段名 | 解析器 `field()` 调用点、通道模板、`create_task` 模板、测试样本 |
| 改可改清单格式 | `declared_files()`、自检中的历史判例、所有项目模板 |
| 改 runlog 结束标记 | `parse_exit_code()`、runner、收尾判定测试 |
| 改项目注册表 | `load_project()`、`apply_project()`、preflight、控制台 |

---

## 6. 模块详设

### 6.1 `watch.py` / `tick()` —— 一轮调度器

**入口与模式**（`watch.py:955-964`）：

| 参数 | 行为 |
|---|---|
| 无参数 | 正式一轮 |
| `--dry-run` | 只打印将做的动作，不写盘、不起进程 |
| `--self-test` | 跑内置断言 |
| `--run-child <runlog>` | 短命 runner：拉起执行端并逐行落盘 |

**主流程伪代码**：

```python
def tick(dry=False):
    if not dry and not take_lock(): return
    try:
        st = load_state()
        if st.day != today: st.wakes = 0; st.alerted = False; st.day = today
        if cheap_skip(st): return
        pid = st.pid
        if alive(pid):
            watch_during_run(st); return
        if pid:
            classify_and_collect(st); return      # 跑完/被杀 → 收网/异常停机
        if missing := precheck_docs():
            waiting_alert(...); return
        status, scope, bound, allow = parse_channel(read(CH_IN))
        owner = OWNER.get(status)
        if not status: heartbeat(); return
        if owner in ("人","codex"): waiting_alert(...); return
        if status not in WAKE_STATES: heartbeat_clean(); return
        task = channel_task_key(text)
        if st.task == task and st.phase in ("已收网","异常停机","已中止"):
            waiting_alert("判停线·"+task); return
        if st.fails >= 3: waiting_alert("熔断"); return
        if st.wakes >= 12: waiting_alert("预算闸"); return
        if dirty := touched_files(): waiting_alert("工作区脏"); return
        head = git("rev-parse","HEAD")
        st.task, st.declared, st.head_before = task, declared_files(...), head
        if start_claude(st, dry, status): st.wakes += 1; st.phase = "已开工"
    except Exception as e:
        alert(f"未捕获异常，停机不重试：{e}")
    finally:
        drop_lock()
```

**关键不变量**：

- 每一轮结束时 `drop_lock()` 必执行。
- 任何异常 → ALERT + 停机，不重试（P4）。
- `wakes` 只在真正 `start_claude` 成功后 +1；被闸门拦下不占配额。
- 收网后不自动改通道状态，也不自动提交 git。

### 6.2 基础设施

| 函数 | 行为 | 复刻要点 |
|---|---|---|
| `stamp()` | `YYYY-MM-DD HH:MM:SS` | 本地时区 |
| `log()` | 打印 + 追加项目日志 | dry-run 只打印 |
| `alert()` | `**ALERT**` 前缀 | 控制台靠它抽红条 |
| `load_state()` | JSON 读，坏则 `{}` | 不抛 |
| `save_state()` | `.tmp` + `os.replace` | 原子写，防断电损坏 |
| `waiting_alert()` | 按 key 去重，30 分钟重报 | 不受 `alerted` 抑制；key 变则重新计时 |
| `mark_active()` | `idle_clean=False` | 任何实活/告警都清零 |
| `chan_hash()` | 全文件 sha1 前 12 | 只用于廉价短路 |
| `cheap_skip()` | pid 空 + idle_clean + 通道指纹一致 | 三条件缺一不可 |
| `take_lock()` | `O_CREAT|O_EXCL` 标记文件；陈旧锁 2×超时后清 | Windows 上不要用 `os.kill(pid,0)` |
| `alive(pid)` | `tasklist /FI "PID eq N"` | 原系统实测可靠 |

### 6.3 通道解析器

已在 §5.1 展开。复刻时**必须原样移植**，并保留自检断言（原系统 `watch.py:836-860` 有 17 条断言，覆盖：带后缀状态、非法状态、禁止项不当声明、`.txt`/`.env.example` 不被丢、P2 指纹、块切分）。

### 6.4 工作区读数

```python
def touched_files():
    rc, out = git("--no-optional-locks", "status", "--porcelain", "-uall")
    return porcelain_paths(out)
```

- **必须** `-uall`：否则 git 把新增目录折叠成 `public/`，范围闸门只能比对到目录。
- **必须** `core.quotepath=false`：否则中文路径变成八进制转义。
- `IGNORE`：通道一级目录、日志目录、`_tmp/` 不计入改动。
- 已知限制：重命名显示为 `a.js -> b.js`，范围闸门可能误判；复刻时可保留，作为已知行为记录。

### 6.5 过程看护（「守夜」的本体）

`watch_during_run()`（`watch.py:477-521`）在任务运行期间每分钟执行：

1. **越界**（最高优先）：`audit_scope(declared, touched)` 判 BLOCK → `taskkill /F /T`。
2. **丢回退点**：`git merge-base --is-ancestor head_before HEAD` 非 0 → kill。
3. **卡死**：已跑 > `STALL_MINUTES`（45）且工作区零改动 → kill。
4. 否则记心跳：PID、已跑秒数、改动数、范围闸门判定、HEAD。

**设计取舍**：

- 只判「有没有超出可改清单」，不判断改动好不好。
- 越界优先于其它两条：先止血。
- 45 分钟是**活性阈值**，不是安全闸门；原系统实测「先读 100 KB 规格」的健康状态会 12–19 分钟零改动，所以从 15 调到 25 再调到 45。
- 每轮仍有 `TASK_TIMEOUT=3600s` 兜底。

### 6.6 机械验收（收网）

`machine_check()`（`watch.py:557-601`）只判四件事：

| # | 判据 | 失败含义 | 是否判 FAIL |
|---|---|---|---|
| 1 | 回执文件 mtime > 开工时间 | 执行端没写回执 | 是 |
| 2 | 范围闸门 = ALLOW | 越界 / 声明解析不出（UNKNOWN 也按失败） | 是 |
| 3 | 代码闸门无 BLOCK | 命中密钥硬编码 / SQL 拼接 / XSS / 关 TLS 等 | 是 |
| 4 | `head_before` 仍是 HEAD 祖先 | 历史被改写 / 回退点丢失 | 是 |

- WARN **不判失败**，但必须冒泡到日志与控制台（`warns` 字段）。
- 代码闸门只扫描被改动且后缀为 `.js/.html/.css/.mjs/.cjs/.json` 的文件（`watch.py:579-590`）。**Python 项目需要扩展** `.py`；这是复刻时的已知缺口，建议在适配层加白名单，但要同步更新测试。
- 收网结果只写状态机与日志，**不自动改通道状态、不自动提交**。

### 6.7 调度器：拉人与 runner

`start_claude()`**（**`watch.py:622-644`**）**：

1. `build_prompt(status)` 生成提示词（整段一个参数，**内部不能有双引号**）。
2. `gate_command(cmd)` 过出口闸门：BLOCK 拒绝，WARN 放行并记日志。
3. 生成 runlog 路径。
4. **不自己 Popen 执行端并重定向到文件**——原系统实测 `claude -p` 在文件句柄下一个字都不写，只有管道能接到。
5. 拉起 detached runner：`pythonw watch.py --run-child <runlog>`，立刻返回。

`run_child()`**（**`watch.py:646-704`**）**：

- 自己重新 `apply_project(load_project())`（独立进程）。
- `Popen(cmd, stdout=PIPE, stderr=STDOUT, text=True, bufsize=1)`，逐行读、逐行 flush。
- 独立 `threading.Timer(TASK_TIMEOUT, kill)`：子进程闷住不出声时读循环会阻塞，不能靠读循环判超时。
- 结束时写 `# 退出码 N`；被强杀时不写。
- Qoder 适配时保持同一模式，只换 `cmd` 的构造。

**提示词模板**（`watch.py:605-614`）：

```
读取 <通道> 里「待执行」的那一节（返工时是「待修正」的返工单），
严格按其中的范围、边界与验收执行：
你当总指挥、调用编程 agent 实现；开工前先提交回退点；
完成后自己跑完全部验收，并按通道约定把回报覆盖写入 <回执>。
本轮只做那一节写的事，其余排队项一律不做。
```

### 6.8 出口闸门（34 条正则）

`agent_auditor.py` 的三组纯函数，**直接 import，不走 MCP**：

| 函数 | 规则数 | 输入 | 输出 |
|---|---|---|---|
| `audit_command(command)` | 18 | 命令行串 | ALLOW / WARN / BLOCK |
| `audit_code(text, path)` | 12 + 1 条特殊（无 WHERE 的 DELETE） | 代码文本 | 同上 |
| `audit_scope(declared, touched)` | 1 组 | 两份逗号分隔清单 | 超范围即 BLOCK |

**复刻要点**：

- 判定词必须从 `判定: (ALLOW|WARN|BLOCK)` 里取，不能直接 `"BLOCK" in text`——结尾样板文字本身含「命中 BLOCK」。
- 正则清单见附录 C，**逐条照抄**，不要「优化」成一条大正则。
- 出口闸门只拦守夜人自己执行的命令；执行端内部跑的命令它拦不到。复刻时用 **Qoder hooks** 补第二层（§7.4）。

### 6.9 控制层 `watchctl.py`

| 能力                                               | 实现要点                                                                         |
| ------------------------------------------------ | ---------------------------------------------------------------------------- |
| `task_state()`                                   | PowerShell `Get-ScheduledTask` → JSON；**不要刮本地化的** `schtasks /v` **文本**       |
| `enable/disable`                                 | `schtasks /change /enable                                                    |
| `run_now`                                        | `schtasks /run`                                                              |
| `set_interval`                                   | `Register-ScheduledTask -Force` 重建；必须 `-AllowStartIfOnBatteries`；重建后按原状态恢复启停 |
| `abort_run`                                      | 读状态机 pid → `taskkill /F /T`；不手改状态，让下一轮自己判「被中断」                               |
| `summary`                                        | 给控制台一屏                                                                       |
| `preflight`                                      | 项目根/通道/日志/pythonw/计划任务五项                                                     |
| `tail_cycle_log` / `tail_runs` / `recent_alerts` | 读日志尾部                                                                        |
| `create_task` / `release`                        | 可选；下单走通道模板 + 回读校验                                                            |

**编码坑**：子进程输出先试 UTF-8，再 `cp936`/`mbcs`；PowerShell 用 `-EncodedCommand` 绕开引号与中文。

### 6.10 控制台 `watchdesk.py`

- **线程模型**：tkinter 不是线程安全的；后台线程只往 `queue.Queue` 塞数据，主线程 `_pump()` 每 120ms 取。
- **刷新**：每 2 秒后台读 `summary/logs/runs`，主线程绘制。
- **动作**：启用/禁用/改频/立即扫/中止/下单/历史/一键放行；动作期间 `busy` 互斥。
- **崩溃**：`pythonw` 无控制台，未捕获异常写 `_watchdesk_error.log`。
- **启动器**：`.bat` 内不能出现中文路径（cmd 按 OEM 代码页读），用 `%~dp0`。

### 6.11 自检体系（复刻的「可信」来源）

| 文件 | 覆盖 |
|---|---|
| `watch.py --self-test` | 状态机、解析、三闸门接线、存活、权限档、提示词、收尾判定、P2 指纹（约 40 条断言） |
| `watch_tick_test.py` | 临时 git 仓库上跑 `tick()`，18 条路径 A–R（见 §9.1） |
| `watchctl.py --self-test` | 路径/任务/状态/日志/告警/前置检查/任务往返 7 组 |
| `watch_stress_test.py` | 单一/组合/双重压力；杀的是自己起的假 runner |
| `watch_stream_test.py` / `watch_channel_key_replay.py` | 流式落盘 / 指纹回放 |
| `探针.py` | 对设计缺陷扎针，`--live` 才真拉会话 |

**复刻时最低要求**：保留 `watch.py --self-test` + 18 条集成路径 + 一条端到端演练。没有自检的复刻，等于把原系统的教训重踩一遍。

---

## 7. Qoder 适配层（复刻的核心差异）

> **原则**：把 AI 调用收敛到一个文件，其余模块对「执行端是谁」无感知。这样赛场换模型、换 CLI、换权限模型，只改一个文件。

### 7.1 能力映射表

| 守夜人需求     | 原实现                                                       | Qoder 侧实现                                             | 验证状态                   |
| --------- | --------------------------------------------------------- | ----------------------------------------------------- | ---------------------- |
| 无头拉起执行端   | `claude.cmd -p "<prompt>"`                                | `qodercli -p "<prompt>"`                              | `-p` 为官方 Print 模式，已见文档 |
| 文本输出      | stdout                                                    | `--output-format=text`（默认）                            | 待核实默认值                 |
| 结构化输出     | 无                                                         | `--output-format=json                                 | stream-json`           |
| 工具白名单     | `--allowedTools Bash(git *) Read Write Edit ...`          | `--allowed-tools=...` + `--disallowed-tools=...`      | 标志存在，取值语法待核实           |
| 权限模式      | `--permission-mode acceptEdits --permission-prompts none` | Qoder 权限模式（名称待核实）                                     | **待核实**                |
| 禁止 bypass | 不加 `--dangerously-skip-permissions`                       | **不使用** `--yolo`                                      | 需写进代码红线                |
| 工作目录      | `cwd=项目根`                                                 | `cwd=项目根` / `-w <dir>`                                | 标志见社区文档                |
| 回合上限      | 无（靠墙钟）                                                    | `--max-turns=N` 可选                                    | 待核实                    |
| 会话续跑      | 无                                                         | `-c` / `-r <session-id>`                              | **设计上不用**（幂等优先）        |
| MCP       | `.mcp.json`                                               | `qodercli mcp add` / `QODER_MCP_CONFIG` / `.mcp.json` | 文档支持                   |
| 工具边界拦截    | 无（只有启动命令闸门）                                               | **hooks：PreToolUse，exit 2 阻断**                        | 社区资料，**待核实**           |
| 长任务       | 墙钟 3600s 强杀                                               | 同 + 可选 Quest Mode                                     | Quest 为 GUI 长任务        |
| 模型选择      | CLI 默认                                                    | `--model` / `QODER_MODEL`                             | 待核实可用值                 |
| 账号/限流     | 登录态                                                       | `QODER_PERSONAL_ACCESS_TOKEN` 等                       | 待核实                    |

> Qoder CLI 的中文发行可能叫 `qoderclicn`；适配器启动时按 `qodercli` → `qoderclicn` 顺序探测，找不到就拒绝开工（ALERT），**不要静默降级**。

### 7.2 适配器接口 `agent_adapter.py`[新增]

```python
# 只做三件事：探测、构造命令、声明权限。不碰状态机、不写日志。
def detect() -> dict:            # {"ok": bool, "cli": "qodercli", "version": "...", "error": "..."}
def build_command(prompt: str, mode: str) -> list[str]:   # mode: "exec" | "fix"
def permission_profile() -> dict: # 语义能力 -> Qoder 参数映射
def env_overrides() -> dict:      # NO_COLOR、QODER_MCP_CONFIG、模型等
```

**约束**：

1. `build_command` 返回的命令必须能通过 `audit_command()` 出口闸门（原系统自检里有一条正是「守夜人自己那条命令过不了闸门」）。
2. 提示词整段一个参数，内部禁止双引号（原系统的 `claude -p` 参数拆断教训）。
3. `detect()` 失败 → `start_claude` 返回 False，ALERT，不占唤醒配额。
4. 所有 Qoder 专属标志集中在这个文件，`watch.py` 只认接口。

### 7.3 权限档（必须显式写死）

**允许（语义能力）**：

| 能力 | 用途 | Qoder 映射（待核实） |
|---|---|---|
| 读 | 读项目文件、通道、规格 | `Read` / `READ` |
| 写 / 编辑 | 写交付物 | `Write` / `Edit` |
| 搜索 | Glob / Grep | `Glob` / `Grep` |
| git 只读与提交前检查 | `git status/diff/log/merge-base` | `Bash(git *)` 或 `--allowed-tools` 的等价语法 |
| 自验收 | `node`/`python`/`npm` 跑验收脚本 | 对应 Bash 白名单 |

**禁止**：

- `--yolo`、任何「跳过权限确认」的等价开关。
- 联网抓取（比赛要求纯本地/允许数据源；执行端不装依赖、不访问外网）。
- 删除文件、强推、改历史、注册持久化任务（命令闸门已 BLOCK）。
- 写 `记忆库/`、`日志/`、`raw_data/` 等非「可改」目录（范围闸门已 BLOCK）。

### 7.4 hooks：工具边界第二道闸门（关键增强）

原系统的出口闸门只能拦住「守夜人自己拼的那条命令」，执行端内部跑什么它看不见（`watch.py:616-621` 的注释明确承认这一点）。Qoder 支持 hooks（社区资料；官方页面待核实），可在 `PreToolUse` 读取工具调用 JSON，对 Bash 命令跑 `audit_command()`，命中 BLOCK 时 `exit 2` 阻断。

**建议实现**：

```python
# qoder_hook_pre_tool.py  —— 只做一件事：把命令交给同一套正则
import json, sys
from agent_auditor import audit_command, level_of
payload = json.load(sys.stdin)
if payload.get("tool_name") in ("Bash", "shell", "terminal"):
    cmd = payload.get("tool_input", {}).get("command", "")
    if level_of(audit_command(cmd)) == "BLOCK":
        print("BLOCK: 命中守夜人出口闸门", file=sys.stderr)
        sys.exit(2)          # 具体阻断语义以 Qoder 版本为准
print("ok")
```

**为什么值得做**：这把「越界只能靠事后 git status 发现」升级为「危险命令在发生前被拦」。但它依赖 hooks 语义，**必须在目标版本实测**：JSON 字段名、`exit 2` 是否阻断、是否影响 `-p` 模式。

### 7.5 MCP 复用

| 服务 | 用途 | 复刻动作 |
|---|---|---|
| `agent_auditor.py` | 审核工具（只读） | 可作为 MCP 挂给 Qoder；但守夜人本体仍直接 import，不绕 MCP |
| `agent_programmer.py` | 编程 Agent（写代码） | 可选；原系统实测在无人值守链路里两次不可用，不建议依赖 |
| `personal` | 个人工作区只读 | 可选；与比赛无关可去掉 |

**挂载方式（Qoder）**：`qodercli mcp add <name> -- <python> <server.py>`，或写 `.mcp.json` / `QODER_MCP_CONFIG`。工具名 `mcp__<server>__<tool>`，权限里按前缀放行。

### 7.6 Quest Mode 与 CLI 的分工

| 用途 | 工具 | 理由 |
|---|---|---|
| 开发守夜人本身 | Qoder IDE + Quest Mode | 长任务、可规划、可看 diff |
| 运行时执行端 | Qoder CLI `-p` | 无头、可脚本化、退出码可判 |
| 人工兜底 | Qoder IDE 打开通道文件手动执行 | CLI 不可用时的降级路径 |
| 展示 | Qoder Sites | 比赛要求公开可访问 |

**不要把运行时执行端做成 Quest Mode**：Quest 是 IDE 内的长任务会话，没有稳定的「无头退出码」契约，无法满足 `parse_exit_code()` 的「跑完/被杀」判定。

### 7.7 会话与幂等

- **每次任务新会话**：不复用 `-c/-r`。理由：任务指纹 + 工作区状态才是真源，续接旧会话会带进上一发的上下文污染。
- 若 Qoder CLI 有自动会话保存，runlog 里记录 session id 仅供排查，不作为幂等依据。
- 同一指纹收网后不再唤醒，靠状态机，不靠会话。

### 7.8 赛场合规

- **AI 调用点只有一处**：`agent_adapter` → Qoder CLI/Quest。其余全是标准库 Python，不含模型。
- **不引入其他 AI**：不调用外部 LLM API、不装本地模型（原系统的「升级模块」明确未开工，复刻不启用）。
- **数据边界**：项目内容只在本机与 Qoder 之间流转；不把通道文件/日志上传到第三方。
- **比赛条款**：以天池赛题页原文为准；本文只做工程约束，不替代规则解读。

### 7.9 必须实测的清单（开工第一天做完）

| # | 测什么 | 通过标准 |
|---|---|---|
| Q1 | `qodercli --help` / `qoderclicn --help` | 找到与本文映射的等价标志 |
| Q2 | `qodercli -p "只输出 OK"` | 非交互退出、stdout 可读、退出码 0 |
| Q3 | `-p` 模式下把 stdout 接到文件句柄 | 是否像 Claude 一样「一个字都不写」；若如此，必须走管道 |
| Q4 | 工具白名单语法 | 只允许 Read/Write/Edit + git/node/python，其他被拒 |
| Q5 | 越权写一个未授权文件 | 被拒绝，且退出码/输出可判 |
| Q6 | 无网络环境 | 执行端不因联网失败而挂 |
| Q7 | hooks PreToolUse | 给一个 `rm -rf` 命令，能否阻断 |
| Q8 | 超时强杀 | 拉起后 kill 进程树，runlog 无退出码行 |
| Q9 | 中文路径/UTF-8 | 通道文件、日志、项目根均不出现乱码 |
| Q10 | MCP 挂载（可选） | `mcp list` 能看到，工具可调用 |

**Q3 是架构分叉点**：如果 Qoder CLI 也「文件句柄下不写」，必须沿用原系统的 runner + 管道模式；如果它正常写，仍建议保留 runner，因为逐行落盘能减少中断损失。

---

## 8. 部署与运行

### 8.1 目录布局（复刻版）

```
<控制面根>\                     例：C:\个人\qoder-watcher\
├─ watch.py                    一轮调度器
├─ watchctl.py                 控制层
├─ watchdesk.py                桌面控制台
├─ agent_auditor.py            机械闸门
├─ agent_adapter.py            Qoder 适配器[新增]
├─ qoder_hook_pre_tool.py      hooks 闸门[新增]
├─ _watch_projects.json        项目注册表
├─ _watch_state.json           状态机（运行时生成）
├─ _watch.lock                 锁（运行时生成）
├─ 守夜人控制台.bat            纯 ASCII 启动器
└─ 自检\
   ├─ watch_tick_test.py
   ├─ watch_stress_test.py
   └─ e2e_demo.py

<项目根>\                       例：C:\个人\python学习\
├─ 记忆库\给执行端的交接说明.md  任务入口
├─ 记忆库\给审核侧的交接说明.md  回执出口
├─ 日志\                        gitignore
│  ├─ 守夜人-YYYY-MM-DD.md
│  └─ runs\
└─ .gitignore                   必须含 `日志/`
```

### 8.2 安装步骤

1. 安装 Node ≥ 20.18.1，`npm i -g @qoder-ai/qodercli`（中文版 `qoderclicn` 以官方为准）。
2. 登录 Qoder，确认 `qodercli -p "OK"` 可非交互运行。
3. 建控制面目录，放入 5 个 Python 文件 + 1 个 bat。
4. 建演示项目，放双通道文件、`日志/`、`.gitignore`，`git init` 并提交初始快照。
5. 写 `_watch_projects.json`，`active` 指向演示项目。
6. 跑 `python watch.py --dry-run`，确认只打印、不写盘、不起进程。
7. 跑 `python watch.py --self-test`、`python watchctl.py --self-test`、`python watch_tick_test.py`。
8. 注册计划任务（见下），先设 5 分钟，人工观察 3 轮。
9. 再设 1 分钟；观察 30 分钟无异常后进入无人值守。

### 8.3 计划任务

**Windows（主）**：

```powershell
$pyw = "C:\...\pythonw.exe"
$watch = "C:\个人\qoder-watcher\watch.py"
$action = New-ScheduledTaskAction -Execute $pyw -Argument "`"$watch`""
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
  -RepetitionInterval (New-TimeSpan -Minutes 1)
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries `
  -DontStopIfGoingOnBatteries -StartWhenAvailable `
  -MultipleInstances IgnoreNew -ExecutionTimeLimit (New-TimeSpan -Minutes 5)
Register-ScheduledTask -TaskName "Qoder-Watcher" -Action $action `
  -Trigger $trigger -Settings $settings -Force
```

**Linux（备）**：

```cron
* * * * * cd /opt/qoder-watcher && /usr/bin/python3 watch.py >> /var/log/qoder-watcher.cron 2>&1
```

**必须避开**：

- 计划任务里不要用 `cmd /c <中文路径>.bat`（OEM 代码页解码坏，任务返回 0 但什么都没起）。
- 不要用 `schtasks /create` 默认设置（`DisallowStartIfOnBatteries=true`，电池上永不跑）。
- 改频率用 `Register-ScheduledTask -Force` 后，按原状态恢复启停（重建默认是启用）。

### 8.4 日常操作手册

| 动作 | 命令 / 界面 |
|---|---|
| 启用 | 控制台「启用」/ `schtasks /change /tn Qoder-Watcher /enable` |
| 禁用 | 控制台「禁用」 |
| 改频率 | 控制台下拉（选中即生效） |
| 立即扫一轮 | 控制台「立即扫一轮」/ `schtasks /run` |
| 中止当前任务 | 控制台「中止当前任务」（只 kill，不回滚） |
| 下单 | 手写通道文件，或控制台「下单」 |
| 放行 | 人/审核侧改状态为「已闭环」并提交；控制台「一键放行」是便捷路径 |
| 清场 | 把已收网那一节的状态改成非活动值（如 `已中止` / `已放行`），避免指纹守卫被后续编辑绕过 |

---

## 9. 测试与验收

### 9.1 集成测试矩阵（照抄原系统 A–R）

| 路径 | 场景 | 期望 |
|---|---|---|
| A | 干净工作区 + 待执行 | 拉起 runner，`wakes+1` |
| B | 排队项 | 什么都不做 |
| E | 同一任务已收网未清 | 判停线报警，不重复拉人 |
| F | 今日唤醒达上限 | 预算闸，停机叫人 |
| G | 收网时代码 WARN | WARN 冒泡，但不判 FAIL |
| H | 回退点被丢弃 | 收网 FAIL |
| I | dry-run 全程 | 不往磁盘写日志 |
| J | runlog 无退出码 | 判「被中断」，不按验收结果处理 |
| K | 退出码非 0 | 异常停机 FAILED |
| L | 连续失败达阈值 | 熔断，停机 |
| M | 待放行 | 只报警，不拉人 |
| N | 待审核 | 只报警（阶段 2 未接线） |
| O | 排队 | 心跳 |
| P | 非法状态值 | 当无事处理 |
| Q | 待修正 | 走唤醒路径，提示词指向返工单 |
| R | 缺信息通道 | 前置闸门拒绝操作 |

> 原文件 `watch_tick_test.py` 覆盖 18 条（A–R，其中 C/D 合并在别的断言里，I 是收尾断言）。复刻时逐条保留，尤其 E/J/K/L。

### 9.2 端到端演练剧本

**剧本 1：正常一发**

1. 演示仓库，干净工作区，状态 `待执行`，可改列 2 个文件。
2. 启用计划任务，观察：检测到任务 → 拉起 → 心跳 → 收网 PASS。
3. 验证：回执 mtime 新、范围一致、回退点仍在、日志有「等人工复核」。

**剧本 2：越界止血**

1. 任务只允许改 `a.txt`，执行端故意写 `b.txt`。
2. 期望：下一次心跳（≤1 分钟）判 BLOCK → taskkill → 状态 `已中止` → ALERT。
3. 验证：`b.txt` 仍在工作区（不自动回滚），但进程已死。

**剧本 3：被杀与判停**

1. 执行端跑到一半，手工 kill runner。
2. 期望：下一轮发现 PID 消失且 runlog 无退出码 → `异常停机 / INTERRUPTED`。
3. 期望：同一任务指纹不再唤醒，只报警；直到人清场或下新单。

**剧本 4：等人重报**

1. 状态 `待放行`，启用。
2. 期望：立即 ALERT 一次；30 分钟后再次 ALERT；不拉任何 agent。

### 9.3 故障注入清单

| 注入 | 期望 |
|---|---|
| 杀 runner / 杀 qodercli | 下一轮判「被中断」，不验收 |
| 断网 | 执行端退出码非 0 → 异常停机；不自愈 |
| 状态机文件写坏 | `load_state()` 返回空，日切重置；记录风险 |
| 锁文件残留 | 超过 2×超时自动清 |
| 通道文件半写 | 解析不到状态 → 心跳；不误唤醒 |
| 工作区脏 | 拒绝开工，ALERT |
| 日志目录未被 gitignore | 前置闸门拒绝操作 |
| 磁盘满 | 写入异常 → tick 异常 → ALERT 停机 |

### 9.4 评委演示脚本（5 分钟）

1. **30 秒**：打开 Qoder Sites 看板，展示「当前状态：已收网 PASS / 今日唤醒 1 次 / 判停线」。
2. **90 秒**：现场改通道为 `待执行`，点「立即扫一轮」，展示 tick 日志逐行出现：检测 → 记录回退点 → 拉起。
3. **90 秒**：切换到「越界」演示（预置分支），展示心跳判 BLOCK → kill → `已中止`。
4. **60 秒**：展示 runlog 原文、回执、四道机械验收。
5. **30 秒**：强调三条设计红线：不自动重试、不自动回滚、不自动放行。

---

## 10. 安全与风险

### 10.1 信任边界

```
[人/审核侧] --写通道--> [项目仓库] --读--> [守夜人]
      ^                                        |
      |                                        ├--拉起--> [Qoder 执行端]
      |                                        |              |
      +---------- 回执 / 日志 / 告警 <---------+              +--写--> 工作区
```

- 守夜人**不信任**执行端的自述；只信 git 状态、文件 mtime、退出码。
- 守夜人**不信任**通道文件里的「边界」描述，只信机械解析出的「可改」清单。
- 人**不信任**守夜人的 PASS；收网后仍需人工复核再放行。

### 10.2 已知风险与对策

| 风险 | 原系统事实 | 复刻对策 |
|---|---|---|
| 幽灵重跑 | 005 收网后 3 分钟又被跑一遍（整文件指纹） | 小节指纹 + 收网后改非活动值 |
| 状态词卫生 | 通道还写「待执行」，靠判停线挡 | 放行流程强制改成 `已放行`；控制台一键放行 |
| 工作区脏导致无回退点 | 判例 1 | 开工前机械检查，拒绝开工 |
| 执行端闷住 | 读循环阻塞会让超时失效 | 独立计时器强杀 |
| 首文件延迟误判卡死 | 健康状态下 12–19 分钟零改动 | 45 分钟阈值 + 3600s 兜底 |
| 代码闸门漏 Python | 只扫 js/html/css/mjs/cjs/json | 复刻扩展 `.py`，并加测试 |
| 范围匹配过宽 | `endswith` 双向匹配可能误放行 | 保留并记录；后续可改规范化路径比较 |
| runlog 含敏感信息 | 原始输出可能有路径/令牌 | 日志目录 gitignore；Sites 只放脱敏快照 |
| Qoder hooks 语义不稳 | 社区资料，非官方定论 | 视为增强项；失败不影响主链路 |
| CLI 权限标志变化 | Qoder 版本迭代 | 适配器集中映射 + 启动探测 + Q10 清单 |

### 10.3 数据与隐私

- 看板只读、只放：状态机摘要、经过脱敏的日志尾部、任务编号；**不放**本机绝对路径、账号、令牌、完整原始输出。
- 原始 runlog 留在本机项目 `日志/runs/`，不进 git、不上传。
- 比赛提交前跑一次「敏感串扫描」（`C:\Users\`、`token`、`password`、`sk-`、本机用户名）。

---

## 11. 里程碑（按 2026-11-30 18:00 截止倒排）

> 假设参加的是「2026 Qoder 高校创造力大赛」（报名 10-29 至 11-30，需用 Qoder 完成并部署到 Qoder Sites）。若实际是其他赛事，以赛题页时间为准。

| 里程碑 | 日期 | 交付 | 验收 |
|---|---|---|---|
| M0 骨架 | 11-17 | 目录、5 个 Python 文件、注册表、演示项目、适配器探测 | `--dry-run` 不写盘；Q1–Q3 有结论 |
| M1 单轮闭环 | 11-18 | tick 状态机 + 通道解析 + cheap_skip + 锁 | 自检通过；A/B/O/P 路径过 |
| M2 三道闸门 | 11-20 | 过程看护 + 收网 + 出口闸门 | E/F/G/H/J/K/L/M/N/Q/R 过 |
| M3 Qoder 接入 | 11-22 | agent_adapter + runner + 权限档 + hooks | Q4–Q9 过；剧本 1 真跑通 |
| M4 控制台与看板 | 11-25 | watchdesk 可用 + Sites 看板 | 启用/禁用/下单/放行/中止可点 |
| M5 压测与演练 | 11-27 | 24h 无人值守 + 三个剧本 + 故障注入 | 日志连续、无重复唤醒、越界必杀 |
| M6 材料提交 | 11-29 | 设计文档、演示视频/截图、代码、Sites 链接 | 11-30 18:00 前提交 |

**最小可交版本**：M0–M3 + 剧本 1/2 + Sites 看板。控制台和下单库可以砍。

---

## 12. 附录

### A. 常量表（原系统实测值）

| 常量 | 值 | 含义 | 出处 |
|---|---|---|---|
| `MAX_WAKE_PER_DAY` | 12 | 一天最多真实唤醒 | `watch.py:96` |
| `TASK_TIMEOUT` | 3600 s | 单次任务墙钟上限 | `watch.py:97` |
| `CONSEC_FAIL_MAX` | 3 | 连续失败熔断阈值 | `watch.py:98` |
| `STALL_MINUTES` | 45 | 零改动判卡死 | `watch.py:125` |
| `WAIT_ALERT_INTERVAL` | 1800 s | 等人/等修重报间隔 | `watch.py:92` |
| `RUNNING_INTERVAL` | 300 s | 跑动期看护下限（实际受计划任务频率限制） | `watch.py:128` |
| `ALLOWED_INTERVALS` | 1/5/15/30/60 分 | 控制台可选频率 | `watchctl.py:25` |
| 日志观察窗口 | 300 行 | 控制台逐轮日志 | `watchdesk.py:27` |
| 原始输出窗口 | 60 行 | 控制台 runs | `watchdesk.py:28` |
| 控制台刷新 | 2 s | 状态轮询 | `watchdesk.py:25` |

### B. 状态迁移表

| 当前 phase | 事件 | 下一 phase | verdict | fails |
|---|---|---|---|---|
| （无） | 检测到唤醒态 | 已开工 | — | 不变 |
| 已开工 | agent 退出码 0 + 验收通过 | 已收网 | PASS | 0 |
| 已开工 | agent 退出码 0 + 验收失败 | 已收网 | FAIL | +1 |
| 已开工 | 退出码非 0 | 异常停机 | FAILED | +1 |
| 已开工 | runlog 无退出码 | 异常停机 | INTERRUPTED | +1 |
| 已开工 | 越界/丢回退点/卡死 | 已中止 | ABORTED | +1 |
| 已收网/异常停机/已中止 | 同一指纹再次出现 | 不变 | — | — |
| 任意 | 人清场/新指纹 | 可再次开工 | — | 按上轮 |

### C. 正则清单（必须逐条照抄）

**命令类（18 条，节选判定档位）**：

| 档 | 规则 |
|---|---|
| BLOCK | 递归强制删除（rm -rf / Remove-Item -Recurse / rd /s / del /s） |
| BLOCK | 磁盘格式化/分区（format / diskpart / mkfs / fdisk） |
| BLOCK | 裸设备写入（dd if= / 裸盘覆写） |
| BLOCK | 数据库结构破坏（DROP TABLE/DATABASE/SCHEMA、TRUNCATE） |
| BLOCK | 丢弃/强推版本库改动（git reset --hard、git clean -f、push --force、checkout --） |
| BLOCK | 关机/重启（shutdown/reboot/halt/poweroff/Restart-Computer/Stop-Computer） |
| BLOCK | 注册表删除（reg delete、Remove-ItemProperty HKLM） |
| BLOCK | 远程脚本直通执行（curl/wget/iwr 管道到 sh/bash/iex/python） |
| BLOCK | 反弹 shell（nc -e） |
| WARN | 绕过执行策略（ExecutionPolicy Bypass / Set-ExecutionPolicy） |
| WARN | 安装依赖（npm/pnpm/yarn/pip install） |
| WARN | 注册持久化任务（schtasks / Register-ScheduledTask / New-Service） |
| WARN | 权限放宽（chmod 777 / icacls Everyone） |
| WARN | 强杀进程（kill/taskkill /F） |
| WARN | 凭据文件访问（id_rsa / credentials / id_ed25519） |

**代码类（12 条 + 1 特殊）**：

| 档 | 规则 |
|---|---|
| BLOCK | 密钥硬编码（api_key/secret/token/password = 12+ 字符串） |
| BLOCK | SQL 字符串拼接 |
| BLOCK | innerHTML 拼接（XSS） |
| BLOCK | 关闭 TLS 校验（rejectUnauthorized:false / verify=False / NODE_TLS_REJECT_UNAUTHORIZED） |
| BLOCK | 删列/改结构（DROP COLUMN / ALTER TABLE DROP） |
| BLOCK | 无 WHERE 的 DELETE（特殊判定） |
| WARN | 直接写 .env/.db |
| WARN | 动态执行（eval/new Function/exec） |
| WARN | 弱哈希（md5/sha1） |
| WARN | bcrypt 强度过低 |
| WARN | CORS 全放开 |
| WARN | 安全中间件关闭（csrf/xsrf/helmet/rateLimit） |
| WARN | 密钥兜底硬编码（process.env.X \|\| "长字符串"） |

> 完整正则见 `agent_auditor.py:111-148`，复刻时逐条复制，不要合并。

### D. 通道文件模板

```markdown
# 给执行端的交接说明（<项目名>）

> 通道约定：**这个文件 = 审核侧写给执行端的消息**，每次直接覆盖，不追加。
> 本文件只由最下方任务节的 `- **状态:**` 行驱动，上方历史节不参与状态机。

---

## 方向确认 001（标题）

- **范围:** …
- **边界:** 除「可改」所列文件外，其余文件一律不动；不装依赖、不联网、不提交 git。
- **可改:** `path/a.py`, `path/b.js`
- **状态:** 待执行

### 验收（机械可判，逐条自己跑一遍）

1. …
2. …

### 交付后

把回执覆盖写入 `记忆库/给审核侧的交接说明.md`，内容包括逐条 `项 | 状态 | 证据`、
`git status --porcelain -uall` 原文、以及未做/做不到/与规格的出入。

---

## 排队（未开工）

- 候选 A
- 候选 B

> 本节故意不写 `状态` 字段：合法状态值会抢在活动节之前被解析到。
```

### E. 职责矩阵（谁写谁读）

| 文件 | watch | watchctl | watchdesk | 人 | Qoder 执行端 |
|---|---|---|---|---|---|
| `_watch_projects.json` | 读 | 读 | 读 | 写 | — |
| `_watch_state.json` | 读写 | 读/abort | 读 | — | — |
| `_watch.lock` | 读写 | — | — | — | — |
| 通道 in | 读 | 读写 | 读 | 写 | 读 |
| 通道 out | 读 mtime | 读 | 读 | 读 | 写 |
| `日志/守夜人-*.md` | 写 | 读 | 读 | 读 | — |
| `日志/runs/*.log` | 写（runner） | 读 | 读 | 读 | 间接 |
| `relay.db`（可选） | — | 读写 | 读 | 读 | — |

### F. 原系统踩坑索引（每条 → 设计对策）

| # | 坑 | 对策 |
|---|---|---|
| 1 | `claude -p` 在文件句柄下一个字不写 | runner + PIPE + 逐行 flush |
| 2 | 跑完才写盘，被 kill 时全丢 | 边跑边落盘 |
| 3 | 读循环阻塞导致超时失效 | 独立 Timer 强杀 |
| 4 | 「跑完」与「被杀」分不清 | `# 退出码` 行哨兵 |
| 5 | 整文件指纹导致幽灵重跑 | 小节指纹 |
| 6 | 通道被改一个字节判停线失效 | 收网后改非活动值 + 放行流程 |
| 7 | 边界里的禁止项被当声明范围 | 「可改」唯一真源 |
| 8 | 后缀白名单丢掉 `.txt`/`.env.example` | 有「可改」就照单全收 |
| 9 | `git status` 折叠新目录 | `-uall` |
| 10 | 中文路径被吞 | `core.quotepath=false` + UTF-8 |
| 11 | 日志目录被 `git add -A` 带进仓库 | `check-ignore` 机械校验 |
| 12 | 电池条件导致任务永不跑 | `AllowStartIfOnBatteries` |
| 13 | 改频率悄悄启用任务 | 重建后恢复原启停 |
| 14 | `cmd /c 中文.bat` 解码坏 | 计划任务直接指定可执行文件 |
| 15 | 本地化 `schtasks` 文本匹配不上 | PowerShell 枚举 + JSON |
| 16 | `alive()` 用 `os.kill(pid,0)` 会真杀 | `tasklist` |
| 17 | 状态文件半写损坏 | `.tmp` + `os.replace` |
| 18 | 等人只喊一次然后静默一天 | 30 分钟重报，不受日常抑制影响 |
| 19 | 15/25 分钟卡死阈值误杀健康读规格 | 45 分钟 + 3600s 兜底 |
| 20 | pythonw 吞异常「窗口一闪就没」 | `_watchdesk_error.log` + excepthook |
| 21 | 控制台后台线程碰控件 | Queue + 主线程 pump |
| 22 | 缩权档缺 `Bash(python *)` 导致自验收跑不了 | 按项目栈补白名单，不加 bypass |

### G. 术语对照（原 → Qoder）

| 原系统 | 复刻后 | 说明 |
|---|---|---|
| Claude Code / `claude.cmd` | Qoder CLI / `qodercli` | 执行端 |
| `--allowedTools` | `--allowed-tools` | 参数名以实测为准 |
| `--permission-mode acceptEdits` | Qoder 权限模式 | 待核实 |
| `--dangerously-skip-permissions` | `--yolo` | 两者都禁用 |
| `.mcp.json` | `.mcp.json` / `QODER_MCP_CONFIG` | MCP 配置 |
| Claude Code hooks | Qoder hooks | 工具边界拦截 |
| 编程 Agent（子会话） | Qoder Expert / Subagent | 可选，不依赖 |
| Codex 审核侧 | 人 / 审核 Agent | 阶段 2 仍未接线 |

### H. 复刻检查清单（交付前逐项打勾）

**架构**

- [ ] AI 调用只出现在 `agent_adapter.py`
- [ ] `watch.py` 不 import `watchctl.py`
- [ ] 控制面文件不写入项目仓库
- [ ] 通道/日志/`_tmp` 在 `IGNORE` 里

**契约**

- [ ] 通道解析器逐字移植，17 条自检断言全过
- [ ] 状态词汇表、OWNER、WAKE_STATES 三处一致
- [ ] 任务指纹只算小节，P2 回放断言过
- [ ] 状态机原子写
- [ ] runlog 退出码哨兵

**闸门**

- [ ] 开工前：前置文档 / 判停 / 熔断 / 预算 / 脏工作区
- [ ] 跑动中：越界 / 丢回退点 / 卡死
- [ ] 收网：回执 / 范围 / 代码 / 回退点
- [ ] 出口：命令闸门 + Qoder hooks（若可用）

**Qoder**

- [ ] Q1–Q10 全部有结论并记录
- [ ] 权限档无 bypass
- [ ] 探测失败拒绝开工并 ALERT
- [ ] 中文路径/UTF-8 正常
- [ ] 超时强杀与退出码判定正常

**运行**

- [ ] 24 小时无人值守日志连续
- [ ] 同一任务不重复唤醒
- [ ] 越界 ≤1 分钟被杀
- [ ] 看板脱敏
- [ ] 提交材料含设计文档、演示、代码、Sites 链接

### I. 资料来源

**原系统源码（本机）**

- `C:\个人\mcp\守夜人\watch.py`
- `C:\个人\mcp\守夜人\watchctl.py`
- `C:\个人\mcp\守夜人\watchdesk.py`
- `C:\个人\mcp\守夜人\agent_auditor.py`
- `C:\个人\mcp\守夜人\watch_tick_test.py`
- `C:\个人\mcp\守夜人\升级模块-本地大模型嵌入.md`
- `C:\个人\mcp\记忆库\进度\2026-10-16.md` 等进度记录

**Qoder 能力（外部）**

- Qoder 产品文档：<https://docs.qoder.com/zh/product-series/what-is-qoder.md>
- Qoder 官网：<https://qoder.com/>
- Qoder CLI npm：<https://www.npmjs.com/package/@qoder-ai/qodercli>
- 阿里云帮助中心「使用 CLI」：<https://help.aliyun.com/zh/lingma/using-the-cli>
- Qoder CLI 参考（社区整理）：<https://github.com/rimio-ai/rimz/blob/main/docs/externals/agent-adapter/qoder-reference.md>
- 2026 Qoder 高校创造力大赛：<https://competehub.dev/zh/competitions/tianchi532536>
- 天池赛题页：<https://tianchi.aliyun.com/competition/>

> 外部资料中，Quest Mode、Repo Wiki、MCP、`-p` Print 模式、`--allowed-tools`/`--disallowed-tools`、`--output-format` 有官方或社区明确描述；hooks 的事件名与 `exit 2` 阻断语义主要来自社区整理，**必须在本机版本实测**。本文不把未验证项当作已知事实。

---

## 结语

复刻守夜人，不是复刻 2000 行代码，而是复刻它的**判断顺序**：先要退路，再看有没有活，再看该谁动，再看有没有被拦住，最后才拉起人。代码可以重写，这个顺序不能乱。

Qoder 只是把「拉起谁」换了一个名字。真正让这个系统成立的，是那三条永远不动的红线：**不自动重试、不自动回滚、不自动放行**。









注！此文档有且仅有洪帮丽有权阅览，切勿二次传播或借阅等手段，泄露
