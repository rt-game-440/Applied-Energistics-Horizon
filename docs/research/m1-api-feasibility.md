# M1 功能在 horizon-mod-api（WASM）下的可行性调研

- 调研日期：2026-11-19
- Horizon 基线：`horizon-rust/horizon` main @ `988f2472b6`（2026-11-19 提交）
- AE2 基线：`AppliedEnergistics/Applied-Energistics-2` tag `v26.3.2-alpha`
- 约束：只用 `horizon-mod-api`（crate 名 `mod_api`），必须能构建为 `mod.wasm`；不用 native route（mixin、`[[internals]]`、`horizon-mod-render`），不用 rustdoc 标注 **Not on wasm** 的条目。
- 方法：只读。通过 `gh api` 读取 horizon 的 `docs/modding/src/**`、`docs/modding/examples/**`、`crates/horizon-mod-api/src/parts/*.rs`、`crates/horizon-mod-vocab/src/{hooks,host,parts}/*.rs`（hook 与 Host 方法的权威声明处）、`crates/horizon-mod-wasm/src/runtime.rs`、`crates/horizon-mod-runtime/src/lib.rs`、`docs/agent-control.md`、`docs/horizon-ctl-commands.md`，并对照 AE2 源码。
- 记号：**【事实】** 表示源码或文档里已核实的内容；**【推断】** 表示根据事实推出、尚未实测的结论。

## 0. 前置说明（影响所有条目）

1. **"Not on wasm" 的实际范围很小。** 全仓检索 `Not on wasm`，命中的只有以下几类：
   - `host/config.rs`：文件系统路径类；
   - `host/services.rs` / `host/tags.rs` / `world/types.rs`：frame resource 访问器，注释为"its methods are the imports"，也就是方法本身仍可经 import 调用；
   - `host/world_request.rs`：标题画面请求；
   - `parts/dimensions.rs`：自定义 ChunkGenerator / BiomeSource / DensityFunction；
   - `parts/entities.rs`：EntityPart 与 mob goal。

   **M1 用到的 API 都不在其中。**【事实】
2. **wasm 逐项可用性只做了间接核实。** 真正的 wasm 缺口表 `UNSUPPORTED_IMPORTS` / `UNSUPPORTED_EXPORTS` 由 `crates/horizon-mod-api-gen/src/wire/host.rs` 在构建时生成，覆盖报告由 `cargo run -p xtask -- mod-api coverage` 产出，仓库里都没有提交。【事实】因此本文的 wasm 可用性只能用以下间接证据支撑：
   - 文档明示"native and wasm"；
   - `crates/horizon-mod-wasm/wire-loopback/tests/` 有对应用例；
   - 示例已构建为 wasm。

   没有直接证据的条目都标注了"wasm 可用性未在生成缺口表中逐项核实"。**建议 M1 开工前在本地跑一次 `cargo run -p xtask -- mod-api coverage`（据 `xtask/tests/book_chapters.rs` 以 `mod-api coverage` 参数生成 "Wire boundary coverage" 页面；确切调用方式待本地确认），把结果补进本文。**
3. **文档之间存在过时与矛盾，本文以 hook 声明文件和可构建示例为准。**
   - `docs/design/mod-api/neoforge-extensions.md` 仍把 `onNeighborChange` 记为 Missing、把 `onLoad` 记为"no after-load notice"。但 `hooks/on_neighbor_change.rs`、`hooks/block_entity_on_load.rs` 和 `how-to/world/block-entity-neighbour-changes.md` 都已存在。
   - `docs/design/mod-api/host-helpers-and-remaining-gaps.md` 称 `screen_key_pressed_pre`、`screen_char_typed_pre`、`screen_mouse_scrolled_pre` "not in the trait"。但 `docs/modding/examples/src/widget_screen.rs` 实现了前两个，而且该示例已构建为 wasm（`examples/wasm-widgets/horizon-mod.toml` 带 `[wasm] sha512`）；`hooks/screen_mouse_scrolled_pre.rs` 也存在。
4. **WASM 运行时的硬约束**（详见第 9 项）：
   - 非加载类调用默认预算 100 ms；
   - 一旦超时或 trap，**整个依赖组在本次会话内停用**。

   这会左右所有每 tick 逻辑的设计。

---

## 1. 内容层（材料物品/方块、世界生成、配方与自定义配方类型、标签、语言、贴图与模型）

**结论：可行。** 只有一处需注意：**AE2 自定义配方的 serializer 必须全部注册，否则整个配方加载会被拒绝。**

**依据**

- **物品与方块**【事实】
  - 两条声明途径：数据文件 `data/<id>/horizon/block/*.json`、`data/<id>/horizon/item/*.json`（复制 vanilla template 再覆盖属性）；代码 `BlocksEvent` / `ItemsEvent` + `mod_api::Block::new(id, template)` / `Item::new`。
  - 两条途径对 native 与 wasm 相同。`how-to/content-mods.md` 原文："This is identical for native and wasm mods."
  - 可声明自有 `state_properties`（boolean / enum）、`shapes`、`collision_shapes`、`state_light`、`render_shape`、`occludes` 等。
