# 代码审查报告 — 提交 211aff9a（feat/atomcode-provider）

- **审查对象**：`211aff9a` feat(atomcode): add provider support and keep live config in sync（19 个文件，+530 / -20 行）
- **审查日期**：2026-11-11
- **方式**：深度审查（correctness / security / reliability 多维度），仅报告未修改代码
- **去重后问题数**：4 个独立问题（2 个 P2，2 个 P3）

## 汇总

| 级别 | 问题 | 位置 |
|---|---|---|
| P2 | 复制 AtomCode 供应商后无法添加到 live 配置（模型 key 未重写） | `src/App.tsx:1324-1327` |
| P2 | 测试 failpoint 泄漏到同 crate 其他测试 | `src-tauri/src/services/provider/mod.rs:1324-1341` |
| P3 | AtomCode 原生写入路径缺少 per-app 锁 | `src-tauri/src/services/provider/mod.rs:5795-5812` |
| P3 | 表单 keyConflict 提示在冲突解决后不消失 | `AtomCodeProviderForm.tsx:210-218` |

## 详细发现

### 1.（P2）复制 AtomCode 供应商后无法添加到 live 配置

**位置**：`src/App.tsx:1324-1327`

复制 AtomCode 供应商走的是共存型分支，`settingsConfig.models` 被原样拷贝，模型 key 未像 Kimi 分支那样用新的 `providerKey` 重写。

由于 `~/.atomcode/config.toml` 中模型 key 全局唯一（`apply_provider` 会拒绝 key 已属于其他账户的写入），用户复制供应商后点「添加」会永远失败，直到手动逐个改 key。

**修复建议**：仿照 Kimi 分支，用新 `providerKey` 重写复制出来的模型 key。

### 2.（P2）测试 failpoint 泄漏到其他测试

**位置**：`src-tauri/src/services/provider/mod.rs:1324-1341`

`crash_at(Some("atomcode:write"))` 是进程全局状态，仅在测试最后一行手动 `crash_at(None)` 清除。前面任何断言失败 panic 后 failpoint 不会解除，导致同 crate 后续所有测试（串行运行）在 live 写入上误报失败。

**修复建议**：用 Drop guard（类似现有 `TempEnvOverride`）自动清除。

### 3.（P3）AtomCode 原生写入路径缺少 per-app 锁

**位置**：`src-tauri/src/services/provider/mod.rs:5795-5812`（`lock_native_provider_operations` 定义在 line 162）

`lock_native_provider_operations` 的 `matches!` 仍只覆盖 `Mcode | Kimi | Dsh`，未包含 `AppType::AtomCode`。文件写入本身有 WRITE_LOCK 串行化，但「live 写入 + db 提交」跨两个并发操作（如 update 与 switch 竞争）可能交错，造成 db 与 live 配置指向不同状态——正是新测试想防止的不一致。

**修复建议**：在 `matches!` 中加入 `AppType::AtomCode`。

### 4.（P3）表单 keyConflict 状态过期不清除

**位置**：`AtomCodeProviderForm.tsx:210-218`

触发重复 key 警告后，通过 `removeModel`、`choosePreset` 或 JSON 编辑解决冲突，「该模型键已存在」提示不会消失，直到用户再改一次 key。仅 UI 问题，不影响提交。

**修复建议**：在 `update()` 中统一 `setKeyConflict(false)`，或改为渲染时从 `config.models` 派生。

## 测试覆盖与 i18n

- i18n 七语言同步未发现问题。
- 问题 1、3 各缺少对应的失败场景测试（复制后添加失败、并发 update/switch 竞争），建议修复时一并补上。

## 建议的处理顺序

1. 问题 1（用户可见的功能性 bug）
2. 问题 3（数据一致性，改动小：一行 `matches!`）
3. 问题 2（测试稳定性，Drop guard）
4. 问题 4（UI 状态派生，低风险重构）
