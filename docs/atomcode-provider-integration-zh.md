# AtomCode Provider 接入 CC Switch

## 范围与现状

`atomcode` 是独立应用 ID。AtomCode TUI 以共存模式管理供应商：一个账号对应
`~/.atomcode/config.toml` 里的一个 `[provider_accounts.<key>]` 节点，其模型以
`account = "<key>"` 挂在 `[models."<model-key>"]` 下；切换供应商即把顶层的
`default_provider` / `default_model` 指到该账号。

| 功能 | 实现 | 数据来源 / 写入目标 |
| --- | --- | --- |
| 供应商共存 | 所有供应商共存于同一 config.toml；卡片「添加」只增删 live 节点并标记库里的 `live_config_managed`，不改 `default_model`；「移除」只删 live 节点、不删库记录 | `~/.atomcode/config.toml` |
| 切换默认 | 卡片「更多」菜单「设为默认」→ 后端 `switch_and_commit` 把 `default_provider` 与 `default_model` 一并指到该账号名下的某个模型键（以 `models_of` 选出的键为准，缺省回落该账号第一个模型） | 同上，库里的 current 指针同步回显 |
| 编辑回填 | 编辑走「先写 config.toml、成功后才提交库」的写序，live 写失败时库不动 | 同上 + `~/.cc-switch/cc-switch.db` |
| 导入 | 从 live 配置一键导入（`import_atomcode_providers_from_live`） | `~/.atomcode/config.toml` |
| MCP | 未接线：需要数据库 schema 新列（本项目禁止随意改 `SCHEMA_VERSION`）；`live_sync_apps()` 目前对 AtomCode 静默跳过 | — |
| Skills | 未接线 | — |

> ko / ru 为社区翻译包，AtomCode 相关文案暂回退英文，完整翻译待后续补齐。

## 验证

前端：

```bash
pnpm typecheck
pnpm format:check
pnpm build:renderer
pnpm test:unit
```

后端：

```bash
cd src-tauri
cargo fmt --check
cargo clippy -- -D warnings
CC_SWITCH_TEST_HOME="$(mktemp -d)" cargo test --lib atomcode_ -- --test-threads=1
```

写序回归覆盖 live 写失败、更新成功、`create` 与 `write_live=false` 四条分支。全量 `cargo test`、切换默认供应商后 AtomCode `/model` 列表的真机验证，以及界面截图，留待合并前补齐。