- **世界生成**【事实】
  - 石英矿：支持 `data/<ns>/worldgen/feature/`、`placed_feature/`，以及 NeoForge 格式的 `data/<ns>/neoforge/biome_modifier/`（`neoforge:add_features`）。代码侧也可用 `Registrar::declare_biome_modifier`。
  - 陨石：可注册 `StructureType`（`how-to/world/structure-types.md`），并用 `PieceContext::set_block` / `block_entity` 放置方块和方块实体。原文："These interfaces also support structures in `mod.wasm`"。
  - 生成代码在 worker 线程上执行，必须无状态、确定性；worker 调用同样受 epoch 预算约束。AE2 对应实现是 `appeng/worldgen/meteorite/MeteoriteStructure.java` 与 `MeteoriteStructurePiece.java`。
- **自定义配方类型**【事实】，见 `crafting-recipe-kinds.md` 与 `crates/horizon-mod-vocab/src/parts/recipe_serializers.rs`：
  - 注册：`r.register_recipe_type("my_mod:grinding")` + `r.register_raw_recipe_serializer(...)`，或 `register_read_only_recipe_serializer(serializer_id, recipe_type_id, impl RecipeSerializer)`；
  - 读取：`host.recipes_of_type(kind)`；
  - 原文："It never joins the crafting grid or a built-in station. The machine owns its matching and menu."；
  - **"An unregistered serializer refuses the entire recipe load"**；
  - 自定义合成网格配方可用 `register_crafting_recipe_serializer` + `CraftingRecipeKind`，编码值上限 1 MiB；
  - 配方会同步到 Horizon 客户端，vanilla 客户端收不到自定义类型。
- **标签、语言、贴图、模型**【事实】
  - 标签：`data/<ns>/tags/`、`data/minecraft/tags/`、`data/c/tags/`。
  - 语言：`assets/<id>/lang/`。
  - 贴图与模型：标准资源包管线，blockstate 的 `variants` / `multipart` 可用。
  - 自定义 blockstate 定义类型：`register_block_state_models` + `instantiate_block_state_definition`。
  - 自定义模型 loader：`register_model_loaders` + `bake_model_loader`，在加载期生成 quad，每个模型上限 65,536 quad。
- **复用原 jar 资源**【事实】：`cargo horizon-mod use-jar <jar>` 可以把原 AE2 jar 当作外部 `assets/` / `data/` 层引用，不复制进 mod。但它提供的 data 里含有 `ae2:inscriber` 等配方，同样受"serializer 必须注册"的约束。【推断】此外还需要单独确认 AE2 美术资源的许可条款。

**降级 / 实施建议**

- AE2 在 `appeng/recipes/AERecipeSerializers.java`（v26.3.2-alpha）共注册 12 个 serializer【事实】：`inscriber`、`facade`、`entropy`、`matter_cannon`、`transform`、`charger`、`storage_cell_upgrade`、`add_item_upgrade`、`remove_item_upgrade`、`quartz_cutting`、`crafting_unit_transform`、`storage_cell_disassembly`（均为 `ae2:` 命名空间）。凡在 data 中出现的都必须注册（如 `register_read_only_recipe_serializer` 或 `register_crafting_recipe_serializer`）；M1 暂不实现的类型也要先注册，或者删掉对应的 json，或者用 load conditions 跳过。
- AE2 datagen 产出的 `src/generated/resources/**` 可以直接作为 `assets/` / `data/` 的起点。

**需上游 API：** 无。

---

## 2. 带方块实体的机器（ME 控制器、能源接收器、ME 驱动器）

**结论：部分可行，需降级。** 控制器和能源接收器可行；驱动器的"外观随元件变化"需要改用方块实体渲染（`block_entity_submit`）实现。

**依据**

- **方块实体类型**【事实】，见 `concepts/parts/block-entity-types.md`：
  - 声明：`r.block_entity_type("drive").blocks(&[...]).build()`，再 `r.block_entity_part(&key, Part)`。
  - 生命周期 hook：
    - `block_entity`：创建时给出初始 NBT；
    - `block_entity_on_load`：每个实例加载后通知，两侧都会触发；
    - `block_entity_tick` / `block_entity_ticks`：服务端 tick；
    - `on_chunk_unloaded`：区块卸载；
    - `pre_remove_side_effects`：移除时掉落额外物品；
    - `block_entity_update_tag` / `block_entity_data_received`：服务端→客户端同步；
    - `block_entity_client_tick`：客户端 tick。
  - 持久状态：`BlockEntityPartState` 或 `merge_data` / `replace_data`。
  - wasm 限制：
    - "It also refuses a declaration without a codec from a mod the game loaded as wasm"；
    - "Part state is server-side for now. A client reads the default"。
- **控制器多方块校验**【事实】
  - 读邻居：`level_block_state` / `level_block` / `level_block_property`（`host/levels.rs`）。
  - 改状态：`level_set_block`。
  - 事件：`neighbor_changed`（服务端 `Block.neighborChanged`）、`on_block_state_change`（两侧）、`on_neighbor_change`（相邻方块实体变化）。
  - 控制器外观的 online / offline / conflicted 以及 column / inside 变体都可以声明为 `state_properties`，再由 blockstate 选模型。AE2 生成的 `assets/ae2/blockstates/controller.json` 与 `controller_block_*` / `controller_column_*` 模型可以沿用。
  - AE2 的校验逻辑在 `appeng/me/pathfinding/ControllerValidator.java`，属纯算法，可移植。
