# Cache-Friendly Scanner (CFS) v4.x

SillyTavern + MVU 场景的 prompt cache 优化脚本。把 prompt cache 命中率从 ~85% 推到 **95-98%**，推理成本约降 50%+。

**最新实测**：缄默之秋 v1.7 大卡（13 万字符 prompt）连续 24 小时稳态 95-97%，单次峰值 **97.8%**。

---

## 这是什么

CFS 在 prompt 拼装前抢跑一步，把每轮变化的 **stat_data YAML 渲染**（典型 25K 字符，占 prompt 13%）替换为跨轮稳定的 BATCH 引用 token。

结果：
- 每轮 prompt 减少约 25K 字符
- cache prefix 大幅延长 → 命中率 84.8% → 96-98%（实测）
- 跨卡通用 90% 稳态命中（实测）

CFS 内含两部分：
- **v3.1.7 基线**：PSIS（prompt 结构守护）+ MVU 接口管理 + 自动 initvar 生命周期
- **v4.x 引擎**：StatData Engine（协议层 + 弹性体系 + 真接管）+ 微内核 + 插件总线 + **位置锚定 audit**

---

## v2.4 新特性（位置锚定 + 共存防御）

### 🛡️ 主动位置审计（Coordinator audit）

CFS 自管 worldbook entry 现在自带 **4 层位置防御**，即使被外部脚本（如世界书缓存优化器 WM、用户手动操作）改坏，也能在 **本轮 prompt 拼装前同步修复**：

| 层 | 机制 |
|---|---|
| ① 写入时锁定 | 数字 `position: 0` (SCHEMA) / `position: 4` (DYNAMIC) + `_cfs4_position_locked` keys + `extensions.cfs.expected_position` 声明 |
| ② verifyAnchors 第 4 锚 | 验证 position/role 是否漂移，不一致触发 `cfs_schema_drift_detected` |
| ③ Coordinator audit | 5 触发点（app_ready+3s / chat_changed / session_ready / macro_committed / `worldinfo_updated` 后 500ms·1500ms·4000ms），任一触发自动修回 |
| ④ 同步钩前置 | `generate_before_combine_prompts` 钩内 await audit，**本轮拼 prompt 前** entry 必正确 |

### 🎛️ 微内核 + 插件总线架构

- **SessionGate**：无状态探针，识别 ST 实际进入会话才允许接管（避免 welcome-screen 期间的疯狂弹窗）
- **Coordinator**：启动状态机 + 串行插件调度 + 看门狗指数退避 60s
- **NotificationCenter**：toast 唯一出口，启动期合并 + 运行期防抖
- **审查协议识别**：mvu_plot / logic_check 类自动识别为「用户决定」，不再误标"建议启用"

---

## 安装

1. SillyTavern → **脚本管理** → **导入**
2. 选择 `Cache-Friendly-Scanner.json`
3. 启用脚本 → **F5** 刷新酒馆

---

## 使用

**全自动**。F5 后 5 秒内 v4.x 自动接管：

工具栏会出现 `🛡️ MVU 守护` 按钮。点击打开控制面板：

```
🔧 MVU 系统接口管理（N 条；常驻 a / 一次性 b）
  v4.x: 🟢 接管中  · 注入 141 字符 · 接管 1 条 mvu
  v4.x 启动状态：✅ 接管中 (用时 X.X s)
  📍 当前: 角色卡 X 第 Y 轮
  ...
```

### 状态行说明

`v4.x:` 实时模式：
- 🟢 **接管中** — 一切正常，享受高命中率
- ⚪ **未接管** — 启动初期 / 手动降级 / 不支持的卡
- 🟡 **已降级** — 自动故障转移，需要排查

`v4.x 启动状态:` 微内核状态：
- ✅ **接管中** — Coordinator + audit 全部正常
- ⏳ **等待会话进入** — SessionGate 未识别到会话（welcome screen 期间正常）
- — **未启用（本卡未使用 MVU）** — 卡不带 MVU，PSIS 仍正常运行
- ❌ **接管失败** — Mvu 60s 未就绪，红字高亮 + 救回命令

