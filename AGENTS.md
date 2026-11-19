# Applied-Energistics-Horizon

把 AE2 移植为 Horizon（Rust 版 Minecraft Java 26.3）mod。开工前先读 `GLOSSARY.md` 与 `docs/adr/`。

- 移植基线：`upstream/ae2`（git submodule，锁定 `v26.3.2-alpha`），克隆后运行 `git submodule update --init` 获取。
- M1 的 API 可行性调研：`docs/research/m1-api-feasibility.md`。

## Agent skills

### Issue tracker

Issues 存放在 GitHub Issues（`rt-game-440/Applied-Energistics-Horizon`），使用 `gh` CLI 操作。See `docs/agents/issue-tracker.md`.

### Triage labels

使用默认的五个 triage 标签：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`。See `docs/agents/triage-labels.md`.

### Domain docs

single-context：仓库根目录一个 `GLOSSARY.md` + `docs/adr/`（按需创建）。See `docs/agents/domain.md`.