- **能源接收器**【事实】：`r.block_energy(&block, ModEnergyBuffer)` 或 `r.block_entity_capability(&ENERGY, &kind, provider)`，详见第 7 项。
- **ME 驱动器的 10 个元件槽**【事实】
  - 槽位：`r.block_container(&block, container)`（`ModContainer`）。
  - 槽位读写：`BlockEntityRef::item_slot` / `set_item_slot(host, tx, slot, stack)`，绕过自动化策略（`how-to/world/machine-menu-data.md`）。
  - 外部访问：`ITEMS` capability。
- **驱动器外观**【事实】
  - 不支持模型随方块实体数据变化：`neoforge-extensions.md` 原文："`getModelData(state, level, pos)` | Missing | — | No `ModelData`; block-entity drawing takes a `block_entity_submit` part with its own model path"，`requestModelDataUpdate()` 同为 Missing。hook 列表中也没有任何 model-data 相关 hook。
  - 可行替代：`block_entity_submit`（`hooks/block_entity_submit.rs`）每帧在视距内绘制。可用 `draw.block(...)`、`draw.item(...)`、`draw.model(ModelRef)`、`draw.terrain_model(...)`、`cube` / `quad`，数据来自 `block_entity_update_tag` 同步过来的 NBT。
  - AE2 原实现：`appeng/client/render/model/DriveModel.java` + `appeng/block/storage/DriveModelData.java`，依赖 ModelData。

**降级 / 实施建议**

- 驱动器机身用静态模型，10 个元件槽（含 LED 状态）由 `block_entity_submit` 绘制。元件模型用 `standalone_models` 注册后，经 `RenderAssets::model` 解析为 `ModelRef` 再绘制。
- 服务端在 `block_entity_update_tag` 里放"每槽元件类型 + 状态"的紧凑数据。状态变化时调用 `host.block_entity_sync(pos)` 或 `send_block_updated`。
- 控制器、驱动器的运行状态（online 等）用 block state，不依赖方块实体渲染。

**需上游 API（可选，非阻塞）：** `mod API: block model data from block entity (ModelData / requestModelDataUpdate equivalent)`

---

## 3. ME 线缆：动态外形、碰撞箱、线缆总线部件

**结论：部分可行，需较大降级。** 线缆本体的连接外形和碰撞箱可以用 block state 实现，不需要方块实体。"任意部件组合 + 由方块实体数据决定的形状和模型"做不到，需要上游补 API。

**依据**

- **形状只能由 state 决定**【事实】，见 `how-to/content-mods.md`：
  - `shapes`、`collision_shapes`、`interaction_shapes` 按 state property 元组查表，每个形状最多 16 个 box；
  - 可按实体类别覆盖（`outline_context_shapes` / `collision_context_shapes`）；
  - 原文："These declarations select state/class-only geometry, not arbitrary world-dependent block behavior or native callbacks."；
  - `neoforge-extensions.md` 中 `collisionExtendsVertically` 为 Missing，注明 "Collision shapes are data"；
  - hook 列表中没有 `getShape` / `getCollisionShape` 类回调。
  - 而 AE2 `CableBusBlock.getShape` / `getCollisionShape` 是从 `CableBusBlockEntity.getCableBus().getShape()` 取的（`appeng/block/networking/CableBusBlock.java` 第 264–279 行）。
- **内置的六向连接方块**【事实】：模板 `minecraft:chorus_plant` + `"connects_to": "#mymod:cable_connects"` + `"thickness"`（2–14 像素）。原文："each side is joined when the block there is the same block or in the tag, on placement and whenever that neighbour changes"，外形按 PipeBlock 规则生成（中心 + 各臂）。
  - 但 `hooks/connects_to.rs` 只覆盖 Fence / Wall / Bars 三种 `ConnectionKind`，**不能用代码改写 pipe 的连接规则**。
- **自定义 state 方案**【事实】
  - 方块可以声明自有 enum / boolean 属性。
  - 服务端在 `neighbor_changed`（以及 `state_for_placement`）里自行计算连接，再用 `level_set_block` 改写 state。
  - blockstate `multipart` 按属性拼装模型。
  - 注意：`content-mods.md` 里"Every combination has to fit in the states a world holds (2^20)"指的是**全局** state ID 空间【推断】，所有线缆变体共享这一预算。
- **动态模型**【事实】：没有 ModelData 或按位置变化的 baked model（见第 2 项）。可用的动态绘制途径有三条：
  - `block_entity_submit`：每帧，需要方块实体；
  - `add_section_geometry`：`SectionGeometry`，在区块网格化时追加 quad。其上下文 `level()` 只能读区块和 8 个邻区块的 `BlockAndTintGetter`，读不到方块实体数据；它运行在网格化 worker 上，文档要求"keeps no mutable state and gets no Host"；
  - `submit_custom_geometry`：每帧世界 quad / 实例，默认上限各 4,096，实例 16,384。
- **部件交互**【事实】：`use_item_on_block` / `right_click_block` 带命中点；`player.open_mod_menu(...)` 可以打开无方块的菜单（见第 6 项）。

**降级 / 实施建议**

- **线缆本体**
  - 每种线缆（玻璃、包层、致密、智能）× 每种颜色做成**独立方块**，避免颜色属性让 state 数量翻倍。
  - 每个方向一个 `enum` 属性，例如 `none | cable | part`，`shapes` 表按 6 个属性给出 core + arm（+ part 底座）。表可由构建脚本生成；3^6 = 729 个元组，每个形状 ≤ 13 个 box，不超过 16。
  - 连接规则（同色或 fluix 互连、致密与普通线缆的关系）在 `neighbor_changed` 里用代码判断后写 state，不使用 `chorus_plant` 的 tag 规则。如果想更省事，可以用 `chorus_plant` + 每色一个 tag 来近似。