ESC 键关闭面板。

---

## 出问题怎么办

### 命中率突降（怀疑 entry 被外部脚本改坏）

F12 console 强制 audit：

```js
await window.CFS4.Coordinator.auditEntries({ force: true })
// → { fixed: N, uids: [4, 12, ...] } 表示修了 N 条
// → { fixed: 0 } 表示位置已正确
```

查 audit 历史：

```js
window.CFS4.Coordinator.getAuditState()
// → { last_run: <timestamp>, run_count: N, debounce_ms: 5000 }
```

### 接管完全没起来

```js
// 看 Coordinator 状态
window.CFS4.Coordinator.getState()
// → phase 应该是 'DONE'，summary 含 v4_bootstrap.ok=true

// 强制重建（兜底）
await window.CFS4.InjectionStrategy.bootstrapTakeover({ force: true })
```

### 手动回退到 MVU 原渲染

```js
window.CFS4.FallbackStrategy.degradeToMvu({ reason: '手动回退' })
```

### 恢复 v4.x 接管

```js
window.CFS4.FallbackStrategy.recoverToV4({ force: true })
```

---

## 工作原理（一段话）

预设里的 stat_data 渲染 worldbook entry（含 `{{format_message_variable::stat_data}}` macro）每轮输出当前完整状态（25K 字符 YAML），其中绝大多数字段跨轮不变。v4.x 把它替换为一个动态 entry，内容是：
- 稳定 path 的 `<STABLE_BATCH schema="..." paths="..."/>` 引用 token（约 141 字符）
- 本轮真实变化字段的 mutable delta

前者跨轮永驻 cache prefix（通过位置锚定保护），后者每轮变化但体积极小（落在 cache miss 区不破坏 prefix）。

设计要点：
- **SCHEMA entry**：`position=0` (before_char) + `role=0` (system) → 进 prefix cache 区，与 char_defs 一起跨轮稳定
- **DYNAMIC entry**：`position=4` (atDepth) + `role=1` (user) = `at_depth_as_user` → 落 chat 末尾，每轮变 delta 不影响 prefix
- Schema 三锚点 + 位置锚（4 锚点）+ canonical JSON
- Path Registry 三态机：present / omitted / deleted
- 故障自动降级阈值：3 次注入失败（启动期事件不计）

---

## 与其他脚本共存

| 脚本 | 共存状态 | 备注 |
|---|---|---|
| SillyTavern + TavernHelper 扩展 | ✓ | 必装依赖 |
| MVU 主扩展（Mvu.getMvuData API） | ✓ | 必装依赖 |
| PSIS R0/R1（本脚本内置） | ✓ | CFS 自管 entry 三锚点豁免 |
| EWC autoSwitch（业火归途等） | ✓ 实测 | CFS 在 generate_before_combine_prompts 后置确认 |
| 世界书缓存优化器 WM | ✓ **协调** | WM 可能误改 CFS entry 位置 → audit 自动修回 |
| NE Memory Engine | ✓ 实测 | NE 的 RAG 召回是独立请求，与 CFS 主接管无冲突 |
| 跨卡通用 | ✓ 实测 | |

### WM 共存机制

如果你同时装了"世界书缓存优化器"或类似脚本：
1. WM 可能把 CFS `[CFS4_SCHEMA|*]` entry 从 `pos=0` 改成 `pos=4`（推到 chat 末尾 miss 区，1KB 内容永远 miss）
2. WM 可能把 CFS `[CFS4_DYNAMIC|*]` entry 从 `pos=4` 改成 `pos=0`（推到 prefix 区，每轮 delta 破坏 cache）
3. **CFS audit 在 ① `worldinfo_updated` 事件触发后 500ms/1500ms/4000ms 三次扫描 ② 每轮 `generate_before_combine_prompts` 钩内同步修正 → 4 秒内必修回**

如果你点了 WM 的"应用修复"按钮，命中率会暂时下降一轮（本轮 prompt 已拼装），第二轮 audit 修完后回归。

