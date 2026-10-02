# $SPIN —— Solana 链上三列老虎机 · 技术实现文档（TDD）

> 版本：v1.0
> 配套文档：《Solana老虎机-游戏设计文档.md》（玩法、赔率表、双层代币模型）
> **计价：原生 SOL（lamports）**。$SPIN 仅作权益层，不进游戏金库。

---

## 0. 技术栈总览

| 层 | 技术 | 说明 |
| --- | --- | --- |
| **前端** | **Cocos Creator 3.x（TypeScript）** | 卷轴动画、粒子、音效、UI；发布 Web / iOS / Android |
| **后端** | **Go 1.23+** | 交易工厂、crank 代付结算、indexer、风控、统计、WS 推送 |
| **链上** | Rust + Anchor 0.30+ | 唯一真值：资金托管、随机数消费、符号推导、赔付结算 |
| 随机源 | Switchboard Randomness（VRF On-Demand） | 由 Go 侧协调创建 randomness 账户 |
| 通信 | HTTP REST + WebSocket | Cocos ⇄ Go；Go ⇄ Solana 用 `solana-foundation/solana-go` |
| 存储 | Postgres + Redis | 流水/明细/统计；限频与会话 |
| 资金单位 | **lamports（原生 SOL）** | 不使用 SPL Token 包装，见 §3.1 |
| 备选架构 | 中央账本（§11.1）／Passkey 智能钱包（§11.2） | 见 §11 |
| **多游戏** | **平台化：一个大厅 N 款 slot**，共享同一个 SOL 余额与同一套资金/随机/结算生命周期 | 见 §3.2–§3.7、§5.4、§6.1.1、§13 |

> **一句话原则**：链上是真值，Go 是协调，Cocos 是表现。后端不托管资金、不决定结果，Cocos 不做任何赔付计算。

---

## 1. 三层架构与职责边界

```
┌─────────────────────────────────────────────────────────┐
│ Cocos Creator 3.x (TypeScript)                          │
│  场景/动画/音效/HUD；NetBridge(HTTP+WS) + WalletBridge   │
│  不含钱包 SDK 重依赖、不含赔付逻辑                       │
└───────────────┬─────────────────────────────────────────┘
                │ REST (签名请求)  +  WebSocket (结果推送)
┌───────────────▼─────────────────────────────────────────┐
│ Go 后端（模块化单体，单二进制多 cmd）                     │
│  api │ txfactory │ crank │ indexer │ risk │ stat │ admin│
│  · 构造未签名交易  · 代付 settle  · 监控/对账/风控        │
└───────────────┬─────────────────────────────────────────┘
                │ JSON-RPC / WS   solana-foundation/solana-go
┌───────────────▼─────────────────────────────────────────┐
│ Solana 链上 · Anchor Program: spin_slot（平台层）        │
│  平台层：PlatformConfig │ PlayerState │ Round │ Vaults  │
│  游戏层：GameConfig(game_id) │ GameAssets(game_id, ver)  │
│  ─ 资金 · 随机 · 结算生命周期  · evaluate 唯一分派点     │
└───────────────┬─────────────────────────────────────────┘
                │ CPI
        ┌───────▼─────────────────┐
        │ Switchboard Randomness  │
        └─────────────────────────┘
```

**职责边界（写进 code review 清单）**

| 能力 | Cocos | Go 后端 | 链上 |
| --- | ---: | ---: | ---: |
| 卷轴停止位置、符号、赔付金额 | 只读展示 | 转发/预测 | **唯一决定** |
| 玩家资金 custody | ✗ | ✗ | ✔ |
| 构造交易、发交易 | ✗（仅唤起签名） | ✔ | — |
| settle 发起（crank 代付） | ✗ | ✔ | permissionless（任何人可替它做） |
| 限注/限频 | ✗ | 软限制（体验层） | **硬限制**（合约级 `require!`） |
| 代币分配（回购/分红） | 只读展示 | 计算与执行 | 销毁/分发在链上执行 |

> **非托管承诺**：即使 Go 后端全站宕机，玩家仍可凭链上 `PlayerState` 用任意 RPC 构造 `withdraw` 取回 SOL。必须提供 fallback 页面 / CLI 并写进文档。

---

## 2. Go ↔ Solana 适配性评估

**结论：适配度 ★★★★☆（4/5）。适合做后端/工具链，但 Go 不能写 Solana 链上程序。**

| 维度 | 评价 |
| --- | --- |
| SDK 现状 | `solana-foundation/solana-go` —— **官方组织下**已有 Go SDK：v1 线延续 `gagliardetto/solana-go`（v1.24.0），另有重构后的 v2 API |
| 能做什么 | JSON-RPC / WebSocket 客户端、交易构造与 ed25519 签名、PDA 派生、版本化交易 v0 + ALT、优先级费用 / ComputeBudget、durable nonce、SPL Token 与 ATA 辅助方法 |
| **不能做什么** | **写链上合约**。Solana 运行时（SBF）只支持 Rust / C。**合约必须 Rust + Anchor**，没有商量余地 |
| 主要缺口 | ① 无官方 Anchor 客户端：discriminator、IDL 编解码、错误码要自己生成；② Token-2022 扩展覆盖不全（本项目权益层用标准 SPL，影响较小）；③ 新特性跟进晚于 TS 生态；④ 资料少于 TypeScript |
| 相对优势 | goroutine + channel 天然适合本项目最重的负载 —— **同时订阅成千上万个 Round 账户、并发发起 settle**；单二进制部署、静态类型 |

**降低风险的四条做法**

1. **IDL → Go 代码生成**：Anchor 产出的 `slot.json` 写个小 codegen 脚本，生成 `internal/gen/anchor/`：指令编码器、账户结构体、8 字节 discriminator、错误码映射。一次性成本 1–2 天。
2. **共享常量单一来源**：`shared/paytable.json` + strip 数据被三方引用 —— Rust 侧构建时内联、Go 侧对账、前端展示。
3. **测试用真链，不用 mock**：CI 里起 `solana-test-validator`，Go 集成测试直接连本地集群跑通全流程。
4. **尽早 spike**：M1 开始前先花 2 天，用 Go 对一个最简 Anchor 程序跑通"构造指令 → 发交易 → 解析账户"，把序列化方案一次验证掉。

---

## 3. 链上合约设计

### 3.1 为什么用原生 lamports 而不是 SPL/WSOL

| 方案 | 合约复杂度 | CU | 说明 |
| --- | --- | --- | --- |
| **原生 lamports + PDA 持有** | 最低 | 最低 | deposit 用 SystemProgram.transfer；withdraw 在指令内**直接增减账户 lamports**，无需 CPI |
| SPL Token | 中 | 中 | 需要 mint / ATA / 多次 CPI，还要处理 Token-2022 兼容 |
| WSOL | 高 | 高 | 多一层 wrap/unwrap，无额外收益 |

采用原生 lamports。**关键实现细节**：

```rust
// withdraw：程序拥有 vault PDA，可直接搬运 lamports，无需 SystemProgram CPI
let vault = ctx.accounts.player_vault.to_account_info();
let user  = ctx.accounts.user.to_account_info();
**vault.try_borrow_mut_lamports()? -= amount;
**user.try_borrow_mut_lamports()?  += amount;
```

> ⚠ **Rent 坑**：vault PDA 必须保留 **rent-exempt minimum**，不能扣干。链上要维护 reservation 常量，可动用余额 = `vault.lamports() - rent_exempt_minimum`。所有金额结算都必须扣除这部分，否则会出现"账上有钱但提不出来"。

### 3.2 多游戏拓扑：三种组织方式