- **线缆总线部件（M1 只有终端、合成终端）**
  - 方向属性里每个部件种类占一个值，即每方向 `none | cable | terminal | crafting_terminal`：4^6 = 4,096 个 state。部件模型由 multipart 选择。
  - 部件的物品与配置存进方块实体（只有带部件的线缆才用带方块实体的变体方块），显示状态（亮屏、暗屏）再加一个属性。
  - 仅 M1 本身：4,096 state × 5 种线缆 × 17 色 ≈ 34.8 万，若再为"带方块实体"的变体复制一套则约 69.6 万，已占全局 2^20（≈ 104.9 万，【推断】为全局预算）的 1/3～2/3。
  - 部件种类增加后，state 数量按 k^6 增长：k = 8 时为 262,144；再乘上线缆种类和颜色的方块数，会很快逼近全局 2^20【推断】。M2 之后的总线（导入、导出、存储总线）需要上游能力，或者改成"部件只用方块实体渲染，碰撞箱取近似"。
- **备选**：所有部件都由 `block_entity_submit` 绘制，碰撞箱固定为"完整核心 + 6 臂"的近似形状。代价是精确选中、碰撞做不到，而且每帧都有绘制开销。

**需上游 API**

- `mod API: BlockPart shape/collision callback driven by block entity data (getShape/getCollisionShape equivalent)`
- `mod API: per-position block model data for chunk meshing (ModelData / IDynamicBakedModel equivalent)`
- 可选：`mod API: connects_to rule for PipeBlock-style (chorus_plant template) blocks`

---

## 4. 网络图（grid）：跨方块构建、tick 更新、区块加载与卸载、频道分配

**结论：可行（逻辑完全在 mod 内实现）。** 前提是采用"集中式 tick + 增量计算"的架构，以适应 wasm 的预算约束。

**依据（可用的 hook，均来自 `crates/horizon-mod-vocab/src/hooks/*.rs`）**【事实】

| 需要 | hook / API | 侧 |
|---|---|---|
| 方块放置 | `ItemPart::on_block_placed_by`、`state_for_placement`、`block_entity`（创建）、`block_entity_on_load`（每个实例加载后，含新放置） | 服务端 / 两侧 |
| 方块破坏 | `player_destroyed_block`（带被破坏方块实体的 NBT）、`break_block`、`pre_remove_side_effects`、`BlockPart::remove_block`、`on_block_state_change` | 服务端 |
| 邻居变化 | `neighbor_changed`（vanilla `neighborChanged`）、`on_neighbor_change`（相邻方块实体变化） | 服务端 |
| tick | `server_tick`（Pre / Post；`host.server_has_time()`）、`level_tick_post`、`block_entity_tick`、`schedule_task` + `scheduled_task` | 服务端 |
| 区块加载 / 卸载 | `chunk_load`、`chunk_unload`、`BlockEntityPart::on_chunk_unloaded`（每个 mod 方块实体一次，带 `BlockEntityRef`） | 服务端 / 两侧 |
| 世界开关 | `world_lifecycle`、`level_load` / `level_unload` | — |
| 持久化 | `AttachmentType`（`Holder::Level` / `Chunk` / `BlockEntity`，Level 级附件存 `data/horizon/attachments.dat`） | 服务端 |
| 读写世界 | `host.level(dim)`、`level_block_state`、`level_set_block`、`host.block_entity(level, pos)` | — |

- `machine-menu-data.md` 原文就以"its place in an energy or pipe network"为例，说明可以在 `on_chunk_unloaded` 里释放网络节点。
- 关键约束【事实】：
  - `block_entity_tick` 每次调用都会把该方块实体的整份 NBT 快照**主动编码**传入 guest。原文："The input snapshot is still sent; this is not batching or a validity cache"。
  - tick 期间方块实体"is out of the world"，在该位置查询会查不到它。
  - 没有改动时应回 `BlockEntityTickReply::Unchanged`。
- 频道分配：AE2 的 `appeng/me/service/PathingService.java` 与 `pathfinding/*` 是纯算法，可以移植；Horizon 侧不需要额外 API。

**降级 / 实施建议**

- **不要**给每段线缆或每个节点注册 `block_entity_tick`。网格状态（节点表、邻接、频道、能量）放在 wasm 内存中的全局结构里，按维度索引；在一次 `server_tick`（Post）中驱动所有 grid 的 tick。
- 拓扑变化（放置、破坏、邻居变化、区块加载卸载）只做标脏；在下一次 `server_tick` 里**分批**重建，并用 `server_has_time()` 判断是否推迟到后续 tick。
- 世界关闭时（`world_lifecycle`）清空全局结构；需要跨重启保存的只有节点自身配置，存在方块实体里。网格在加载时重建，与 AE2 行为一致。

**需上游 API：** 无。可选：`mod API: batched block entity tick / opt-out of per-tick input snapshot`，用于性能优化。

---

## 5. 物品存储元件：物品上保存大量数据

**结论：可行。** 有一处待核实：wasm 上 `component_types` 的可用性。备选方案是 `minecraft:custom_data`，同样可行。

**依据**