---

## 控制台 API 速查

### 微内核（v2.x 新增）

| API | 用途 |
|---|---|
| `CFS4.Coordinator.getState()` | 启动状态机 + summary + 启动用时 |
| `CFS4.Coordinator.auditEntries({force:true})` | 手动审计并修正 entry 位置 |
| `CFS4.Coordinator.getAuditState()` | audit 跑过几次、上次时间 |
| `CFS4.SessionGate.probe()` | 当前会话探针结果 |
| `CFS4.NotificationCenter.flushStartup(phase)` | 强制弹一次启动报告 |

### v4.x 核心

| API | 用途 |
|---|---|
| `CFS4.FallbackStrategy.getCurrentMode()` | 当前 mode |
| `CFS4.InjectionStrategy.bootstrapTakeover({force:true})` | 一键救回 |
| `CFS4.InjectionStrategy.applyInjection()` | 手动跑一次注入 |
| `CFS4.InjectionStrategy.simulateInjection()` | 模拟（不真改）|
| `CFS4.InjectionStrategy.selfTest()` | 完整诊断报告 |
| `CFS4.HealthMonitor.getStatus()` | 健康状态 + 故障计数 |
| `CFS4.PathRegistry.getAll()` | 当前注册 path 表 |
| `CFS4.FallbackStrategy.getState()` | mode + history 完整 state |
| `CFS4.InjectionStrategy.listCandidateMvuEntries()` | 列候选 MVU rendering entry |
| `CFS4.InjectionStrategy.setManagedMvuEntries([uid])` | 手动锁定 v4.x 接管的 entry |

---

## 故障排查

| 现象 | 原因 | 处理 |
|---|---|---|
| F5 后 `v4.x 启动状态：⏳ 等待会话进入` 持续 | SessionGate 没收到 chat_changed | 点角色卡进入会话；如已进会话仍 ⏳，跑 `CFS4.SessionGate.probe()` |
| F5 后 `v4.x: ⚪ 未接管` 持续超过 30 秒 | Mvu 加载特别慢 | console 跑 `bootstrapTakeover({force:true})` |
| 命中率从 96% 突降到 85-93% | 外部脚本（WM 等）改了 CFS entry 位置 | 跑 `Coordinator.auditEntries({force:true})`，应返回 `{fixed: N>0}` |
| 出现 `🟡 已降级` 不能自动恢复 | 连续 3 次注入失败触发降级 | `HealthMonitor.getStatus()` 看故障历史，修复后 `recoverToV4({force:true})` |
| 面板上 `[mvu_plot]普通审查` 显示「✋ 审查协议（用户决定）」 | v2.2+ 正常行为 | 这是审查协议类 entry，CFS 不再强建议启用，由用户决定 |
| 启动期弹很多 toast | 你装的是 v1.x 老版本 | 升级到 v2.x（微内核架构，启动期完全静默）|

---

## 卸载

ST → 脚本管理 → 关闭脚本 → F5。worldbook 内的 v4.x 自管 entry（comment 含 `[CFS4_SCHEMA|` 或 `[CFS4_DYNAMIC|`）需要手动删除，或者重启后跑：

```js
CFS4.SchemaFrozenLayer.cleanupLegacyEntries();
```

---

## 版本历史

| 版本 | 关键改动 |
|---|---|
| v4.0 baseline | StatData Engine 协议层完整 |
| v2.0 微内核 | 引入 SessionGate / Coordinator / NotificationCenter，启动期完全静默 |
| v2.1 | SessionGate 以 `getCurrentChid()` 为主判据 + 多事件名 + eventSource 兜底 |
| v2.2 | writeSchema 数字 position + verifyAnchors 第 4 锚 + audit + mvu_plot 误判修复 |
| v2.3 | audit 4 触发点 + console 日志 + F12 手动入口 |
| **v2.4** | **audit 同步前置（解决"慢一步修复"）+ `worldinfo_updated` 钩 + 3 次延迟扫描 → 命中率突破 97.8%** |

---

最后更新：2026-06-18
