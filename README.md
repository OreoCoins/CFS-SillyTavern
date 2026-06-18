# Cache-Friendly Scanner (CFS)

SillyTavern 用的 prompt cache 优化脚本。包含两个独立部分：

- **v3.1.7（基线）**：PSIS 提示词结构守护、MVU 接口管理、自动 initvar 生命周期。把无 MVU 优化时常见的 25% 左右命中率拉到 85% 左右。和 MVU stat_data 渲染优化无关，纯靠把世界书 entry 推到 cache prefix 友好的位置实现。
- **v4.x（StatData Engine）**：把 MVU 每轮变化的 stat_data YAML 渲染（25K 字符上下）替换为跨轮稳定的 STABLE_BATCH 引用 token。在 v3.1.7 的 85% 基础上把命中率推到 95-97%（实测大卡 13 万字符 prompt 上稳态，单次峰值 98%）。

两部分一起装，缺一不可。

---

## 安装

1. SillyTavern → 脚本管理 → 导入
2. 选 `Cache-Friendly-Scanner.json`
3. 启用脚本 → F5 刷新酒馆

---

## 怎么用

全自动。F5 后几秒内 v4.x 接管。

工具栏会出现 `MVU 守护` 按钮。点开看到：

```
MVU 系统接口管理（N 条；常驻 a / 一次性 b）
  v4.x: 接管中  · 注入 141 字符 · 接管 1 条 mvu
  v4.x 启动状态：接管中 (用时 X.X s)
  当前: 角色卡 X 第 Y 轮
```

### 状态行说明

`v4.x:` 实时模式：

- 接管中 — 正常
- 未接管 — 启动初期 / 手动降级 / 卡不支持
- 已降级 — 自动故障转移，需要排查

`v4.x 启动状态:` 微内核状态：

- 接管中 — 一切正常
- 等待会话进入 — 还没进角色卡（welcome screen 正常状态）
- 未启用（本卡未使用 MVU） — 卡不带 MVU，PSIS 仍正常运行
- 接管失败 — Mvu 60s 未就绪，面板会出现「超时手动接管」按钮

---

## 出问题怎么办

### 等了一分钟还显示「接管失败」

打开 `MVU 守护` 面板，找「v4.x 启动状态」区块，点「超时手动接管」按钮。手机端没控制台也能用。

如果按钮也没让它接管成功，多半是 MVU 扩展本身没装、加载有问题、或者这张卡确实不用 MVU。

### 命中率突然从 96% 掉到 85-93%

通常是外部脚本（世界书缓存优化器 WM 之类）把 CFS entry 的位置改坏了。F12 控制台跑：

```js
await window.CFS4.Coordinator.auditEntries({ force: true })
```

返回 `{ fixed: N }` 表示修了 N 条，`{ fixed: 0 }` 表示位置本来就对。

### 想看 Coordinator 当前状态

```js
window.CFS4.Coordinator.getState()
```

### 想手动重建接管

```js
await window.CFS4.InjectionStrategy.bootstrapTakeover({ force: true })
```

### 想回退到 MVU 原渲染

```js
window.CFS4.FallbackStrategy.degradeToMvu({ reason: '手动回退' })
```

恢复：

```js
window.CFS4.FallbackStrategy.recoverToV4({ force: true })
```

---

## 工作原理

预设里有一条 worldbook entry 含 `{{format_message_variable::stat_data}}` macro，每轮被渲染成 25K 字符 YAML。这些字段大部分跨轮不变，但因为 YAML 整块每轮都重新拼，cache prefix 在这里被打断。

v4.x 用一条自管的 dynamic entry 替代它：

- 稳定 path 用 `<STABLE_BATCH schema="..." paths="..."/>` 引用 token 表达（约 141 字符）
- 本轮真正变了的字段单独列出 delta

引用 token 跨轮稳定，进 cache prefix；delta 体积小，落 chat 末尾不破坏 prefix。

设计要点：

- SCHEMA entry：`position=0` (before_char) + `role=0` (system) → 进 prefix
- DYNAMIC entry：`position=4` (atDepth) + `role=1` (user) = at_depth_as_user → 落 chat 末尾
- Schema 三锚点 + 位置锚（共 4 锚） + canonical JSON
- Path Registry 三态机：present / omitted / deleted
- 自动降级阈值：3 次注入失败（启动期事件不计）

---

## 共存

| 脚本 | 状态 | 备注 |
|---|---|---|
| SillyTavern + TavernHelper | 必装 | |
| MVU 主扩展（Mvu.getMvuData API） | 必装 | |
| 内置 PSIS R0/R1 | 自动豁免 | CFS 自管 entry 三锚点识别 |
| EWC autoSwitch（业火归途等） | 实测兼容 | |
| 世界书缓存优化器 WM | 协调 | 详见下方 |
| NE Memory Engine | 实测兼容 | NE 的 RAG 召回是独立请求 |
| 跨角色卡 | 实测兼容 | |