- **注册组件类型**【事实】
  - `hooks/component_types.rs`："Register namespaced NBT component types before content loads"，`side = "both"`，`context = "registration"`，事件里往 `types` 推入 id 字符串（见 `horizon-mod-api-gen/dispatch-tests/src/fixtures/base.rs`）。
  - 该 hook 没有 Not on wasm 标注。但 `how-to/content-mods.md` 原文写的是"**Native** mods can register component types before content loads"，与 hook 声明不一致，**wasm 可用性未在生成缺口表中逐项核实**。
- **组件值**【事实】
  - 值是原始 NBT（`Tag`），会保存并同步。原文："Values must fit the network NBT limits and round-trip through saved NBT"。
  - 网络 NBT 读配额：`mod_api::NETWORK_NBT_QUOTA` = 2,097,152 字节（`reference/maths-and-network-nbt.md`）。
  - 只有 `side = "both"` 的 mod 才能声明会同步的内容，加入服务器时会比对 mod 列表。
- **读写**【事实】
  - 读：`ItemResource::component(&ComponentId)`。
  - 改：`resource.with_component(host, &id, Tag)` 后 `place.exchange(host, tx, &changed, 1)`；或者在事务中用 `ItemPlace::edit(host, tx, |stack| stack.set(...))`（`concepts/capabilities/item-places-and-components.md`）。
  - 原文："raw NBT components such as `custom_data` and mod components compare by NBT type and value"。
- **AE2 的需求**【事实】：`AEComponents.STORAGE_CELL_INV` 为 `List<GenericStack>`；`BasicCellInventory.MAX_ITEM_TYPES = 63`，每种数量为 long。按 NBT 估算，单个元件远小于 2 MiB【推断】。

**降级 / 实施建议**

- 首选 `ae2:storage_cell_inv` 自定义组件，存成 NBT list：每项 `{id, components, count: Long}`。
- 如果 wasm 不能注册组件类型，改用 `minecraft:custom_data` 下的 `ae2` 子键。行为等价，只是存档格式与原版 AE2 不兼容（反正跨加载器存档本来就不兼容）。
- 元件放在驱动器里时，服务端应把缓存放在 grid 内存中，周期性回写组件，不要每次变化都改写物品（AE2 也是延迟保存）。

**需上游 API：** 如果核实后 wasm 不可用，提 `mod API: component_types registration for wasm mods`。

---

## 6. ME 终端 / 合成终端界面

**结论：部分可行，需降级。** 自定义菜单、自绘屏幕、滚动网格、搜索框、按钮、点击插入或取出、服务端→客户端增量同步，都有对应 API。不存在现成的"虚拟槽位"控件，网格的绘制、命中和交互必须由 mod 自己实现。

**依据**

- **菜单**【事实】
  - 菜单类型：数据文件 `data/<id>/horizon/menu/<name>.json` 声明 `{ "slots": 1..256, "data_slots": 0..256, "results": [...] }`（`content-mods.md` 的 "Menu types" 一节）。
  - 方块打开：`r.block_container(&block, container)` + `player.open_menu(host, entity)`。
  - 无方块打开：`player.open_mod_menu(host, menu, title, extra, pos, replace)`。`extra` 最多 32 KiB，客户端用 `host.screen_extra_data()` 读取（`how-to/world/machine-menu-data.md`）。
  - 服务端点击回调：`menu_clicked`、`menu_button`（`screen_click` 把屏幕点击转成按钮 id）、`menu_quick_move`、`menu_slot_rules`、`menu_still_valid`、`menu_closed`。
  - 光标物品：`ItemPlace::PlayerCursor { menu }`（`crates/horizon-mod-vocab/src/block/item_place.rs`）可在事务内写入，因此"取出到光标"可行。
- **屏幕布局与绘制**【事实】
  - 布局：方块菜单通过 `screen` hook 返回 `Screen { width, height, slots: Vec<Option<[f32;2]>>, inventory_x/y, own_labels, own_background, ... }`（`crates/horizon-mod-vocab/src/client/types.rs`），可以隐藏槽位、自绘背景。
  - 绘制：`screen_render_post`（`event.draw` 为 GUI 绘制上下文，`event.screen.menu` 给出菜单类型 id）。`host/draw.rs` 提供：
    - `fill`、`sprite`、`gui_nine_slice`、`tiled_sprite`、`text`；
    - `item_stack`，以及 `gui_draw_item_with(stack, at, Some("1.2k"))` 自定义数量文字；
    - `gui_tooltip`；
    - `push_clip` / `pop_clip`（滚动区域裁剪）；
    - `widget_button`、`widget_text_field`。
  - 控件：`mod_api::client::widgets::{Widgets, TextField, Button}`，示例 `docs/modding/examples/src/widget_screen.rs`，已构建为 wasm（`examples/wasm-widgets`）。
- **输入**【事实】：`screen_mouse_button_pressed_pre` / `released_pre` / `dragged_pre`、`screen_mouse_scrolled_pre`（带 `scroll` 与 `modifiers`，可取消）、`screen_key_pressed_pre`、`screen_char_typed_pre`，均返回 `Flow::Cancel` 来吞掉事件。另有 `screen_room`（在容器右侧预留面板空间）。
- **同步**【事实】
  - 自定义 payload：`PayloadsRegistrar::add_payload` / `add_typed_payload`（typed 用 `ToClient::send` / `ToServer::send`），或 `PlayerRef::send_payload`（服务端→指定玩家）与 `send_payload_to_server`（客户端）。
  - `host/payloads.rs` 原文："No generic registered-body or per-tick count cap applies"。
  - 单包上限受 `ClientboundCustomPayloadPacket.MAX_PAYLOAD_SIZE` 约束（`host/network.rs` 提到 `PayloadTooLarge`）。vanilla 的值是 1 MiB【推断，按 vanilla 常量】。
  - 有 wasm 用例：`horizon-client/src/game/modding/payload_transport/typed_wasm_tests.rs`、`wasm_tests.rs`。
