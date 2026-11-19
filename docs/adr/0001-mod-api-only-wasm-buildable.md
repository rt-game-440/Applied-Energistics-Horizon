# 只使用 Horizon mod API，并始终保持可构建为 WASM

AE2-Horizon 只依赖 `horizon-mod-api`，不走 native route 去访问游戏内部，也不使用标注 "Not on wasm" 的 API。默认以 WASM 形式分发。原因：native mod 被 Horizon 标为实验性，锁定精确的游戏修订且不承诺跨修订兼容；玩家还要信任原生代码，注册表也要做源码安全审查。遇到 API 缺口时，向 horizon-rust/horizon 提 `mod API:` issue 并主动提交 PR，等待期间使用降级实现。

## Considered Options

- native 为主：性能和能力上限更高，但每个 Horizon 构建都要重新适配。
- native 并访问游戏内部：交付最快，但基本放弃版本兼容。

## Consequences

- WASM 每次普通调用有约 100 ms 的预算，且不能使用线程。网格、频道等计算必须增量进行，并分摊到多个 tick；一旦超时，整个 mod 在本次会话内停用。
- 只有在性能确实不够时（例如大型自动合成计算），才重新评估 native。重新评估需要写新的 ADR 来取代本 ADR。