平台要支持"一个大厅、多款 slot"，第一件事是决定**多个游戏的代码放在哪**。

| 拓扑 | 做法 | 优点 | 缺点 | 适用 |
| --- | --- | --- | --- | --- |
| **A. 单程序 + `game_id` 分派（MVP 采用）** | 一个 `spin_slot` 程序，内部按 `game_id` 路由到不同的 evaluate 实现 | 共享 PlayerState / 金库 / 权限；实现最简单；一次部署全站生效 | 程序会变大；单个游戏改动需整体升级；新游戏要重新走一次全程序审计 | **MVP，前 3–5 个游戏** |
| **B. 平台程序 + per-game 程序（演进目标）** | 平台程序独占资金/随机/生命周期；游戏程序通过 CPI 调 `lock_bet` / `pay_out` | 完全风险隔离；新游戏独立部署与独立审计；可开放第三方开发 | 多一层 CPI 与 CU；需要 `AuthorizedGame` 白名单信任模型 | **游戏数 > 5 或引入第三方开发者** |
| C. 多程序 + 共享 PDA 约定 | 各游戏程序直接读写约定的 PDA | 无 CPI 开销 | 等于把金库写权限开放给多个程序，**任一游戏有漏洞即全站失守** | **禁止** |

**演进路径**：MVP 用 A，但**从第一行代码起就用 §3.3 的 `SlotGame` trait 划边界**。等到要切 B 时，只是把 trait impl 搬进新程序 + 包一层 CPI —— 平台层零改动。这是这套架构唯一值得提前投入的设计约束。

### 3.3 平台边界：GameModule 统一契约

**平台只负责三件事：资金、随机数、结算生命周期。平台永远不解释游戏结果。**

```rust
pub trait SlotGame {
    fn id(&self) -> GameId;            // "classic_fruits"…，定长 16 字节，用于 PDA seed
    fn top_multiplier(&self) -> u64;   // 单次最大倍数，用于动态 max_bet
    fn rng_u32_count(&self) -> usize;  // 一局需要几个 u32（3  reel=3，freespin 版可能=13）

    /// 纯函数：随机数 → 赔付与结果。禁止依赖本局之外的任何外部状态
    fn evaluate(&self, rng: &[u32], bet: u64) -> Result<Outcome>;
}

pub struct Outcome {
    pub payout: u64,
    pub result: Vec<u8>,   // 不透明 payload：平台只负责存取与 emit，不解释含义
}
```

三条设计红线：

1. **`evaluate` 必须是纯函数** —— 输入只有 `rng` + `bet` + 该游戏的只读配置。这带来三个免费好处：可纯单元测试、可做 Rust⇄Go 对拍、玩家可离线复算。
2. **`result: Vec<u8>` 是不透明的** —— 平台不需要知道"符号""payline""免费旋转"是什么，**加新游戏时平台代码零改动**。
3. **每个游戏自带一份 verifier** —— 既然平台不懂结果的含义，那么"让玩家复算"的能力就必须由游戏 bundle 自己提供（前端 §6.1.2）。

### 3.4 账户设计（平台层 / 游戏层）

**平台层（所有游戏共享）**

| 账户 | PDA Seeds | 关键字段 |
| --- | --- | --- |
| `PlatformConfig` | `["platform"]` | `authority`, `house_vault`, `treasury`, `paused`, `fees_bps`, `game_count` |
| `PlayerState` | `["player", user]` | `credit: u64`, `nonce: u64`, `wagered: u128`, `won: u128`, `last_settle_slot` |
| `PlayerVault` | `["vault", "player", user]` | 持有 lamports —— **跨游戏共享余额** |
| `HouseVault` | `["vault", "house"]` | 持有 lamports（全局准备金） |
| `Round` | **`["round", game_id, player, nonce]`** | `game_id`, `bet`, `request_slot`, `randomness`, `reveal_slot`, `payout`, `result: Vec<u8>`, `status` |

**游戏层（per `game_id`）**

| 账户 | PDA Seeds | 关键字段 |
| --- | --- | --- |
| `GameConfig` | `["game", game_id]` | `enabled`, `min_bet`, `max_bet_hard`, `max_exposure_bps`, `top_multiplier`, `assets_version`, `wagered`, `won` |
| `GameAssets` | `["assets", game_id, version]` | Strip（1000 字节 u8）/ Paytable / 该游戏专有参数 + `hash` + `version` |

> **为什么余额跨游戏共享**：玩家在大厅里切游戏的心理模型是"一个赌场账号"，切游戏不该要他再充值一次。代价是风险不能按金库隔离，必须靠 §3.5 的敞口配额来隔离。

> **为什么 Strip 存 1000 字节全表**：压缩的区间表能省 ~95% rent（约 0.007 SOL / 游戏），但会让外部验证者必须自己实现同样的生成算法。**存全表后任何人直接读链上这一个账户就能 1:1 复算**。每个游戏多花 0.007 SOL 换可验证性，在有多个游戏时依然很划算。

> **为什么 `Round` 的 seed 里要有 `game_id`**：否则同一玩家在不同游戏的 nonce 会撞车，且 indexer 无法按 round_pda 反查所属游戏。

### 3.5 金库与风险隔离

资金共享一个 `HouseVault`，但**风险按游戏分账**：

```
// 每个游戏的动态限注（链上强制）
game.max_bet = min(
    game.max_bet_hard,
    house_vault_available * game.max_exposure_bps / 10_000 / game.top_multiplier
);

// 全局并发敞口上限：所有 pending Round 的潜在最大赔付之和
Σ(pending_round.bet * game.top_multiplier) ≤ house_vault_available * global_open_exposure_bps / 10_000
```

三条隔离手段：

| 手段 | 作用 |
| --- | --- |
| **per-game `max_exposure_bps`** | 头奖极高的游戏（如 50,000×）自动获得更严的 max_bet，**不需要为每个游戏手工配额度** |
| **全局 `open_exposure`** | 防止多游戏同时停留在 pending 状态、多个头奖在极短时间内叠加刺穿共享金库 |
| **per-game `wagered/won`** | RTP 与累计统计按游戏分开，某个游戏数学异常可单独发现并下架 |

> ⚠ **联合风险必须重算**：多游戏并行时，**分散效应会让联合 σ 降低**（不同游戏的局结果相互独立），这对金库是有利的；但同时也带来了"多头奖叠加"的新风险，这正是 `open_exposure` 存在的原因。**每上线一个新游戏前，必须重算一次联合破产概率**，而不是沿用单个游戏的结论。

### 3.6 指令

**平台层（与具体游戏无关，上线后几乎不再变动）**

```
initialize_platform(...)                # 部署初始化
init_player()                           # 首次进入创建 PlayerState + PlayerVault
deposit(amount)                         # lamports → PlayerVault，credit += amount
withdraw(amount)                        # credit -= amount，vault lamports → 用户
request_spin(game_id, bet, randomness)  # 冻结 bet，创建 Round["round", game_id, ...]
settle_spin(game_id, round)             # permissionless：读 reveal → evaluate → 结算
expire_round(game_id, round)            # 超时未结算：bet 充公，Round 关闭
platform_pause / update / house_withdraw  # 管理端（Squads 多签）
```

**游戏层（每加一款游戏调一次）**

```
register_game(game_id, top_multiplier, risk_params)   # 注册并写入 GameConfig（多签）
upload_game_assets(game_id, strip, paytable)          # 写入 GameAssets，version += 1 + 更新 hash
set_game_enabled(game_id, bool)                       # 上架 / 下架
set_game_risk(game_id, min_bet, max_bet_hard, exposure_bps)
```