- **合成网格**【事实】
  - `LevelRef::recipe_for` 查询 `minecraft:crafting`。
  - 9 个网格槽 + 1 个结果槽可以作为菜单的真实槽位（`results` 声明结果槽只能取出）。
  - 配方书：`menu_recipe_book` + `recipe_sources`（可借出相邻容器）。

**降级 / 实施建议**

- 终端菜单至少声明 1 个真实槽位，合成终端声明 10 个（9 网格 + 1 结果）；ME 存储网格**全部虚拟化**：
  - 客户端：在 `screen_render_post` 中自绘网格，`push_clip` 裁剪，滚动条来自 `screen_mouse_scrolled_pre`；搜索框和排序按钮用 `widgets`；`gui_draw_item_with` 绘制 AE 风格数量；`gui_tooltip` 显示悬停信息。
  - 点击：客户端截获鼠标事件后，发 typed payload `TerminalAction { serial, action, ... }` 给服务端；服务端在事务中从网格取出物品，写入 `PlayerCursor` 或玩家背包，并把插入请求写回网格。
  - 同步：服务端 grid tick 计算差量，用 `MEInventoryUpdate` 风格的 typed payload 分片发送（每片远小于 1 MiB），对应 AE2 的 `core/network/clientbound/MEInventoryUpdatePacket.java`。
- 终端作为线缆部件（无独立方块实体位置）时，用 `open_mod_menu(..., pos: Some(cable_pos), ...)` 打开。【推断】此时屏幕布局可能落入 `screen` hook 的 "block" 作用域之外，需要实测 `open_mod_menu` 打开的屏幕的默认布局，以及能否用 `screen_render_post` 完整覆盖绘制。不行就让终端先作为独立方块（带方块实体）降级实现。
- **兜底方案**：`host.open_menu(uuid, rows≤6, title, items, replace)` 的箱子菜单 + `menu_set_items` 分页刷新，可以做成"无自绘"的最小可用终端。

**需上游 API（可选）**

- `mod API: virtual/ghost slot widgets for mod screens (item grid with hit-testing)`
- `mod API: screen layout hook for menus opened via open_mod_menu`（需先验证）

---

## 7. 能量：AE 能量与外部 FE 互通

**结论：可行（需处理数值范围）。**

**依据**【事实】，见 `concepts/capabilities/block-energy.md`、`crates/horizon-mod-api/src/parts/energy.rs`、`crates/horizon-mod-vocab/src/host/fluids_energy.rs`：

- `ENERGY` 对应 NeoForge `Capabilities.Energy.BLOCK`。
- `EnergyHandler::{amount, capacity, insert(host, tx, u32), extract(host, tx, u32)}`，**数量类型为 `u32`**。
- 提供方式：`r.block_energy(&block, ModEnergyBuffer)`，或 `r.block_entity_capability(&ENERGY, &kind, provider)` 实现自定义 handler。
- 转移：`move_energy(host, &from, &to, max, tx)`。
- 物品能量：`ITEM_ENERGY`、`ItemComponentEnergy`。
- 自带 `LimitingEnergyHandler` 等包装器。

AE2 的 AE 单位是 `double`，与 FE 的换算为 1 AE = 2 FE。

**降级 / 实施建议**

- 能源接收器实现自定义 `EnergyHandler`：`insert` 收到的 FE 换算成 AE 后注入 grid 能量池（grid 内部用 `f64`）；`capacity` / `amount` 夹到 `u32::MAX`。
- handler 写入的状态必须进事务日志（方块实体 NBT 或附件），否则事务 abort 时会丢失或重复能量。另一种做法是让 handler 只操作方块实体内的缓冲区，由 grid tick 再从缓冲区抽取。

**需上游 API：** 无。

---

## 8. 物品在方块间移动（capability + 事务）

**结论：可行。**

**依据**【事实】，见 `concepts/capabilities.md`、`concepts/transactions.md`：

- `ITEMS` 为 `ResourceHandler<ItemResource>`，方法有 `size`、`resource`、`amount`、`capacity`、`is_valid`、`insert` / `extract(host, tx, ...)`。
- 查询：`level.capability(host, &ITEMS, pos, side)`。
- 转移：`move_resources(host, &from, &to, filter, max, tx)`。
- 事务：`host.transaction(|host, tx| ...)` 支持嵌套和 abort 回滚。没有 simulate 参数，模拟靠 abort 实现。
- 日志覆盖：方块实体物品、NBT、附件，玩家背包与光标等。
- 自定义 capability：`CapabilityKey::new("ae2:...")`。
- 文档建议每 tick 只查一次邻居 capability 并复用，以减少 wasm 跨界次数。

**实施建议**

- ME 接口、存储总线（M2）对外暴露 `ITEMS` handler，内部直接读写 grid 存储。
- 读写 grid 的副作用要能被事务回滚：做法是把"本 tick 的变更"记在 grid 的待提交队列里，靠事务的 commit / abort 决定是否生效。【推断】API 中没有看到 mod 自有内存状态的 commit 回调，需要设计成"只在外层事务提交后才写 grid"，或者把 grid 存储的权威副本放进方块实体 NBT / 附件，让游戏的事务日志覆盖它。这一点需要在 M2 前专项验证。

