# Cache-Friendly Scanner (CFS) v4.0

SillyTavern + MVU 场景的 prompt cache 优化脚本。把 prompt cache 命中率从 ~85% 推到 ~97%，推理成本约降 56%。

---

## 这是什么

CFS 在 prompt 拼装前抢跑一步，把每轮变化的 **stat_data YAML 渲染**（典型 25K 字符，占 prompt 13%）替换为跨轮稳定的 BATCH 引用 token。

结果：
- 每轮 prompt 减少约 25K 字符
- cache prefix 大幅延长 → 命中率 84.8% → 96.7%（实测）
- 跨卡通用 90% 稳态命中（实测）

CFS 内含两部分：
- **v3.1.7 基线**：PSIS（prompt 结构守护）+ MVU 接口管理 + 自动 initvar 生命周期
- **v4.x 引擎**（新）：StatData Engine — 协议层 + 弹性体系 + 真接管

---

## 安装

1. SillyTavern → **脚本管理** → **导入**
2. 选择 `Cache-Friendly-Scanner.json`
3. 启用脚本 → **F5** 刷新酒馆

---

## 使用

**全自动**。F5 后 15-30 秒内 v4.x 自动接管：
- 自动 autoRegister stat_data 字段
- 自动锁定 MVU 渲染 entry
- 自动启用 dynamic 注入

工具栏会出现 `🛡️ MVU 守护` 按钮。点击打开控制面板，可以看到：

```
🔧 MVU 系统接口管理（N 条；常驻 a / 一次性 b）
  v4.x: 🟢 接管中  · 注入 141 字符 · 接管 1 条 mvu
  📍 当前: 角色卡 X 第 Y 轮
  ...
```

`v4.x:` 状态行说明：
- 🟢 **接管中** — 一切正常，享受高命中率
- ⚪ **未接管** — 启动初期 / 手动降级 / 不支持的卡，走 MVU 原渲染（无副作用）
- 🟡 **已降级** — 自动故障转移，需要排查

ESC 键关闭面板。

---

## 出问题怎么办

**F12 console 救回**（兜底，30 秒后还没接管时用）：

```js
await window.CFS4.InjectionStrategy.bootstrapTakeover({force: true});
```

**回退到 MVU 原渲染**（保险开关，< 1 秒生效）：

```js
window.CFS4.FallbackStrategy.degradeToMvu({reason: '手动回退'});
```

**恢复 v4.x 接管**：

```js
window.CFS4.FallbackStrategy.recoverToV4({force: true});
```

---

## 工作原理

预设里的 stat_data 渲染 worldbook entry（含 `{{format_message_variable::stat_data}}` macro）每轮输出当前完整状态（25K 字符 YAML），其中绝大多数字段跨轮不变。v4.x 把它替换为一个动态 entry，内容是：
- 稳定 path 的 `<STABLE_BATCH schema="..." paths="..."/>` 引用 token（约 150 字符）
- 本轮真实变化字段的 delta

前者跨轮永驻 cache prefix，后者每轮变化但体积极小。

设计契约见 `protocol-contract-v3` 章节（在工程日志内）。简化要点：
- Schema 双锚点 + canonical JSON
- Path Registry 三态机：present / omitted / deleted
- 故障自动降级阈值：3 次注入失败

---

## 兼容性

| 项 | 状态 |
|---|---|
| SillyTavern + TavernHelper 扩展 | ✓ |
| MVU 主扩展（Mvu.getMvuData API） | ✓ |
| 跨卡通用 | ✓ 实测 |
| PSIS R0/R1 共存 | ✓ |
| EWC autoSwitch 共存 | ✓ |

---

## 控制台 API 速查

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
| F5 后 `v4.x: ⚪ 未接管` 持续超过 30 秒 | Mvu 加载特别慢，或者卡没用 MVU | console 跑 `bootstrapTakeover({force:true})` |
| 控制面板显示 `⚠️ 建议启用` 而不是 `✅ v4.x 接管中` | mvu entry 没被 v4.x keys 标记 | `listCandidateMvuEntries()` 找候选，再 `setManagedMvuEntries([uid])` |
| 命中率仍然 ~85%（没提升） | dynamic entry content 没每轮更新（钩没触发） | 每轮回复后手动 `applyInjection()`，或者上 console 查 `getLastInjection()` 验证 |
| 出现 `🟡 已降级` 不能自动恢复 | 连续 3 次注入失败触发降级 | `HealthMonitor.getStatus()` 看故障历史，修复后 `recoverToV4({force:true})` |
| CFS UI 把 v4.x 管理的 entry 报 `⚠️ 建议启用` | entry keys 还没加 `_cfs4_managed_mvu` 标记 | 跑一次 `applyInjection()` 即可写入标记 |

---

## 卸载

ST → 脚本管理 → 关闭脚本 → F5。worldbook 内的 v4.x 自管 entry（comment 含 `[CFS4_SCHEMA|` 或 `[CFS4_DYNAMIC|`）需要手动删除，或者重启后跑：

```js
CFS4.SchemaFrozenLayer.cleanupLegacyEntries();
```

---

最后更新：2026-06-18