> **`set_game_enabled` 是大厅最重要的运维开关**：某款游戏的数学或动画出问题，可以立刻单独下架，其他游戏照常运行，无需全站 `paused`。这是"多游戏"相对"单游戏"最现实的运维收益。

### 3.7 结算核心逻辑（含游戏分派）

```rust
// settle_spin 内部 —— 平台层不知道游戏细节，只做：取随机数 → 分派 → 结算
let rnd_bytes = read_randomness(&ctx.accounts.randomness)?;   // 32 字节
require!(ctx.accounts.randomness.reveal_slot > round.request_slot, Err::NotRevealedYet);

// 平台只负责派生随机数流；要几个、怎么解释，是游戏的事
let game = load_game(&ctx.accounts.game_config, round.game_id)?;
require!(game.enabled, Err::GameDisabled);
let rng: Vec<u32> = derive_u32_stream(&rnd_bytes, round.request_slot, game.rng_u32_count());

// ▶ 唯一的游戏专属逻辑：一个纯函数
let Outcome { payout, result } = game.evaluate(&rng, round.bet)?;

// ⚠ 结算前二次校验敞口，不能只依赖 Request 时刻的判断
require!(payout <= house_vault_available(&ctx) * game.max_exposure_bps / 10_000, Err::Exposure);

player.credit = player.credit.checked_add(payout)?;
game.won = game.won.checked_add(payout)?;
round.payout = payout;
round.result = result;              // 原样存进 Round：平台不解释，indexer 解析、前端渲染
emit!(RoundSettled { game_id: round.game_id, round_pda, payout, result });
```

要点：

- **这是全平台唯一一个 `match game_id` 的地方**，之后加游戏只动 `evaluate` 的分派表。
- `result` 必须 `emit` 出去（in-event），因为平台不解析它，前端/ indexer 只能靠事件拿到。
- 所有金额用 `checked_add / checked_mul / checked_sub`。
- `GameAssets` 可更新，但必须 `version += 1` 并广播；Go 侧按 version 重新加载并重新验证 hash。

### 3.8 CU 预算（每款游戏都要单独 profile）

| 指令 | 估算 | 备注 |
| --- | --- | --- |
| `request_spin` | 20–40k | Switchboard commit CPI + Round 创建 |
| `settle_spin`（Game #1 经典三列） | 40–90k | randomness 读取 + strip 查表 |
| `settle_spin`（含免费旋转的复杂游戏） | 150–400k | **必须实测**；接近上限时需拆分多帧结算或提高 compute unit limit |
| `withdraw` | 10–20k | 无 CPI |
| 单指令上限 | 1.4M | 默认 compute budget 为 200k / 指令，超了要显式申请 |
| 单交易上限 | 1.4M | **多游戏共享账户不冲突，但同一笔交易也只能 1.4M** |

---

## 4. 随机源

| 方案 | 安全性 | 成本/局 | 延迟 | 结论 |
| --- | --- | --- | --- | --- |
| **Switchboard Randomness (VRF, On-Demand)** | 高（密码学可验证，抗验证者操纵） | ~0.001–0.002 SOL（创建 randomness 账户） | 1–3 slot | **推荐** |
| ORAO VRF | 高 | 类似 | 1–2 slot | 备选 |
| SlotHashes commit-reveal | 中（验证者可小额操纵，仅适合小额） | 0 | 1 slot | 仅作小额快速档备选 |
| 区块哈希 / 时间戳 | 低 | 0 | 0 | **禁止使用** |

> Solana 无原生 VRF precompile，必须依赖外部随机源或 commit-reveal。
> 若走 §11.1 的账本架构，随机性改用 Provably Fair（commit-reveal + 批量锚定），详见 §11.1.5。

---

## 5. Go 后端

### 5.1 模块设计

| 模块 | 职责 | 实现要点 |
| --- | --- | --- |
| `api` | REST + WS 网关，SIWS（Sign-In With Solana）签发 JWT | gin/echo；WS 用 gorilla 或 nhooyr；连接即订阅该钱包事件流 |
| `txfactory` | 组装**未签名交易**返回 base64：ComputeBudget 提价 + 创建 randomness 账户 + `request_spin` | 返回体带 `lastValidBlockHeight` 与过期时间 |
| `crank` | 核心：监听 Round 创建 → 等 VRF reveal → **自动发 `settle_spin`** 并代付手续费 | `logsSubscribe` + `accountSubscribe`；worker pool 控并发；失败重试 + 优先级费 bump；**密钥存 KMS/HSM，仅能调 `settle_spin`/`expire_round`，永不触碰金库 authority** |
| `indexer` | Round 明细、流水、玩家统计落库 | WS 订阅 + Postgres；量大后换 Yellowstone Geyser gRPC |
| `risk` | 动态注额校验、单地址限频、黑名单、金库健康度 | Redis 计数器；与链上硬限制形成"软+硬"双层 |
| `stat` | 实测 RTP、留存、排行榜、**周收入结算（回购/分红/金库）** | 定时任务 + Postgres 聚合 |
| `burndist` | $SPIN 回购 TWAP 执行 + Merkle distributor 生成 | 独立发布周期的 keeper，密钥与 crank 隔离 |
| **`gamectrl`** | **多游戏路由**：`game_id` → 交易构造器 / 结果解析器 / 风控参数的运行时注册表 | 详见 §5.4；**平台代码不因新增游戏而改动** |
| `admin` | 配置热更新、暂停开关（写链上需多签） | 内网接口 |

> **为什么 Go 特别适合本项目**：每局都要"reveal 完成后立刻 settle"，本质是高并发事件驱动 + 大量账户订阅 + 交易重试，这是 Go 的主场。

### 5.2 数据表（Postgres 摘要）

```sql
CREATE TABLE rounds (
  round_pda      TEXT PRIMARY KEY,
  game_id        TEXT NOT NULL,          -- 哪一款游戏
  wallet         TEXT NOT NULL,
  nonce          BIGINT NOT NULL,
  bet_lamports   BIGINT NOT NULL,
  payout_lamports BIGINT,
  symbols        SMALLINT[3],
  randomness     TEXT,
  request_slot   BIGINT,
  settle_sig     TEXT,
  created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE daily_revenue (
  day                DATE PRIMARY KEY,
  volume_lamports    NUMERIC NOT NULL,
  gross_lamports     NUMERIC NOT NULL,   -- = volume × 4.05%
  buyback_lamports   NUMERIC NOT NULL,   -- 50%
  staker_lamports    NUMERIC NOT NULL,   -- 30%
  bankroll_lamports  NUMERIC NOT NULL,   -- 15%
  ops_lamports       NUMERIC NOT NULL    -- 5%
);
```

- `rounds` 是审计对账的本地索引，**真值永远是链上账户**，不可用它覆盖链上。
- `daily_revenue` 直接支撑设计文档 §5.6 / §5.7 的执行与公示，分配比例从配置读取，必须与实际执行留痕一致。

### 5.3 周结算流程（对接双层代币模型）

```
每日 00:00  → 聚合昨日 gross = Σ bet_lamports × 4.05%
每周一      → 生成本周分配：buyback / staker / bankroll / ops
            → buyback：交给 burndist 按 TWAP 执行并 burn，记录 tx
            → staker：按质押快照生成 Merkle root，写入 distributor，用户 claim
            → 全部结果写入 daily_revenue + 对外公示面板
```

### 5.4 多游戏路由与热配置

**目标：新增一款游戏，Go 后端只加一个 adapter，不动平台代码。**