**需上游 API（可选）：** `mod API: transaction commit/abort callbacks for mod-owned in-memory state`

---

## 9. WASM 限制与对大型网络的影响

**结论：部分可行需降级。** 大型网络的每 tick 计算必须**增量化**、**分摊**，并控制跨界调用次数；否则一旦超过 100 ms，整个 mod 会在本次会话被停用。

**具体数值**【事实】

| 项 | 值 | 出处 |
|---|---|---|
| 普通（非加载）调用预算 | **100 epoch tick ≈ 100 ms** / 顶层调用（`CALL_BUDGET` 为 0 时取 100；同线程重入共享外层剩余时间） | `crates/horizon-mod-wasm/src/runtime.rs` 第 1211–1221 行、第 322–324 行 |
| 加载类调用预算 | **30,000 ms**（`MAX_GUEST_CALL_BUDGET_MILLIS`） | `crates/horizon-mod-runtime/src/lib.rs` 第 5 行 |
| epoch 时钟 | 后台线程每 1 ms `increment_epoch`（墙钟） | `runtime.rs` 第 195–200 行 |
| 等待组锁 | 普通调用最多 5 s；嵌套调用遇到借出的 store 立即拒绝 | `how-to/wasm-mods.md`，`runtime.rs` 第 1174 行 |
| 嵌套深度 | 64 | `runtime.rs` "nested past depth 64" |
| 线性内存 | 默认 1 GiB（`DEFAULT_GROUP_MEMORY_MIB = 1024`）；manifest 中 `wasm_memory_mib` 可声明 1–4096 MiB；玩家侧 `wasm.memory_cap_mib` 默认 4096 | `runtime.rs` 第 45 行，`wasm-mods.md` 的 Memory 一节 |
| 函数表 | 2^20 元素 | `runtime.rs` 第 47 行 |
| 栈 | guest shadow stack 256 KiB（链接器）；Wasmtime 调用栈 128 MiB | `concepts/wire-types.md`，`wasm-mods.md` 的 Guest stack 一节 |
| 模块大小 | ≤ 64 MiB | `content-mods.md` |
| 线程 | 禁用 threads、shared memory、atomics | `wasm-mods.md` 的 Threads 一节 |
| Wire 嵌套 | 结构深度 32；NBT / Text 512 | `concepts/wire-types.md` |
| 故障后果 | "A trap or limit hit disables the affected group"；`runtime.rs`："wasm mod {} stopped its group" | `wasm-mods.md` |

**对大型网络的影响**

- 【推断】epoch 按墙钟计时，guest 内调用 host import 所花的时间很可能也计入 100 ms，所以跨界调用次数直接消耗预算。
- 【推断】AE2 的频道分配（BFS / 生成树）在数千节点规模下单次通常远低于 100 ms。但"全网重建 + 大量 `level_block_state` 查询 + 大量 payload 编码"挤在同一次调用里就可能超时。
- 对策：
  - grid 拓扑只在 wasm 内存里维护，尽量不在 tick 中回查世界；
  - 重建分批进行，用 `server_has_time()` 和 tick 内计时把单次工作控制在约 20–30 ms；
  - 每个 grid 的 tick 拆成多个阶段，分布到不同 `server_tick` 调用；
  - 终端同步只发差量并分片；
  - 避免对每个节点使用 `block_entity_tick`（参见第 4 项）。
- 内存：默认 1 GiB 足够；为大型存储预留时可声明 `wasm_memory_mib`。分配失败时可用 `mod_api::wire::guest::register_eviction` 回收缓存。

**需上游 API（可选）：** `mod API: configurable per-mod wasm call budget / tick-phase budget`。100 ms 写死在运行时，manifest 里没有可调项。

---

## 10. 测试：单元测试、fake host、游戏内测试与 agent 自动化验收

**结论：可行，且自动化程度高。** 有几点限制：fake host 不覆盖客户端，game test 不支持多 tick 序列，截图需要 `--offscreen`。

**依据**【事实】

- **单元测试**（`concepts/testing.md`）
  - 在 `horizon-mod-api` 的 dev 依赖上开 `testing` feature。
  - `testing::with_mods` 安装 mod，用 `dispatch_<hook>(&mut host, ...)` 按游戏方式派发事件。
  - `testing::FakeHost` 提供 registries、players、level（方块实体物品、流体、能量）、journal（事务回滚）。
  - "The fake has no client backend, so client handle writes refuse"；遇到不支持的方法会以方法名 panic。
  - 测试在 native 目标上跑，与 wasm 构建同源。
- **游戏内测试**（`reference/process-environment.md`、`hooks/register_game_tests.rs`）
  - 开启方式：`--game-tests` 或 `HORIZON_GAME_TESTS=1`。
  - 注册：`event.functions.register("ae2:check", Arc<dyn GameTestFunction>)` / `register_test(id, GameTestDefinition::new(structure, function))`。
  - 执行：`/test` 始终可用。回调拿到 `ServerHost` + `GameTestContext { level, absolute_pos }`，返回 `Pass` / `Fail(msg)` / `Pending`。
  - 限制原文："This adds no sequence engine"；函数在每次尝试开始时只运行一次（`GameTestDefinition` 有 setup delay、timeout、attempts 字段）。
  - vocab 有 `exports/game_tests.rs` 绑定。【推断】wasm 可用，但未在缺口表中逐项核实。