### 和 WM 共存

WM 不识别 CFS 的命名前缀，会按通用规则改 entry 位置：

1. 把 SCHEMA entry 从 `pos=0` 改成 `pos=4`，1KB 内容被推到 chat 末尾 miss 区
2. 把 DYNAMIC entry 从 `pos=4` 改成 `pos=0`，每轮 delta 把 prefix cache 打断

CFS 在 `worldinfo_updated` 事件触发后 500ms / 1500ms / 4000ms 各扫一次，加上每轮 `generate_before_combine_prompts` 钩内同步校对，能在 4 秒内把位置改回。

WM 按一次「应用修复」会让命中率掉一轮，第二轮 audit 修完就回归。

---

## 控制台 API

微内核（v2.x）：

| API | 用途 |
|---|---|
| `CFS4.Coordinator.getState()` | 启动状态机 + summary + 启动用时 |
| `CFS4.Coordinator.auditEntries({force:true})` | 手动审计并修正 entry 位置 |
| `CFS4.Coordinator.getAuditState()` | audit 跑过几次、上次时间 |
| `CFS4.SessionGate.probe()` | 当前会话探针 |
| `CFS4.NotificationCenter.flushStartup(phase)` | 强制弹一次启动报告 |

v4.x 核心：

| API | 用途 |
|---|---|
| `CFS4.FallbackStrategy.getCurrentMode()` | 当前 mode |
| `CFS4.InjectionStrategy.bootstrapTakeover({force:true})` | 手动接管 |
| `CFS4.InjectionStrategy.applyInjection()` | 手动跑一次注入 |
| `CFS4.InjectionStrategy.simulateInjection()` | 模拟（不真改）|
| `CFS4.InjectionStrategy.selfTest()` | 完整诊断 |
| `CFS4.HealthMonitor.getStatus()` | 健康状态 + 故障计数 |
| `CFS4.PathRegistry.getAll()` | 注册的 path 表 |
| `CFS4.FallbackStrategy.getState()` | mode + history |
| `CFS4.InjectionStrategy.listCandidateMvuEntries()` | 候选 MVU rendering entry |
| `CFS4.InjectionStrategy.setManagedMvuEntries([uid])` | 手动锁定接管的 entry |

---

## 故障排查

| 现象 | 处理 |
|---|---|
| F5 后 `v4.x 启动状态：等待会话进入` 一直不动 | 点角色卡进会话。已进会话仍不动，跑 `CFS4.SessionGate.probe()` 看返回 |
| F5 后 `v4.x: 未接管` 超过 30 秒 | 面板点「超时手动接管」按钮，或 console 跑 `bootstrapTakeover({force:true})` |
| 命中率从 96% 掉到 85-93% | 跑 `Coordinator.auditEntries({force:true})`，返回 `{fixed: N>0}` 说明修了 |
| 出现「已降级」不能自动恢复 | `HealthMonitor.getStatus()` 看故障历史，修完跑 `recoverToV4({force:true})` |
| 面板上 `[mvu_plot]普通审查` 显示「审查协议（用户决定）」 | v2.2+ 正常行为，由用户决定是否启用 |
| 启动期弹很多 toast | v1.x 老版本，升级到 v2.x |

---

## 卸载

ST → 脚本管理 → 关掉脚本 → F5。worldbook 内的自管 entry（comment 含 `[CFS4_SCHEMA|` 或 `[CFS4_DYNAMIC|`）需要手动删除，或者重启后跑：

```js
CFS4.SchemaFrozenLayer.cleanupLegacyEntries();
```

---

## 版本历史

| 版本 | 改动 |
|---|---|
| v3.1.7 | PSIS + MVU 接口管理 + 自动 initvar 生命周期。基线 25% → 85% |
| v4.0 | StatData Engine 协议层。85% → 96% 左右 |
| v2.0 | 微内核重构：SessionGate / Coordinator / NotificationCenter。启动期完全静默 |
| v2.1 | SessionGate 以 `getCurrentChid()` 为主判据，加多事件名监听 |
| v2.2 | writeSchema 用数字 position，verifyAnchors 加第 4 锚（位置锚）|
| v2.3 | audit 4 触发点，加 F12 手动入口 |
| v2.4 | audit 在 `generate_before_combine_prompts` 内同步前置（修「慢一步」问题）+ `worldinfo_updated` 钩 3 次延迟扫描。命中率稳态 95-97%，峰值 98% |
| v2.5 | 「MVU 守护」面板加「超时手动接管」按钮，手机端不开 F12 也能救 |

---

最后更新：2026-06-18