```go
// 每个 game_id 注册一套适配器
type GameAdapter interface {
    ID() string
    // 构造：bet → request_spin 指令（可能附带该游戏专有的额外账户如 GameAssets）
    BuildSpin(ctx BuildCtx, betLamports uint64) (solana.Instruction, error)
    // 解析：Round.result blob → 该游戏自己的结构体（平台不认识）
    ParseResult(blob []byte) (any, error)
    // 复算：给玩家的信任面板用；必须与链上 evaluate 逐字节一致
    Verify(randomness []byte, betLamports uint64) (VerifyResult, error)
    // 风控 per-game 参数来源
    RiskParams() RiskParams
}
```

要点：

| 要点 | 做法 |
| --- | --- |
| **热加载** | 游戏配置（`min_bet` / `max_bet` / `exposure_bps` / `enabled`）以 `GameConfig` 账户为准，Go 侧定时拉取并按 `assets_version` 刷新本地缓存，**禁止硬编码** |
| **crank 隔离** | 按 `game_id` 分 worker pool；某个游戏体验付费卡顿不会影响其他游戏的 settle 吞吐 |
| **数学对拍** | 每个 adapter 的 `Verify` 必须与 Rust 的 `evaluate` 对同一组随机数产出相同结果 —— CI 里跑 N 万组随机用例对拍（§10.1） |
| **下架生效** | `GameConfig.enabled == false` 时，`BuildSpin` 直接拒单并返回友好错误，前端立即把该游戏卡片置灰 |
| **不要让游戏逻辑进入平台层** | 平台只认 `game_id` 与 `Outcome.payout`；任何"如果是 X 游戏就……"的分支都说明抽象破了 |

---

## 6. Cocos 前端

### 6.1 场景与模块

> 下面是 Game #1（经典三列水果机）的场景树。每款游戏在自己的 Asset Bundle 里有各自的结构，平台只要求它们实现 `ISlotGame`（§6.1.1）。

```
SlotScene
├─ CanvasReels ── ReelController（3 × ReelColumn）
│                 └─ SymbolCellPool（对象池，Sprite + Tween）
├─ PaylineLayer     ← 中间行金色横线 + 中奖时连线高亮
├─ HudPanel         ← 余额(SOL) / 注额档位 / SPIN 按钮 / 金库健康度
├─ BenefitsPanel    ← 本周回购 / 我的分红 / 质押入口（$SPIN 权益入口）
├─ NetBridge (TS)   ← HttpClient + WsClient（纯 TS，不引 solana 依赖）
├─ WalletBridge     ← A/B/C 三种方案的统一抽象（§6.2）
└─ AudioMgr         ← 滚动 · 停止 · 中奖 · 大奖 音效
```

实现要点：

- **对象池 + Mask 裁剪**：每列 5~7 个 symbol 节点循环复用即可实现无限滚动；用 Mask 裁一列窗口。
- **运动模糊**：材质/shader 纵向拉伸模糊，或 Y 轴 scale 压缩近似，成本最低。
- **数据与表现分离**：9 格数据来自 WS 的 `window` 字段（列优先，含上中下三行）；**Cocos 侧不做任何符号生成或赔付计算**。
- **60fps 预算**：DrawCall 控制在 ~20 以内；符号图集合并（TexturePacker / Auto Atlas）。
- **弱网容错**：WS 心跳 + 自动重连；`round.pending` 超过 5s 未收 `settled` 则轮询 `/v1/rounds` 补拉，UI 显示"结果确认中"，**绝不本地编造结果**。

### 6.1.1 大厅 + Asset Bundle（多游戏的关键手段）

Cocos Creator 的 **Asset Bundle（分包）**机制天然适合"一个大厅 + N 款游戏"：

```
首包（很小）
└─ LobbyScene          ← 游戏卡片列表、余额、入口
   └─ Core bundle      ← NetBridge / WalletBridge / AudioMgr / GameSdk / 通用 UI

按需下载（assetManager.loadBundle）
├─ games/classic_fruits/    ← Game #1：9 格卷轴、对本局结果的解析器、复算器
├─ games/<未来游戏 2>/
└─ games/<未来游戏 3>/
```

带来的直接好处：

- **大厅秒开**，游戏资源进入时才下载；新增游戏不需要重新打包主程序，也不需要玩家更新 App（Web 端尤其明显）。
- **单个游戏更新只更新自己的 bundle**，其他游戏不受影响（相当于每个游戏的发布互不阻塞）。
- **二进制体积可控**，避免"加三款游戏后安装包翻三倍"。

**每个游戏 bundle 必须实现统一接口**（平台层只认这个接口）：

```ts
interface ISlotGame {
  readonly gameId: string;
  readonly assetsVersion: number;         // 必须匹配链上 GameAssets.version，否则拒绝启动
  mount(container: Node, sdk: GameSdk): Promise<void>;
  startSpin(): void;                      // 本地播放期望动画，真实结果由 sdk 事件驱动
  onRoundSettled(ev: RoundSettled): void; // ev.payload 由本游戏自己反序列化，平台不认识
  verifyRound(data: RoundTrust): VerifyResult;   // 自带复算器，供信任面板使用
  unmount(): void;
}
```

```ts
interface GameSdk {                       // 平台提供给游戏 bundle 的能力（唯一切入口）
  state(): Promise<PlayerState>;
  bet(gameId: string, lamports: bigint): Promise<string>;   // 返回 roundPda
  on(ev: SdkEvent, cb: (e: any) => void): void;
  trustData(roundPda: string): Promise<RoundTrust>;         // randomness + strip hash 等原始材料
}
```

> ⚠ **安全约束**：游戏 bundle **只能通过 `GameSdk` 发起下注**，拿不到私钥、不能自行构造交易、不能接触到 wallet API。这样即使未来引入第三方开发的游戏，也**无法**触及玩家资金 —— 这是能否开放第三方生态的前提。
>
> ⚠ **每个游戏自带复算器**：平台不理解 `payload`，所以"向玩家证明这局公平"必须由游戏 bundle 自己提供 `verifyRound`。上线 checklist 里这一项是硬性项。

### 6.2 钱包接入方案

Cocos Creator 3.x 脚本语言是 TypeScript，但**不建议在 Cocos 里直接引整套 `@solana/kit`**：一是 `@solana/wallet-adapter` 的成熟部分依赖 React DOM；二是原生平台缺 Node polyfill（Buffer、crypto），容易埋坑。

| 方案 | 适用端 | 做法 | 评价 |
| --- | --- | --- | --- |
| **A. WebView 钱包层（MVP 主方案）** | Web / iOS / Android 通用 | Cocos 之上覆盖一个 WebView（或独立签名页），用标准 wallet-adapter 完成连接与签名，结果经 `postMessage` / JSB bridge 回传 | 跨端一致性最好、复用成熟生态、升级无需动游戏工程；代价是有弹层切换感 |
| **B. Cocos TS 直连 wallet-standard** | 仅 H5 | 只引 `wallet-standard` 核心，自己实现 connect / signTransaction | 体验最顺滑；原生需 polyfill，谨慎用于生产 |
| **C. 原生桥 + MWA** | iOS / Android | 原生侧接入 Solana **Mobile Wallet Adapter**（deeplink / WS），Cocos 经 `jsb.reflection`（Android）/ `JsbBridge`（iOS）调用 | 移动端原生体验最好；需维护双端原生代码 |

**推荐路径**：MVP 全端用 **A**；H5 跑通后可针对桌面 / 钱包内置浏览器切 **B** 优化；原生版本量起来后再做 **C**。

#### 6.2.1 方案 B 是什么

**在 Cocos 的 TypeScript 里直接调用 Solana Wallet Standard —— 只用这套标准的纯 JS API（`getWallets()` + feature 调用），不引入 wallet-adapter 的 React 层，也不引沉重的 `@solana/web3.js` v1。**