- **agent 控制**（`docs/agent-control.md`、`docs/horizon-ctl-commands.md`）
  - `horizon-ctl mcp`（或 `horizon-launcher mcp` 的 `game_*`）可以：
    - 启动：`launch --world --creative --offscreen|--headless`；
    - 等待：`wait_ready`、`wait_for screen|event|chat`；
    - 世界操作：`setblock`、`give`、`place`、`interact`、`break`、`command "/test runall"`；
    - 读取状态：`block`、`blocks`、`inventory`、`screen`（含槽位）、`ui`、`menu`、`events`、`logs`、`screenshot`；
    - 操作 GUI：`click_slot`、`click_at`、`drag_at`、`scroll_at`、`type`；
    - 节拍控制：`--paced`。
  - `--headless` 不能截图，需要截图时用 `--offscreen`。

**实施建议**

- 三层验收：
  1. **纯逻辑**（grid、频道、元件存储、配方匹配）：用 cargo 单元测试 + `FakeHost`。
  2. **服务端集成**（放置后 grid 上线、能量流入、物品进出）：用 `register_game_tests` 注册测试，结合 setup delay，或在 `scheduled_task` 里跑断言后写结果；用 `horizon-ctl command "/test ..."` 加 `logs` 判定。
  3. **客户端界面**（终端网格、搜索、点击取出）：用 `horizon-ctl --offscreen` + `click_at` / `scroll_at` / `type` + `inventory` / `screenshot` 断言，并让 mod 在 `logs` 目录写结构化调试日志，辅助判定。
- 多 tick 序列断言：先用 setup delay 或多个 test，后续视需要向上游提议序列引擎。

**需上游 API（可选）：** `mod API: multi-tick game test sequences (GameTestSequence / succeedWhen equivalent)`

---

## 附：需上游补齐的 API 清单（汇总）

| 优先级 | 建议 issue 标题 | 关联 M1 项 | 不补时的降级 |
|---|---|---|---|
| 高 | `mod API: BlockPart shape/collision callback driven by block entity data (getShape/getCollisionShape equivalent)` | 3 | 部件存在性编码进 block state，碰撞取近似；部件种类受 state 预算限制 |
| 高 | `mod API: per-position block model data for chunk meshing (ModelData / IDynamicBakedModel equivalent)` | 2、3 | multipart + state 属性；驱动器、部件改用 `block_entity_submit` 每帧绘制 |
| 待核实 | `mod API: component_types registration for wasm mods`（仅在核实 wasm 不可用时提） | 5 | 用 `minecraft:custom_data` 存元件内容 |
| 中 | `mod API: virtual/ghost slot widgets for mod screens (item grid with hit-testing)` | 6 | 在 `screen_render_post` 中自绘网格并自行命中检测 |
| 中（需先实测） | `mod API: screen layout hook for menus opened via open_mod_menu` | 6 | 终端先做成独立方块实体方块，或用箱子菜单分页 |
| 中 | `mod API: transaction commit/abort callbacks for mod-owned in-memory state` | 8（M2 前） | grid 存储的权威副本放进方块实体 NBT / 附件，由游戏日志覆盖 |
| 低 | `mod API: connects_to rule for PipeBlock-style (chorus_plant template) blocks` | 3 | 自声明 state 属性，在 `neighbor_changed` 中自行计算连接 |
| 低 | `mod API: batched block entity tick / opt-out of per-tick input snapshot` | 4、9 | 集中在 `server_tick` 中驱动 grid |
| 低 | `mod API: configurable per-mod wasm call budget / tick-phase budget` | 9（部分可行需降级） | 增量化、分摊计算 |
| 低 | `mod API: multi-tick game test sequences (GameTestSequence / succeedWhen equivalent)` | 10 | setup delay + 外部 `horizon-ctl` 断言 |

另：建议在开工前于本地运行 `cargo run -p xtask -- mod-api coverage`（确切调用方式待本地确认），拿到生成的 wasm 缺口表（`UNSUPPORTED_IMPORTS` / `UNSUPPORTED_EXPORTS`），逐项确认以下条目，并把结论回填本文：

- `component_types`
- `register_game_tests` / `GameTestFunction`
- `StructureType`
- `add_section_geometry`
- `block_entity_submit`
- `register_model_loaders` / `bake_model_loader`
- `RecipeSerializer`
- 各 capability provider

## 附：本地核实（2026-11-19，horizon main @2613d3b5）

在本地运行 `cargo run -p xtask -- mod-api coverage`。输出表只列出 wasm 可用的条目（已排除 "Not on wasm" 和未绑定的条目），共 2224 条，状态都是 `crosses`。

- `Hooks::component_types` 出现在表中，说明 **wasm mod 可以注册自定义数据组件**。第 5 项的待核实点就此解除，不需要再向上游提 `component_types registration for wasm mods`。
- 以下 M1 用到的 API 也都在表中：`block_entity_submit`、`open_mod_menu`、`server_tick`、`screen_render_post`、`neighbor_changed`、`register_game_tests`、recipe serializer、energy 相关条目。
- biome modifier 和结构没有以 Host 或 Hooks 方法的形式出现在表中。它们可能通过 `register` 阶段的 Registrar 声明，这一点在世界生成 issue 中实测确认。