```
Cocos(游戏逻辑) ──► WalletBridge(自研薄层) ──► window 上注册的 wallet-standard 钱包
                         │                        (Phantom / Backpack / Solflare …)
                         └──► @wallet-standard/app 的 getWallets()  ← 唯一运行时依赖
```

Wallet Standard 本身就是"钱包被发现 + 被连接 + 被请求签名"的通用协议，wallet-adapter 只是它外面的一层 React 封装。绕开封装直接调协议，就能在没有 React、没有 Node polyfill 的环境里用。

#### 6.2.2 适用边界（B 不能单独上线）

| 运行环境 | 是否可用 | 说明 |
| --- | --- | --- |
| 桌面浏览器 + 扩展钱包 | ✔ | 最佳场景 |
| 钱包内置浏览器 | ✔ | 移动端主要可用场景 |
| 移动设备桌面浏览器（iOS Safari / Android Chrome） | ✘ | **没有钱包向 window 注册**，须降级到 A 或 C |
| Cocos Native 包内置 WebView | ✘ | 同上 |
| 微信 / 抖音小游戏 | ✘ | 平台禁止加密货币支付 |

> 必须实现 `AdapterRouter`：启动时探测 `getWallets().get()`，有可用钱包走 B，无则自动降级到 A。**这条降级链要做进 MVP。**

#### 6.2.3 依赖清单与打包

**要引的**

| 包 | 用途 | 体积 |
| --- | --- | --- |
| `@wallet-standard/app` | `getWallets()`：发现与监听钱包注册 | 极小 |
| `@wallet-standard/base` | `Wallet` / `WalletAccount` 类型 | 0（纯类型） |
| `@wallet-standard/features` | `standard:connect` / `standard:events` 类型 | 极小 |
| `@solana/wallet-standard-features` | Solana 专有 feature 类型 | 极小 |
| `@solana/kit`（仅 decoder） | 反序列化交易做盲签校验 | 按需 tree-shake |

**不要引的**：`@solana/web3.js` v1、`@solana/wallet-adapter-*`。

**推荐打包路径 B-Ⅰ（预打包）**

```bash
# webauth/ 目录：独立小前端工程，只负责产出钱包 SDK bundle
pnpm i @wallet-standard/app @wallet-standard/base \
        @wallet-standard/features @solana/wallet-standard-features @solana/kit
pnpm exec esbuild src/index.ts --bundle --format=iife \
        --global-name=SolanaWalletSdk --minify \
        --outfile=../client/assets/scripts/vendor/solana-wallet-sdk.js
```

产物作为**普通脚本**放进 Cocos 的 `assets/scripts/vendor/`，Cocos 侧用 `declare global` 声明后调用：

```ts
// client/assets/scripts/wallet/SdkTypes.d.ts
declare const SolanaWalletSdk: {
  listWallets(): Promise<WalletInfo[]>;
  connectByName(name: string): Promise<{ address: string }>;
  signTransactionB64(txBase64: string): Promise<Uint8Array>;
};
```

优点：**完全绕开 Cocos 编辑器对 ESM/CJS 与 node_modules 的解析差异**。

<details>
<summary>B-Ⅱ：项目根目录 npm 直引（备选）</summary>

Creator 3.x 支持外部 npm 模块（`node_modules` 下通常无需配 import-map），可直接 `import { getWallets } from '@wallet-standard/app'`。
**风险**：ESM/CJS 互操作、`@solana/kit` 的 subpath export 与编辑器二次打包容易踩坑；第三方库若用到 `process`/`Buffer` 还需 polyfill。**建议**：仅在 B-Ⅰ 失败时使用。
</details>

#### 6.2.4 模块划分

```
WalletBridge（统一入口，游戏层只认它）
├─ AdapterRouter      ← 运行时选择 A / B / C
├─ StdWalletScanner   ← B 专属：getWallets() 发现 + feature 探测 + 缓存
├─ SessionManager     ← 记住上次钱包名、自动重连、处理账户切换
├─ TxInspector        ← 盲签防护：反序列化并白名单校验（§6.2.6）
└─ TxSigner           ← signTransaction → 已签名交易字节 → NetBridge 提交
```

```ts
interface WalletBridge {
  available(): Promise<WalletKind>;          // 'standard' | 'webview' | 'native' | 'none'
  connect(): Promise<{ address: string }>;
  signAndSendBuild(buildResp: BuildResp): Promise<string>;  // 返回交易签名
}
```

#### 6.2.5 钱包发现与连接

```ts
import { getWallets } from '@wallet-standard/app';
import type { Wallet } from '@wallet-standard/base';

const SOLANA_CHAIN = 'solana:devnet' as const;   // 上线切 'solana:mainnet'
const { get, on } = getWallets();

// 1) 发现：既读快照也订阅后续注册
const candidates: Wallet[] = [];
const push = (w: Wallet) => {
  const okSign = 'solana:signTransaction' in w.features;   // ← 关键 feature 探测
  if (okSign) candidates.push(w);
};
get().forEach(push);
const off = on('register', push);

// 2) solana 系钱包优先排前
const pick = (w: Wallet) =>
  w.chains.some((c: string) => c.startsWith('solana:')) ? 0 : 1;
candidates.sort((a, b) => pick(a) - pick(b));

// 3) 连接（幂等）
async function connectByName(name: string) {
  const w = candidates.find((x) => x.name === name)!;
  const connect = w.features['standard:connect']?.connect;
  const { accounts } = connect && w.accounts.length === 0
    ? await connect({ silent: false })
    : { accounts: w.accounts };

  // 4) 账户变化（切账户 / 断开）→ Cocos 侧需重新 SIWS 登录
  w.features['standard:events']?.on('change', ({ accounts }) => {
    if (!accounts.length) emit('disconnected');
    else emit('accountChanged', accounts[0].address);
  });

  return { address: accounts[0]!.address };   // base58 字符串
}
```

注意事项：

- **`standard:connect` 是可选 feature**；若 `w.accounts` 已有值说明已授权，直接用。
- **`solana:signTransaction` 才是必需探测项**。
- Wallet Standard 无强制 autoconnect；`silent: true` 只是提示不弹 UI，静默重连靠 localStorage 记住上次钱包名。
- chain 常量用 CAIP-2：`solana:mainnet` / `solana:devnet` / `solana:testnet` / `solana:localnet`。

#### 6.2.6 签名一笔 spin 交易

Go 的 `/v1/spin/build` 返回的交易包含 `Switchboard create+commit` → `request_spin` → ComputeBudget 提价三条指令，Cocos 侧工作是：**校验 → 签名 → 回传**。

```ts
async function signTransactionB64(txBase64: string): Promise<Uint8Array> {
  const txBytes = base64ToBytes(txBase64);

  await TxInspector.assertSafe(txBytes);         // ← 盲签防护，绝不能省

  const wallet = currentWallet!;
  const account = wallet.accounts.find(a => a.address === currentAddress)!;

  const [output] = await wallet.features['solana:signTransaction']!
    .signTransaction({
      account,
      chain: SOLANA_CHAIN,
      transaction: txBytes,
      options: { skipPreflight: false, preflightCommitment: 'confirmed' },
    });

  return output.signedTransaction;   // ⚠ 是"已签名完整交易"，不是签名本身
}
```

踩坑清单：

1. **`signedTransaction` ≠ signature**。它是完整交易字节流，直接 base64 后 POST 给 `/v1/tx/submit`，由 Go `sendRawTransaction` 广播。
2. **钱包本身不会改交易**，但若后端被入侵或 DNS 被劫持，**返回的 tx 本身可以是恶意的** → TxInspector 是安全底线。
3. **blockhash 有新鲜期**（约 60s / 150 slot）。`/spin/build` 返回体带 `lastValidBlockHeight`，过期或收到 blockhash not found 时自动重走一次 build。
4. **交易大小**：含创建 randomness 账户，用 **v0 版本化交易 + ALT** 压缩；Cocos 透传字节即可。
5. **每局都要弹钱包**：这是方案 B 的固有成本（解法见 §6.2.8 会话密钥）。UI 应提示"钱包弹窗已唤起"。

#### 6.2.7 盲签防护：TxInspector（重头戏）

玩家签名的是后端构造、自己看不懂的字节流。**一旦后端被入侵，一次签名就能转走全部资产。** 所以 Cocos 侧必须做第三方不可信校验：

```ts
class TxInspector {
  static async assertSafe(txBytes: Uint8Array) {
    const tx = decodeTransaction(txBytes);          // @solana/kit 的 decoder，别手写
    const instrs = getCompiledInstructions(tx);

    for (const ix of instrs) {
      // ① 程序白名单：只允许我们的程序 / Switchboard / ComputeBudget / System
      must(PROGRAM_ALLOWLIST.has(ix.programId), 'unknown_program');

      // ② Solana 原生转账（本项目筹码就是 SOL，这条尤其重要）
      if (ix.programId === SYSTEM_PROGRAM && parseSystemKind(ix.data) === 'Transfer') {
        // 只允许：玩家 → 自己的 player_vault，金额 ≤ 本次 bet
        must(isOwnPlayerVault(destOf(ix)) && amountLeqBet(ix), 'transfer_scope');
      }

      // ③ 危险指令一律拒绝
      must(!isSetAuthorityToStranger(ix), 'set_authority');
      must(!isDelegateApproval(ix), 'delegate_approval');
      must(!isCloseAccountToStranger(ix), 'close_account');
    }

    // ④ fee payer 必须是自己
    must(tx.feePayer === currentAddress, 'fee_payer_mismatch');
  }
}
```

配套动作：

- **签名预览页**：弹出"本次将下注 0.01 SOL 至合约 XXX，最多支付手续费 0.000005 SOL"。
- **allowlist 硬编码在 bundle 里**，构建期从 `shared/config.json` 注入，**绝不接受后端下发**（否则等于没校验）。
- **CI 加一条测试**：喂一段含 `SetAuthority` / 向陌生账户转账的恶意交易，断言必须抛错。

> 这是方案 B 相对 A 的**额外安全成本**：A 里出错由成熟钱包生态兜着，B 里所有安全都属于你自己。做不好就别上 B。

#### 6.2.8 状态机与错误处理

```
Idle → Detect（探测钱包 300ms 无结果 → 降级 A）
     → Select（列出候选 / 沿用 localStorage 上次选择）
     → Connecting → Ready
     → Signing → Submitting → (WS round.pending → round.settled) → Idle
     ↘ 任意失败 → Error → 可重试 → Idle
```

| 错误 | 处理 |
| --- | --- |
| 用户拒绝签名 | 不重试，回 Idle，文案"已取消" |
| 未检测到钱包 | 静默降级到方案 A，不弹错误打扰玩家 |
| 钱包不支持 `solana:signTransaction` | 从候选列表剔除 |
| blockhash 过期 / 交易被丢弃 | 自动重取 build 一次；再失败才提示玩家 |
| 账户被切换 | 清 JWT，重新 SIWS 登录，停新一局 |
| WS 断开 | 重连后按 roundPda 补拉结果 |

#### 6.2.9 会话密钥（消除"每局弹钱包"）

方案 A/B 的共同痛点是**每局都要玩家确认一次**。解法是链上会话密钥（V2，不进 MVP）：

```rust
// PlayerState 增加字段（示意）
session_key:        Pubkey,   // 玩家授权的一次性 key
session_expiry:     u64,      // 过期 slot（建议 ≤ 15 分钟）
session_limit:      u64,      // 累计可下注额度上限（lamports）
session_spent:      u64,
```

- 玩家用主钱包签一次 `authorize_session`，之后由后端用 session key 代签 `request_spin`，玩家侧零确认。
- 合约在 `request_spin` 中判断签名者：session key 走限额与过期校验，**不得触碰 `withdraw` / `deposit` 权限**。
- 注意：一旦采用会话密钥，"被动的 fee payer" 与 "主动的资金签名者" 就分离了，§9 的不变量 1 与 6 需要重新论证。

---

## 7. 接口协议（Cocos ⇄ Go）

**REST**

```
POST /v1/auth/challenge  {wallet}                 → {nonce}
POST /v1/auth/verify     {wallet, signature}      → {token}
GET  /v1/state?wallet=                            → {creditLamports, bankrollHealth}
GET  /v1/games                                    → [{gameId, enabled, minBet, maxBet, rtp, topMultiplier, assetsVersion}]
GET  /v1/games/{gameId}                           → {同上 + paytable + stripHash}   # 给前端展示与离线复算用
POST /v1/deposit/build   {wallet, amount}         → {txBase64, blockhash, expiry}
POST /v1/spin/build      {wallet, gameId, bet}    → {txBase64, randomnessPubkey, blockhash, expirySlot}
POST /v1/tx/submit       {wallet, signedTx}       → {signature, roundPda?}
POST /v1/withdraw/build  {wallet, amount}         → {txBase64}
GET  /v1/rounds?wallet=&limit=20                  → [{roundPda, symbols, payoutLamports, randomness, sig, slot}]
GET  /v1/benefits                                 → {weeklyBuybackLamports, myRewardsLamports, merkleRoot, claimable}
```

**WS（服务端 → 客户端，驱动动画）**

```jsonc
{"type":"round.pending","gameId":"classic_fruits","round":"<pda>","betLamports":10000000}
{"type":"round.settled","gameId":"classic_fruits","round":"<pda>",
 "payload":"eyJ3aW5kb3ciOi4uLn0=",   // base64：不透明负载，只有该游戏的 bundle 认识
 "payoutLamports":200000000,"creditLamports":1234000000,"sig":"...","randomness":"<pk>"}
{"type":"balance.changed","creditLamports":1234000000}
{"type":"risk.reject","reason":"bet_exceeds_max"}
{"type":"game.disabled","gameId":"<id>"}          // 游戏被下架，前端立即置灰卡片
```

要点：

- **`payload` 是不透明的**：平台层不理解它（可能是一次卷轴的 window，也可能是一次免费旋转的完整轨迹），原样转发给对应的游戏 bundle。**这样新增一款游戏，WS 协议本身不需要改一行。**
- 金额一律用 **lamports 整数字符串**传输，禁止浮点数，避免精度问题。
- **WS 只推送已上链确认的结果**，动画在收到 `round.settled` 后播放；本地不做赔付判定。
- `round.pending` 时开始滚动动画 —— VRF reveal 约 1–3 slot（≈0.8–1.2s），正好被三列依次停止的动画时长覆盖，等待感几乎为零。
- WS 断线自动重连 + 按 roundPda 补拉，防止"转了半天没结果"。

---

## 8. 一局完整时序

```
 Cocos              Go Backend                Solana
  │ 点击 SPIN        │                         │
  │─ POST /spin/build→│                         │
  │                  │ create randomness 指令   │
  │                  │ + request_spin 指令      │
  │←── txBase64 ─────│                         │
  │ 唤起钱包签名      │                         │
  │─ POST /tx/submit→│── sendRawTransaction ──→│ Round 创建（下注锁定）
  │←─ WS round.pending│←── logs ───────────────│
  │ 播放卷轴滚动      │                         │ Switchboard VRF
  │                  │←── randomness revealed ─│ reveal
  │                  │── settle_spin(crank) ──→│ 推导符号并结算 payout
  │←─ WS round.settled│←── logs ───────────────│
  │ 停止 + 中奖特效   │                         │
```

**玩家全程只签一次名**（settle 由 crank 代付），体感与 Web2 游戏一致。

---

## 9. 关键安全不变量

1. **不可预测**：`reveal_slot > request_slot`，随机数在下注之后才产生。
2. **不可重放**：一个 randomness 账户只能被消费一次（Round PDA seed 含 randomness pubkey + 用后置 `used` 标记；关闭账户回收 rent）。
3. **不可挑局**：`expire_round` 让"输了就不结算"的玩家被没收注额；crank 主动 settle 是常态。
4. **CU 预算**：Switchboard CPI 较贵，settle 需实测 profile，必要时 `compute budget` 提额。
5. **算术**：所有金额用 `checked_*`；符号查表用 `u32` 取模。
6. **权限**：`settle_spin` 任何人可调，但资金只能进 `PlayerState.credit`，不会外流。
7. **暂停开关**：`paused` 时禁止 request_spin，但允许 settle/withdraw。
8. **PDA 隔离**：vault seed 区分 player/house，禁止跨账户 CPI 到未校验程序。
9. **Rent 保护**：vault 永远保留 rent-exempt minimum，可用余额 = `lamports - rent_minimum`。
10. **crank 密钥最小化**：crank 热钱包只能发起 `settle_spin` / `expire_round`，不持有任何 authority；金库与参数由 Squads 多签管理。
11. **burn 密钥隔离**：$SPIN 回购执行密钥与 crank 密钥分离，且不涉及游戏金库。
12. **后端不可信原则**：Go 后端只做转发与预测，任何金额最终以链上账户为准；前端不得把后端单独给出的赔付数字当作唯一来源。
13. **游戏逻辑不得触碰资金**：无论哪种拓扑，`evaluate` 只能返回 `Outcome{payout, result}`，**不得直接读写任何 vault lamports**；演进到 B 拓扑后，per-game 程序只能经平台 CPI 操作资金，且必须在 `AuthorizedGame` 白名单内。
14. **单游戏下架不影响取款**：任何 `set_game_enabled(false)` 或全局 `paused`，**都必须保留 `settle_spin` 与 `withdraw` 可用**，否则玩家资金会被卡在游戏内。
15. **资产版本强校验**：`Round` 必须记录结算时使用的 `assets_version`；更新 paytable 或 strip 后，旧 Round 的复算结果会与新版本不同，没有版本号就无法做事后审计。

---

## 10. 测试、监控与发布

### 10.1 测试分层

| 层级 | 工具 | 覆盖 |
| --- | --- | --- |
| 数学验证 | Python 蒙特卡洛 1e8 + 解析公式 | RTP 收敛到 95.95%（设计文档 §4.5） |
| 合约单测 | `litesvm` / `solana-program-test` | 赔付表边界、三连 vs 两连判定、溢出、rent 保护 |
| Rust⇄Go 对拍 | 同一组随机数喂给两侧 | 符号推导与赔付结果必须逐位一致 |
| Go 集成测试 | `solana-test-validator`（Docker） | deposit/spin/settle/withdraw 全流程 |
| Fuzz | Trident | 随机 bet / 随机 slot / 畸形 randomness 账户 |
| 前端测试 | TxInspector 恶意交易断言 | 见 §6.2.7 |

### 10.2 监控告警

| 指标 | 告警阈值 |
| --- | --- |
| 未结算 Round 数 / 停留时长 | pending > 30s 的 Round 数 > 阈值 |
| crank 成功率 | settle 失败率 > 1% |
| 金库健康度 | 金库 / (1000 × 平均注) < 1.0 |
| 实测 RTP（滚动 100 万局） | 偏离 96% 超过 ±1% |
| 链上⇄账本差额（若用 §11.1） | 任何非零漂移立即告警并暂停提现 |
| RPC 延迟/错误率 | P99 > 阈值 或 error rate > 1% |

### 10.3 发布流程

1. devnet 全链路压测 → 2. 第三方审计 + 修复 → 3. Squads 多签接管所有 authority →
4. 限制额额灰度（max_bet 与全局限额设低）→ 5. 逐步提升限额 → 6. 发布 fallback 提现文档。

> **必须先发布 fallback**：告诉用户"当网站不可用时如何自己提现"，这条不兑现就配不上非托管叙事。

---

## 11. 备选资金架构

### 11.1 备选 X：中央账本 + 批量上链

> 需求：玩家不连钱包，充值记在服务器账本，游戏链下跑，资金批量发往链上。

**结论：技术上完全可行，但会把产品变成"链下赌博平台，链上只做充提通道"。** 三个连锁反应：信任模型降级、合规定性为托管（必须持牌 + KYC/AML）、被黑后果可能全损。

**三种批量粒度**

| 粒度 | 批什么 | 成本 | 可验证性 |
| --- | --- | --- | --- |
| L1 只批充提 | 提现批量转账、充值归集 | 最低 | 无 |
| L2 批局摘要 | batch 结果承诺 + 净额 | 低 | 中 |
| **L3 批余额快照（推荐）** | 全站账户树 **Merkle root + 负债总额** | ≈0.000005 SOL/次 | 较高（用户可自证被包含） |

推荐 L1 + L3 组合。

**账本必须 append-only 双分录**

```
ledger_entries(
  id, user_id, kind,          -- deposit|withdraw|bet|payout|bonus|fee|adjust
  asset, amount_delta,        -- 有符号，绝不用 UPDATE 覆盖余额
  ref_type, ref_id, balance_after, created_at
)
```

- 余额 = `SUM(amount_delta)`，权威值永远是账本求和。
- 下注原子性：`BEGIN; SELECT ... FOR UPDATE;` 或 Redis Lua 原子扣减 + 异步落库。**任何"先改缓存再落库"都必须在并发下证明幂等**。
- 充值幂等：以 `tx signature` 建唯一索引。

**随机性改用 Provably Fair**

```
① 开局前公开 server_seed_hash = sha256(server_seed)
② 客户端提供 client_seed
③ 结果 = HMAC_SHA256(server_seed, client_seed:nonce) → 3 个 reel 落点
④ 局后揭示 server_seed，玩家可自行复算
⑤ 每批承诺随 L3 Merkle root 上链
```

补救：**server_seed 揭示超时（如 24h）自动判负并赔付**，写入 ToS。

**生命线**

| 措施 | 做法 |
| --- | --- |
| 热钱包限额 | ≤ 日提现峰值 × 2；冷仓多签补仓 |
| 大额人工审核 | 超阈值进人工队列 + 延迟到账 |
| 对账 | 每 10 分钟核对；漂移超阈值 → **自动暂停提现并告警**（宁可停服，不可错付） |
| 储备金证明 | L3 Merkle root 做 PoR 页面 |

**与现方案对比摘要**

| 维度 | 链上合约（现方案） | 中央账本 + 批量 |
| --- | --- | --- |
| 用户门槛 | 需钱包、每局签名 | 邮箱即可，转化率显著提升 |
| 单局延迟 / 成本 | 0.8–1.5s / ~0.002 SOL | < 50ms / ~0 |
| 资金安全 | **非托管** | **托管（须持牌）** |
| 被黑后果 | 合约漏洞，损失有限 | 热钱包 + DB，可能全损 |
| 工期 | 合约 2 周 + VRF 1 周 | 省合约，但账本+充提+对账约 +2.5 周，且合规是外部成本 |

### 11.2 备选 Y：Passkey 智能钱包 + 会话密钥

> 需求：应用**自动为用户创建钱包**（用户不装 Phantom），资金在链上、用户可查，金额以链上为准。

**先拆开三个维度**（很多争论混为一谈）：

| 维度 | 由什么决定 |
| --- | --- |
| ① 资金可见性 | 资金是否在链上公开账户 |
| ② **资金控制权** | **谁能单方面转走**（私钥归谁 / 转移是否必须用户授权） |
| ③ 结果可信度 | 随机数是否链上可验证 |

托管 vs 非托管的判定**只看 ②**。"能在链上看见" ≠ "别人动不了"。

**信任分级**

| 档 | 形态 | 私钥控制 | 谁能单方面转走 | 定位 |
| --- | --- | --- | --- | --- |
| L0 | 中央账本 | 无链上私钥 | 平台 | 纯托管 |
| L1 | 自动创建钱包，**私钥平台持有** | 平台独有 | 平台 | **仍是托管** |
| L2 | MPC/TSS 分片 | 平台 + 用户各一片 | 需双方合作 | 接近非托管 |
| **L3** | **Passkey 智能钱包（PDA smart wallet）** | **用户设备 Secure Enclave** | **只有用户本人** | **非托管** |
| L4 | 用户自装 Phantom | 用户 | 只有用户本人 | 非托管（门槛最高） |

**推荐解（L3）**：Solana 已具备条件 —— secp256r1 验签 + instruction introspection + PDA 智能钱包 + relayer，开源实现可参考 LazorKit（passkey 认证 + session key delegation + gas sponsorship + on-chain RBAC）。

```
用户登录 → 首次触发 WebAuthn 创建 Passkey
  · 私钥生成并留在用户设备 Secure Enclave，永不出设备
  · 链上创建 PDA 智能钱包，登记 passkey 公钥为授权根
  · 平台没有私钥 → 无法单方面转账 → 非托管成立

资金流：用户 → 自己的 PDA 金库（链上，可查）
手续费：relayer / paymaster 代付（用户无需持有 SOL）

游戏：一次 Face ID 授权 session key
  → 只能调 request_spin，带额度上限 + 过期 slot
  → 提现：不在 session key 权限内，必须重新 Face ID
```

```rust
// PlayerWallet（PDA："wallet", user）
owner_root:      Secp256r1Pubkey,   // passkey 公钥（授权根）
session_key:     Pubkey,
session_expiry:  u64,
session_limit:   u64,
session_spent:   u64,
locked:          bool,

// 权限矩阵（链上强制）
authorize_session  ← 必须 owner_root（passkey 验签）
request_spin       ← session_key 或 owner_root，受 expiry / limit 约束
withdraw           ← 必须 owner_root（提现永不交给 session key）
deposit            ← 任何人（只能进，不能出）
```

**安全边界**

| 场景 | 结果 |
| --- | --- |
| 后端被入侵、session key 泄露 | 最多在额度内反复下注，无法提现；损失有上限 |
| 平台跑路 / 下线 | 用户可用 passkey 自行 withdraw，或换任意 RPC + 开源 ABI 提现 |
| **设备丢失** | 需设计恢复（第二 passkey / 备份设备 / 延迟社交恢复），否则资金永久锁死 —— **这是 L3 最大的产品风险，不是安全风险** |

**建议**：权限矩阵自研（不想把资金安全外包给第三方程序的存续与升级），客户端 passkey 交互与 relayer 可参考开源实现。**不要放进 MVP**，作为 M10 增量。

---

## 12. 工程目录

```
repo/
├─ programs/slot/              # Rust + Anchor（链上真值）
│  └─ src/{lib.rs,state/,instructions/,math/}
├─ backend/                    # Go 模块化单体
│  ├─ cmd/{api,crank,worker,burndist}
│  ├─ internal/{api,txfactory,crank,indexer,risk,stat,burndist,ws}
│  └─ internal/gen/anchor/     # IDL → Go bindings（代码生成产物，勿手改）
├─ client/                     # Cocos Creator 3.x 工程
│  └─ assets/scripts/{net,wallet,game,audio}
├─ webauth/                    # 方案 A 的 WebView 钱包签名页（可 React，独立构建）
├─ shared/
│  ├─ idl/slot.json            # 唯一来源（平台 + 全部游戏指令）
│  └─ games/${game_id}/        # 每个游戏一份数学资源，三方同源
│     ├─ paytable.json         # 链上 / Go 对账 / 前端展示 共用
│     ├─ strip.json            # 卷带数据 + hash
│     └─ verify.py             # 该游戏专属的蒙特卡洛验证与离线复算
├─ token dist/                 # Merkle distributor 脚本（分红发放）
└─ client/assets/games/        # Cocos 每个游戏一个 Asset Bundle 源码目录
   └─ classic_fruits/{scene,scripts,verify.ts}
```

**环境依赖**：Rust stable + `avm`（锁定 Anchor 版本）+ Solana CLI + Go 1.23+ + Node 18+。版本必须三者对齐，用 `avm use <ver>` 锁定，勿混用旧教程里的 coral-xyz 路径（已迁至 solana-foundation）。

---

## 13. 新增一款游戏的 checklist

平台化的验收标准是：**上线第 N 款游戏时，平台层代码零改动。** 若某一步被迫要改平台代码，说明 §3.3 的抽象漏了东西，应该回头补抽象而不是硬加分支。

| # | 项目 | 交付物 / 验收标准 |
| --- | --- | --- |
| 1 | **数学模型** | `games/${id}/paytable.json` + `strip.json` 定稿；蒙特卡洛 1e8 收敛到目标 RTP，与解析解偏差 < 0.05% |
| 2 | **联合风险重算** | 把新游戏并入现有组合后重算 **联合破产概率**（§3.5），ROR ≤ 1% |
| 3 | **风险参数** | 确定 `top_multiplier` / `max_exposure_bps` / `min_bet` / `max_bet_hard` |
| 4 | **Rust `SlotGame` impl** | `evaluate` 必须是纯函数；单测覆盖三连/两连边界、最大 bet 溢出、`assets_version` 切换 |
| 5 | **Go `GameAdapter` impl** | `BuildSpin` / `ParseResult` / `Verify` / `RiskParams` 四项齐全 |
| 6 | **Rust ⇄ Go 对拍** | CI 跑 N 万组随机用例，`Outcome` 逐字段一致 |
| 7 | **链上注册** | 多签执行 `register_game` → `upload_game_assets`，记录 `version` 与 `hash` |
| 8 | **Cocos bundle** | 实现 `ISlotGame`；入口从大厅加载；**必须自带 `verifyRound`** 且与链上一致 |
| 9 | **CU profile** | 实测 `settle_spin` 峰值 CU（§3.8），接近上限则拆分多帧结算 |
| 10 | **低额度灰度** | 先设 `max_bet = 计算值 × 10%`，跑够样本（建议 ≥ 10 万局）验证实测 RTP 后再放开 |
| 11 | **独立监控与一键下架** | per-game 实测 RTP 告警；异常时 `set_game_enabled(false)` 可立即止损 |
| 12 | **审计（A 拓扑下）** | 单程序形态下仍需对**增量代码**做一轮审计，审计方要确认未影响其他游戏的资金路径 —— 这正是未来演进到 B 拓扑（per-game 程序）的主要动因 |

**平台层零改动的检查点**：`request_spin` / `settle_spin` / `expire_round` / `deposit` / `withdraw` / WalletBridge / WS 协议 / crank 主循环 —— 新增游戏时这些文件都不应出现在 diff 里。
