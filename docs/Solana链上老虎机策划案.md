# $SPIN —— Solana 链上三列老虎机（Slot）策划案

> 版本：v0.2（草案）
> 定位：传统三列水果机（3 Reel × 3 Row = 9 格），单条 payline（中间一行），代币计价
> 技术栈：**Cocos Creator（前端）** + **Go（后端）** + Solana Anchor/Rust（链上） + Switchboard VRF

---

## 0. 一页速览

| 项目 | 设定 |
| --- | --- |
| 前端 | **Cocos Creator 3.x（TypeScript）**，发布 Web / iOS / Android |
| 后端 | **Go**（交易工厂 + crank 代付结算 + indexer + 风控 + WS 推送） |
| 链上 | Rust + Anchor，**唯一真值**（资金/随机/结算） |
| 玩法 | 3 列 × 3 行（9 格），唯一 payline = 中间一行 |
| 下注单位 | 游戏代币 $SPIN（或稳定币，见 §5） |
| 目标 RTP | 96.0%（house edge ≈ 4.05%） |
| Hit Frequency | ≈ 42.3%（其中"净赢"≈ 9.4%，其余为返还本金的 1× push） |
| 波动率 σ | ≈ 4.44 × 单注 |
| 最高奖 | 5000×（钻石三连），受动态限注保护 |
| 随机源 | Switchboard VRF（On-Demand），备选 slot-hash commit-reveal |
| 结算方式 | 两步交易：request_spin → settle_spin（permissionless + 超时惩罚） |
| 上线节奏 | 数学验证 → 合约 → 随机源 → 前端 → devnet 公测 → 审计 → 主网 |

---

## 1. 产品定义

传统 slot 的数字化复刻：不是多线视频老虎机，而是**三列机械水果机**——玩家一眼看懂，一次旋转 3 秒结束，快感密度高。

**核心循环**

```
充值代币 → 选注额 → 拉拉杆 → 三列依次停下 → 中间行判定 → 赔付入账 → 继续 / 提现
```

**设计原则**

1. **极简**：一条 payline、一张赔率表、一次点击。不做小游戏、不做多线、不做剧情。
2. **可验证公平**：每局结果由链上可验证随机数产生，任何人可用 round 数据 + 随机数复算符号与赔付。
3. **高频低摩擦**：链上余额（credit）模型，充值一次可连续旋转，提现才上链转账。
4. **代币闭环**：house edge 形成持续回购/分红买盘，赔付形成卖压，净差额由 4% 边际覆盖。

---

## 2. 玩法规则

### 2.1 九格与 payline

- 界面是 **3 列 × 3 行** 的窗口（9 个可见符号格）。
- 只有**中间一行**（Row 2）参与结算，即 payline = `[R1C2, R2C2, R3C2]`（列 × 中间行）。
- 上下两行是"窗口展示"：不是独立随机的，而是由中间行在虚拟 reel strip 上的位置 ±1 决定（见 2.2）。这让画面与真实机械机的物理卷轴完全一致，也让"差一格"的 near-miss 表现天然成立。

### 2.2 Reel Strip 机制（关键实现）

每条 reel 是一条**固定长度的虚拟卷带**（strip），长度 N = 1000，按 §3 的权重把符号填入并打乱（部署前链下生成、哈希固化写入链上配置账户）。

一局旋转，每列独立取一个随机落点 `pos ∈ [0, N)`：

```
该列显示的三格（从上到下）= strip[(pos - 1 + N) % N]   // Row 1
                            strip[pos]                 // Row 2  ← payline
                            strip[(pos + 1) % N]       // Row 3
```

好处：
- 上下行与中间行在视觉上"连续"，符合机械机直觉；
- near-miss 判定简单（第三列 pos 与目标只差 1 时播放抖动动画）；
- 随机数只需 3 个 `u32`，一次 VRF reveal（32 字节）足够；
- 权重即"strip 上该符号出现的格子数"，数学验证与链上实现完全对齐，无偏差。

### 2.3 结算规则

| 情形 | 判定 | 赔付 |
| --- | --- | --- |
| 三连 | 中间行 3 个符号相同 | 按 §4 三连赔付表 |
| 两连 | 中间行**恰好** 2 个符号相同（任意位置：AAB / ABA / BAA） | 按 §4 两连赔付表 |
| 无 | 3 个符号互不相同 | 0 |

> 说明：采用"任意两连"而非"左对齐两连"，是为了让 hit rate 落在合理区间；低符号两连给 1×（返还本金，等价于 push，不输不赢），保留传统水果机的手感。

### 2.4 下注档位

| 档位 | 单注（代币） |
| --- | --- |
| 1 | 1 |
| 2 | 5 |
| 3 | 25 |
| 4 | 100 |
| 5 | 500 |

赔付 = 档位注额 × 赔率倍数。实际 `max_bet` 由链上按金库规模动态计算（§7.2），前端只显示当前可用档位。

---

## 3. 符号与权重表

9 种符号，经典水果机谱系。权重 = strip 上该符号占用的格子数（N = 1000）。

| 符号 | ID | 权重 | 单列概率 p |
| --- | --- | ---: | ---: |
| 🍒 樱桃 CHERRY | 0 | 200 | 0.200 |
| 🍋 柠檬 LEMON | 1 | 200 | 0.200 |
| 🍊 橙子 ORANGE | 2 | 180 | 0.180 |
| 🫐 李子 PLUM | 3 | 150 | 0.150 |
| 🍉 西瓜 WATERMELON | 4 | 120 | 0.120 |
| 🔔 铃铛 BELL | 5 | 90 | 0.090 |
| 🟥 BAR | 6 | 45 | 0.045 |
| 7️⃣ 红七 SEVEN | 7 | 12 | 0.012 |
| 💎 钻石 DIAMOND | 8 | 3 | 0.003 |
| **合计** | | **1000** | **1.000** |

三条 reel 使用**同一条 strip**（权重一致），便于数学解析计算；V2 可给不同 reel 不同 strip 增加变化。

---

## 4. 赔率表与 RTP 数学

### 4.1 三连赔付（p³）

| 符号 | 三连概率 | 赔付（×bet） | RTP 贡献 |
| --- | ---: | ---: | ---: |
| 🍒 CHERRY | 0.008000 | 8 | 0.064000 |
| 🍋 LEMON | 0.008000 | 10 | 0.080000 |
| 🍊 ORANGE | 0.005832 | 15 | 0.087480 |
| 🫐 PLUM | 0.003375 | 20 | 0.067500 |
| 🍉 WATERMELON | 0.001728 | 40 | 0.069120 |
| 🔔 BELL | 0.000729 | 80 | 0.058320 |
| 🟥 BAR | 0.000091125 | 250 | 0.022781 |
| 7️⃣ SEVEN | 0.000001728 | 1000 | 0.001728 |
| 💎 DIAMOND | 0.000000027 | 5000 | 0.000135 |
| **小计** | 0.02775688 | | **0.451064** |

### 4.2 两连赔付（3p²(1−p)，恰好两个相同）

| 符号 | 两连概率 | 赔付（×bet） | RTP 贡献 |
| --- | ---: | ---: | ---: |
| 🍒 CHERRY | 0.096000 | 1 | 0.096000 |
| 🍋 LEMON | 0.096000 | 1 | 0.096000 |
| 🍊 ORANGE | 0.079704 | 1 | 0.079704 |
| 🫐 PLUM | 0.057375 | 1 | 0.057375 |
| 🍉 WATERMELON | 0.038016 | 2 | 0.076032 |
| 🔔 BELL | 0.022113 | 3 | 0.066339 |
| 🟥 BAR | 0.005801625 | 5 | 0.029008 |
| 7️⃣ SEVEN | 0.000426816 | 15 | 0.006402 |
| 💎 DIAMOND | 0.000026919 | 60 | 0.001615 |
| **小计** | 0.39546436 | | **0.508475** |

### 4.3 总指标

| 指标 | 数值 |
| --- | --- |
| **理论 RTP** | **95.95% ≈ 96.0%** |
| House Edge | 4.05% |
| Hit Frequency（任意赔付） | 42.32% |
| 其中"净赢"（>1×） | ≈ 9.41% |
| 其中"回本"（=1× push） | 32.91% |
| 输光概率 | 57.68% |
| E[X²] | ≈ 20.66 |
| 方差 Var | ≈ 19.74 |
| 标准差 σ | **≈ 4.44 × bet** |

### 4.4 金库充足度（Risk of Ruin）

用经典破产概率近似 `ROR ≈ exp(−2·B·edge / σ²)`（B 以"单注"为单位，edge = 0.0405，σ² = 19.74）：

| 目标破产概率 | 所需 Bankroll |
| --- | --- |
| 5% | ≈ 730 × 单注 |
| **1%** | **≈ 1120 × 单注** |
| 0.1% | ≈ 1680 × 单注 |

> 结论：以平均单注 B 计，金库应常备 **≥ 1000× 平均单注** 的代币。若平均注 25 代币，金库需 ≥ 25,000 代币等值。

### 4.5 数学验证（M0 必做）

在写合约之前先跑离线验证，避免"链上改参数"的高成本：

```python
# 伪代码：蒙特卡洛 1e8 局
strip = build_strip(weights)          # 长度 1000
for _ in range(100_000_000):
    cols = [strip[randrange(N)] for _ in range(3)]
    pay  = paytable_lookup(cols)      # 三连 / 恰好两连 / 无
    total_pay += pay
print(total_pay / N)                  # 应收敛到 0.9595 ± 0.0001
```

同时用解析法（上表公式）交叉验证，两者偏差 > 0.05% 说明 strip 生成或查表有 bug。

---

## 5. 代币经济设计

### 5.1 核心风险：代币计价的老虎机

如果"下注用代币、赔付也用代币"，则存在**双向价格风险**：

- 币价跌 → 玩家赢到的代币变现价值缩水 → 体验崩坏、用户流失；
- 币价跌 → 金库（以代币计）名义够但实际价值缩水 → 偿付能力下降；
- 玩家赢币即卖 → 持续卖压；house 赢币 → 回购销毁 = 买盘；净买盘 = 4% 边际 × 流水。

**结论：单代币模式在币价下行期会死亡螺旋。** 因此给出两个方案：

### 5.2 方案 A：单代币（$SPIN 全包）

下注、赔付、金库全用 $SPIN。
优点：叙事简单、代币需求直接。
缺点：上述价格风险全部承担；金库必须做**部分稳定币对冲**。

### 5.3 方案 B：双代币（推荐）

| 层 | 资产 | 作用 |
| --- | --- | --- |
| **筹码层** | SOL 或 USDC | 下注与赔付的计价单位，金库以稳定资产持有，偿付能力稳定 |
| **权益层** | $SPIN | 质押分红、回购销毁、治理、VIP 权益；不直接做筹码 |

收益分配（house edge 4% 的流水收入）：

```
流水收入 100%
├── 50%  二级市场回购 $SPIN 并销毁      → 持续买盘 + 通缩
├── 30%  $SPIN 质押者分红（按周，稳定币计价发放）→ 持币理由
├── 15%  金库补充（提升限注上限）
└── 5%   团队/运营
```

优点：币价波动不击穿金库；$SPIN 有真实现金流支撑（可以用 P/S 估值）。
缺点：SOL/USDC 计价对"炒币型用户"吸引力弱一些 → 用**代币奖励包**补：每笔流水额外赠送少量 $SPIN（rebate），让"玩 = 挖矿"，但 rebate 从营销预算出，不计入 RTP 的数学结构（或计入，需重新校准）。

> **建议：MVP 用方案 B，跑通后把 $SPIN 作为可选筹码开放（方案 A 通道），并对 $SPIN 通道设置更严格的限注。**

### 5.4 代币发行参数（若采用 $SPIN）

| 项目 | 设定 |
| --- | --- |
| 标准 | SPL Token（Token-2022 可选，但 transfer hook 会增加 CU，游戏内代币建议用标准 SPL） |
| 总供应 | 1,000,000,000（10 亿），铸造权限上线即销毁 |
|  decimals | 9（与 SOL 一致，便于换算） |
| 分配 | LP 40%（锁 12 个月）／社区与空投 20%／金库储备 15%／团队 15%（ cliff 12M + 线性 24M）／ treasury 10% |
| 销毁机制 | 每周回购销毁，链上可查 |
| 防狙 | 上线初期配合限注 + 单地址流水上限 |

---

## 6. 技术方案

### 6.1 技术栈总览

| 层 | 技术 | 说明 |
| --- | --- | --- |
| **前端** | **Cocos Creator 3.x（TypeScript）** | 卷轴动画、粒子、音效、UI；跨平台发布 Web / iOS / Android |
| **后端** | **Go 1.23+** | 交易工厂、crank 自动结算、indexer、风控、统计、WebSocket 推送 |
| **链上** | Rust + Anchor 0.30+ | 唯一真值：资金托管、随机数消费、符号推导、赔付结算 |
| 随机源 | Switchboard Randomness（VRF On-Demand） | 由 Go 侧协调创建 randomness 账户 |
| 通信 | HTTP REST + WebSocket | Cocos ⇄ Go；Go ⇄ Solana 用 `solana-foundation/solana-go` |
| 存储 | Postgres + Redis | 流水/明细/统计；限频与会话 |
| **备选架构 A** | **中央账本 + 批量上链**（§6.13） | 玩家不连钱包、余额记在服务器、链上只做充提与 Merkle 锚定 |
| **备选架构 B** | **Passkey 智能钱包 + 会话密钥**（§6.14） | 应用自动创建钱包、用户 Face ID 授权、私钥不出设备、资金在链上且平台动不了 |

> **一句话原则**：链上是真值，Go 是协调，Cocos 是表现。后端不托管资金、不决定结果，Cocos 不做任何赔付计算。
>
> ⚠ 若改用 §6.13 的中央账本架构，上述原则中的"后端不托管资金"不再成立，合规与安全要求整体升级。

### 6.2 Go ↔ Solana 适配性评估（结论：适配，做后端服务层很合适）

**先说结论：适配度 ★★★★☆（4/5）。适合做后端/工具链，但 Go 不能写 Solana 链上程序。**

| 维度 | 评价 |
| --- | --- |
| SDK 现状 | `solana-foundation/solana-go` —— **官方组织下**已有 Go SDK：v1 线延续 `gagliardetto/solana-go`（v1.24.0），另有重构后的 v2 API。说明 Go 客户端是被官方认可的方向，不是社区野库 |
| 能做什么 | JSON-RPC / WebSocket 客户端、交易构造与 ed25519 签名、PDA 派生（`find_program_address`）、版本化交易 v0 + Address Lookup Table、优先级费用 / ComputeBudget、durable nonce、SPL Token 与 ATA 辅助方法 |
| **不能做什么** | **写链上合约**。Solana 运行时（SBF）只支持 Rust / C，没有 Go → SBF 的编译链。**合约必须 Rust + Anchor**，这一点没有商量余地 |
| 主要缺口 | ① **无官方 Anchor 客户端**：8 字节 discriminator、IDL 编解码、Anchor 错误码都要自己生成/实现；② Token-2022 各扩展覆盖不全，用到需自测；③ 新 RPC / 新特性跟进晚于 TS 生态；④ 中文与问答资料远少于 TypeScript |
| 相对优势 | goroutine + channel 天然适合本项目最重的后端负载 —— **同时订阅成千上万个 Round 账户、并发发起 settle**，比 Node 单线程模型省心得多；单二进制部署、静态类型、风控/统计逻辑写起来扎实 |

**降低风险的四条做法**

1. **IDL → Go 代码生成**：Anchor 产出的 `slot.json` 写个小 codegen 脚本（Python/Go 均可），生成 `internal/gen/anchor/`：指令编码器、账户结构体、8 字节 discriminator、错误码映射。一次性成本约 1–2 天，之后改合约只需重跑脚本。
2. **共享常量单一来源**：`shared/paytable.json` + strip 数据被三方引用 —— Rust 侧构建时内联、Go 侧对账、前端展示。避免"三套表漂移"。
3. **测试用真链，不用 mock**：CI 里起 `solana-test-validator`（Docker），Go 集成测试直接连本地集群跑通 deposit/spin/settle/withdraw 全流程。
4. **先验证 Token-2022（若使用）**：若筹码走 Token-2022，早期就做一次转账/手续费的兼容性验证，别留到上线前。

### 6.3 三层架构与职责边界

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
│ Solana 链上 · Anchor Program: spin_slot                 │
│  HouseConfig │ Strip │ PlayerState │ Round │ ×2 Vault    │
│  ─ 资金托管 · VRF 消费 · 符号推导 · 赔付结算（真值）     │
└───────────────┬───────────────────┬─────────────────────┘
                │ CPI               │
        ┌───────▼────────┐  ┌───────▼─────────────┐
        │  SPL Token     │  │ Switchboard VRF     │
        └────────────────┘  └─────────────────────┘
```

**职责边界（务必写进代码 review 清单）**

| 能力 | Cocos | Go 后端 | 链上 |
| --- | ---: | ---: | ---: |
| 卷轴停止位置、符号、赔付金额 | 只读展示 | 转发/预测 | **唯一决定** |
| 玩家资金 custody | ✗ | ✗ | ✔ |
| 构造交易、发交易 | ✗（仅唤起签名） | ✔ | — |
| settle 发起（crank 代付） | ✗ | ✔ | permissionless（任何人可替它做） |
| 风控限注/限频 | ✗ | 软限制（体验层） | **硬限制**（合约级 `require!`） |

> **非托管承诺**：即使 Go 后端全站宕机，玩家仍可凭链上 `PlayerState` 直接用任意 RPC 构造 `withdraw` 取回资金。文档中要明说，并提供 fallback 页面/CLI。这是链上 slot 相对"链下积分赌博站"的核心卖点。

### 6.4 Go 后端模块设计

| 模块 | 职责 | 实现要点 |
| --- | --- | --- |
| `api` | REST + WS 网关，SIWS（Sign-In With Solana）签发 JWT | gin/echo；WS 用 gorilla 或 nhooyr；连接即订阅该钱包的事件流 |
| `txfactory` | 组装**未签名交易**返回 base64：含 ComputeBudget 提价 + 创建 randomness 账户 + `request_spin` | 依赖 `solana-go`；带 `lastValidBlockHeight` 与过期时间返回给前端 |
| `crank` | 核心：监听 Round 创建 → 等 VRF reveal → **自动发 `settle_spin`**（平台钱包签名代付手续费） | `logsSubscribe` + `accountSubscribe`；worker pool 控制并发；失败重试 + 优先级费 bump；**密钥存 KMS/HSM，该密钥仅能调 `settle_spin` / `expire_round`，永远不接触金库 authority** |
| `indexer` | Round 明细、流水、玩家统计落库 | WS 订阅 + Postgres；量大后换 Yellowstone Geyser gRPC（Go 生态支持好） |
| `risk` | 动态注额校验、单地址限频、黑名单、金库健康度 | Redis 计数器；与链上硬限制形成"软+硬"双层 |
| `stat` | 实测 RTP、留存、排行榜、周分红计算 | 定时任务 + Postgres 聚合 |
| `admin` | 配置热更新、暂停开关（写链上需多签） | 内网接口 |

> **为什么 Go 特别适合本项目**：每局都要一次"Reveal 完成后立刻 settle"，本质是高并发事件驱动 + 大量账户订阅 + 交易重试重放，这是 Go 的主场。用 Node 也能做，但在几千并发 Round 时资源与复杂度更高。

### 6.5 Cocos 前端与钱包适配

Cocos Creator 3.x 脚本语言就是 TypeScript，但**不建议在 Cocos 里直接引整套 `@solana/kit`**：
一是 `@solana/wallet-adapter` 的成熟部分依赖 React DOM（社区已有踩坑记录，只能在外面套一层 React 再通信）；二是原生平台（iOS/Android）缺 Node polyfill（Buffer、crypto），容易埋坑。

| 方案 | 适用端 | 做法 | 评价 |
| --- | --- | --- | --- |
| **A. WebView 钱包层（推荐，主方案）** | Web / iOS / Android 通用 | Cocos 之上覆盖一个 WebView（或独立签名页/应用内浏览器），里面用标准 wallet-adapter 完成连接与签名，结果经 `postMessage` / JSB bridge 回传 Cocos | 跨端一致性最好、复用成熟钱包生态、升级无需动游戏工程；代价是有弹层切换感 |
| B. Cocos TS 直连 wallet-standard | 仅 H5 | 只引 `wallet-standard` 核心，自己实现 `connect` / `signTransaction`，避开 React 依赖 | 体验最顺滑；原生需 polyfill，谨慎用于生产 |
| C. 原生桥 + MWA | iOS / Android | 原生侧接入 Solana **Mobile Wallet Adapter**（deeplink / WS 协议），Cocos 经 `jsb.reflection`（Android）、`JsbBridge`（iOS）调用 | 移动端原生体验最好；需维护双端原生代码 |

**推荐路径**：MVP 全端用 **A**；H5 跑通后可针对桌面/钱包内置浏览器切 **B** 优化体验；原生版本量起来后再做 **C**。

---

#### 6.5.1 方案 B 是什么（一句话）

**在 Cocos 的 TypeScript 里直接调用 Solana Wallet Standard —— 只用这套标准的纯 JS API（`getWallets()` + feature 调用），不引入 `@solana/wallet-adapter` 的 React 层，也不引沉重的 `@solana/web3.js` v1。**

钱包与本项目的关系变成：

```
Cocos(游戏逻辑) ──► WalletBridge(自研薄层) ──► window 上注册的 wallet-standard 钱包
                         │                        (Phantom / Backpack / Solflare …)
                         └──► @wallet-standard/app 的 getWallets()  ← 唯一运行时依赖
```

关键点：**Wallet Standard 本身就是"钱包被发现 + 被连接 + 被请求签名"的通用协议**，而 wallet-adapter 只是它外面的一层 React 封装。我们绕开封装直接调协议，就能在没有 React、没有 Node polyfill 的环境里用，这正是 Cocos 需要的。

#### 6.5.2 适用边界（先说清楚，B 不能单独上线）

| 运行环境 | Wallet Standard 是否可用 | 说明 |
| --- | --- | --- |
| 桌面浏览器 + 扩展钱包（Chrome/Firefox + Phantom 等） | ✔ | 最佳场景，弹窗小、确认快 |
| 钱包内置浏览器（Phantom App 内浏览 / Backpack 内浏览） | ✔ | 移动端主要可用场景 |
| 移动设备桌面浏览器（iOS Safari / Android Chrome） | ✘ | **没有钱包向 `window` 注册**，必须用方案 A（WebView 钱包页）或 C（MWA deeplink）兜底 |
| Cocos Native 包（iOS/Android App 内置 WebView） | ✘ | 同上，无注入环境 |
| 微信 / 抖音小游戏 | ✘ | 平台禁止加密货币支付，整体不可用 |

> **结论：方案 B 是"环境允许时的最优路径"，不是全量方案。** 必须实现 `AdapterRouter`：启动时探测 `getWallets().get()` 是否有可用钱包，有则走 B，无则自动降级到 A。这条降级链要做进 MVP，不要留到上线前。

#### 6.5.3 依赖清单与打包方式

**要引的（都很小，几乎全是类型定义）**

| 包 | 用途 | 运行时体积 |
| --- | --- | --- |
| `@wallet-standard/app` | `getWallets()`：发现与监听钱包注册 | 极小 |
| `@wallet-standard/base` | `Wallet` / `WalletAccount` 类型 | 0（纯类型） |
| `@wallet-standard/features` | `standard:connect` / `standard:events` 的 feature 类型 | 极小 |
| `@solana/wallet-standard-features` | `solana:signTransaction` 等 Solana 专有 feature 类型 | 极小 |
| `@solana/kit`（仅 decoder 部分） | 反序列化交易做**盲签校验**（§6.5.6） | 按需 tree-shake |

**不要引的**：`@solana/web3.js` v1（重、依赖多、Node polyfill 麻烦）、`@solana/wallet-adapter-*`（React 依赖）。

**打包路径：推荐 B-Ⅰ（预打包）**

```bash
# webauth/ 目录：一个独立的小前端工程，只负责产出钱包 SDK bundle
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

优点：**完全绕开 Cocos 编辑器对 ESM/CJS 与 node_modules 的解析差异**，改依赖只需重跑一条命令，出问题也只在一个工程里。

<details>
<summary>B-Ⅱ：项目根目录 npm 直引（备选）</summary>

Creator 3.x 支持外部 npm 模块（`node_modules` 下通常无需配 import-map），可直接 `import { getWallets } from '@wallet-standard/app'`。
**风险**：ESM/CJS 互操作、`@solana/kit` 的 subpath export 与编辑器二次打包容易踩坑；第三方库若用到 `process`/`Buffer` 还需 polyfill。
**建议**：只在 B-Ⅰ 验证失败或无打包工具链时使用。
</details>

#### 6.5.4 模块划分（Cocos 侧）

```
WalletBridge（统一入口，游戏层只认它）
├─ AdapterRouter      ← 运行时选择 A / B / C
├─ StdWalletScanner   ← B 专属：getWallets() 发现 + feature 探测 + 缓存
├─ SessionManager     ← 记住上次钱包名、自动重连、处理账户切换
├─ TxInspector        ← 盲签防护：反序列化并白名单校验（§6.5.7）
└─ TxSigner           ← 调 signTransaction → 返回已签名交易字节 → 交给 NetBridge 提交
```

游戏层只看到三个方法，A/B/C 日后切换改动为零：

```ts
interface WalletBridge {
  available(): Promise<WalletKind>;          // 'standard' | 'webview' | 'native' | 'none'
  connect(): Promise<{ address: string }>;
  signAndSendBuild(buildResp: BuildResp): Promise<string>;  // 返回签名！
}
```

#### 6.5.5 钱包发现与连接

```ts
// webauth/src/index.ts —— B-Ⅰ 的 SDK 内部实现
import { getWallets } from '@wallet-standard/app';
import type { Wallet } from '@wallet-standard/base';

const SOLANA_CHAIN = 'solana:devnet' as const;   // 上线切 'solana:mainnet'
const { get, on } = getWallets();

// 1) 发现：钱包会自行注册，我们既读快照也订阅后续注册
const candidates: Wallet[] = [];
const push = (w: Wallet) => {
  const okChain = true;                                   // 多数钱包对所有 chain 都注册
  const okSign  = 'solana:signTransaction' in w.features; // ← 关键 feature 探测
  if (okChain && okSign) candidates.push(w);
};
get().forEach(push);
const off = on('register', push);

// 2) 优先：solana 系 + 有 chain-specific 声明的钱包排在前面
const pick = (w: Wallet) =>
  w.chains.some((c: string) => c.startsWith('solana:')) ? 0 : 1;
candidates.sort((a, b) => pick(a) - pick(b));

// 3) 连接（幂等，已连接时直接返回现有账户）
async function connectByName(name: string) {
  const w = candidates.find((x) => x.name === name)!;
  const connect = w.features['standard:connect']?.connect;
  const { accounts } = connect && w.accounts.length === 0
    ? await connect({ silent: false })
    : { accounts: w.accounts };

  // 4) 账户变化（切账户 / 断开）→ Cocos 侧需重新做 SIWS 登录
  w.features['standard:events']?.on('change', ({ accounts }) => {
    if (!accounts.length) emit('disconnected');
    else emit('accountChanged', accounts[0].address);
  });

  return { address: accounts[0]!.address };   // base58 字符串
}
```

注意事项：

- **`standard:connect` 是可选 feature**，少数钱包没有；此时若 `w.accounts` 已有值说明已授权，直接用。
- **`solana:signTransaction` 才是必需探测项**，没有它的钱包直接不列入候选。
- Wallet Standard **没有强制的 autoconnect**，`silent: true` 只是提示不弹 UI；静默重连的实现方式是把上次钱包名存 localStorage，进来先 `connect({ silent: true })`。
- chain 常量用 CAIP-2：`solana:mainnet` / `solana:devnet` / `solana:testnet` / `solana:localnet`。

#### 6.5.6 签名一笔 spin 交易（主流程）

Go 的 `/v1/spin/build` 返回的交易包含三条指令（`Switchboard create+commit` → `request_spin` → 附带 ComputeBudget 提价），Cocos 这边的工作是：**校验 → 签名 → 回传**。

```ts
async function signTransactionB64(txBase64: string): Promise<Uint8Array> {
  const txBytes = base64ToBytes(txBase64);

  await TxInspector.assertSafe(txBytes);         // ← 盲签防护，绝不能省，见下节

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

容易踩的坑：

1. **`signedTransaction` ≠ signature**。它是完整的交易字节流，直接 base64 后 POST 给 `/v1/tx/submit`，由 Go `sendRawTransaction` 广播。别再手动拼 ER 签名。
2. **用户可能改交易吗？** Wallet Standard 语义上钱包不得修改你的交易；但**如果后端被入侵或 DNS 被劫持，返回的 tx 本身可以是恶意的** —— 所以下面的 TxInspector 是安全底线。
3. **blockhash 有新鲜期**（约 60s / 150 slot）。`/spin/build` 返回体要带 `lastValidBlockHeight`，Cocos 在开始滚动前若发现过期（或 submit 收到 blockhash not found），自动重走一次 build，不要让玩家卡住。
4. **交易大小**：含创建 randomness 账户，用 **v0 版本化交易**（`.create()` + ALT）把上限与费用压下来；Cocos 不需要理解版本，透传字节即可。
5. **每局都要弹钱包**：这是方案 B 的固有体验成本（见 §6.5.9 会话密钥优化）。UI 上给"钱包弹窗已唤起"提示，避免玩家以为卡死。

#### 6.5.7 盲签防护：TxInspector（重头戏）

玩家签名的是 Go 后端构造、自己完全看不懂的字节流。**一旦后端被入侵，签名即可转走全部资产。**所以在 Cocos 侧必须做**第三方不可信校验**：

```ts
class TxInspector {
  static async assertSafe(txBytes: Uint8Array) {
    const tx = decodeTransaction(txBytes);              // @solana/kit 的 decoder，别手写
    const instrs = getCompiledInstructions(tx);          // [{ programId, accounts, data }]

    for (const ix of instrs) {
      // ① 程序白名单：只允许我们的程序 / Switchboard / SPL Token / ComputeBudget / ATA
      must(PROGRAM_ALLOWLIST.has(ix.programId), 'unknown_program');

      // ② 资金动作限制
      if (ix.programId === SPL_TOKEN) {
        const kind = parseTokenIxKind(ix.data);
        // 只允许：向"自己的 vault/ATA"转账 / ATA 创建；且金额 ≤ 本次 bet
        must(kind === 'TransferChecked' || kind === 'Transfer' || kind === 'CreateAssociatedToken',
             'unexpected_token_ix');
        must(amountLeqBet(ix) && isOwnAccount(destOf(ix)), 'amount_or_dest_mismatch');
      }

      // ③ 危险指令一律拒绝
      must(ix.programId !== SYSTEM_PROGRAM || parseSystemKind(ix.data) !== 'Transfer'
           || isOwnAccount(destOf(ix)), 'sol_transfer_to_stranger');
      must(!isCloseAccountToStranger(ix), 'close_account');
      must(!isSetAuthorityToStranger(ix), 'set_authority');
    }

    // ④ fee payer 必须是自己，且不含额外的未知顶层指令
    must(tx.feePayer === currentAddress, 'fee_payer_mismatch');
  }
}
```

配套的产品动作：

- **签名预览页**：在 Cocos 里弹出"本次将下注 25 代币至合约 XXX，最多支付手续费 0.000005 SOL"，让玩家有知情点。
- **allowlist 硬编码在 bundle 里**，从 `shared/config.json` 构建期注入，**绝不接受后端下发**（否则等于没校验）。
- **CI 里加一条测试**：拿一段恶意交易（含 `SetAuthority`/`CloseAccount`）喂给 TxInspector，断言必须抛错。

> 这一节是方案 B 相对方案 A 的**额外安全成本**：方案 A 里抽风的是成熟钱包 UI + React adapter 生态；方案 B 里所有安全都属于你自己。做不好就别上 B。

#### 6.5.8 状态机与错误处理

```
Idle → Detect（探测钱包 300ms 内无结果 → 降级 A）
     → Select（列出候选钱包 / 或沿用 localStorage 上次选择）
     → Connecting → Ready
     → Signing → Submitting → (WS round.pending → round.settled) → Idle
     ↘ 任意失败 → Error → 可重试 → Idle
```

| 错误 | 处理 |
| --- | --- |
| 用户拒绝签名 | 不重试，回 Idle，文案"已取消" |
| 未检测到钱包 | 静默降级到方案 A（WebView），不弹错误打扰玩家 |
| 钱包不支持 `solana:signTransaction` | 从候选列表剔除 |
| 区块 hash 过期 / 交易被丢弃 | 自动重取 build 一次；再失败才提示玩家 |
| 账户被切换 | 清 JWT，重新 SIWS 登录，停新一局 |
| WS 断开 | 与 §6.6 一致：重连后按 roundPda 补拉结果 |

#### 6.5.9 进阶优化：会话密钥（消除"每局弹钱包"）

方案 B 最大的体验痛点是**每一局都要玩家点一次钱包确认**，与"一次点击一局"的 slot 爽感直接冲突。解法是在链上加一层会话密钥授权（V2，不进 MVP）：

```rust
// PlayerState 增加字段（示意）
session_pubkey:      Pubkey,    // 玩家授权的一次性 key
session_expiry_slot: u64,       // 过期 slot（建议 ≤ 15 分钟）
session_spent:       u64,       // 已消费额度
session_limit:       u64,       // 总额度上限
```

- 玩家用主钱包签一次 `authorize_session`（含 pubkey / expiry / limit），之后 **Go 后端用 session key 代签 `request_spin`**，玩家侧零确认。
- 合约在 `request_spin` 中判断签名者：session key 走限额与过期校验，**不得触碰 `withdraw` / `deposit` 权限**。
- 可与 §11 的 Ephemeral Rollup / delegate 方案合并评估。

**平台限制（务必提前知道）**

- 微信 / 抖音等**小游戏平台禁止加密货币与链上支付**，不能作为发布渠道。
- iOS App Store 对真钱赌博类 App 审核极严（通常要求当地牌照）。
- → **主渠道建议：Web / PWA 优先，其次是 Android（APK / 第三方商店）**。

### 6.6 接口协议（Cocos ⇄ Go）

REST：

```
POST /v1/auth/challenge  {wallet}                 → {nonce}
POST /v1/auth/verify     {wallet, signature}      → {token}
GET  /v1/state?wallet=                            → {credit, minBet, maxBet, bankrollHealth, rtpTip}
POST /v1/deposit/build   {wallet, amount}         → {txBase64, blockhash, expiry}
POST /v1/spin/build      {wallet, bet}            → {txBase64, randomnessPubkey, blockhash, expirySlot}
POST /v1/tx/submit       {wallet, signedTx}       → {signature, roundPda?}
POST /v1/withdraw/build  {wallet, amount}         → {txBase64}
GET  /v1/rounds?wallet=&limit=20                  → [{roundPda, symbols, payout, randomness, sig, slot}]
```

WS（服务端 → 客户端，驱动动画）：

```jsonc
{"type":"round.pending","round":"<pda>","bet":25}
{"type":"round.settled","round":"<pda>",
 "window":[[1,0,8],[2,0,4],[5,0,3]],   // 9 格，列优先，每列 [上,中,下]
 "payline":[0,0,0],                    // 中间行 = 参与结算的 3 个符号
 "payout":200,"credit":1234,"sig":"...","randomness":"<pk>"}
{"type":"balance.changed","credit":1234}
{"type":"risk.reject","reason":"bet_exceeds_max"}
```

要点：

- **WS 只推送已上链确认的结果**，动画在收到 `round.settled` 后播放；本地不做任何赔付判定。
- `round.pending` 时前端开始播放滚动动画 —— VRF reveal 约 1–3 slot（≈0.8–1.2s），正好被 3 列依次停止的动画时长覆盖，等待感几乎为零。
- WS 断线自动重连 + 重连后按 `roundPda` 补拉结果，防止"转了半天没结果"。

### 6.7 一局完整时序（Cocos + Go + 链上）

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

### 6.8 账户设计

| 账户 | PDA Seeds | 关键字段 |
| --- | --- | --- |
| `HouseConfig` | `["house"]` | `authority`, `mint`, `house_vault`, `treasury`, `fee_bps`, `min_bet`, `max_bet_hard`, `max_exposure_bps`(500), `paused`, `strip_hash`, `top_multiplier`(5000) |
| `Strip` | `["strip"]` | 1000 字节符号表（或 9 个 (symbol, start, end) 区间以省空间）+ `hash` |
| `PlayerState` | `["player", user]` | `credit: u64`, `nonce: u64`, `wagered: u128`, `won: u128`, `last_settle_slot` |
| `Round` | `["round", player, nonce]` | `bet: u64`, `request_slot`, `randomness: Pubkey`, `reveal_slot`, `symbols: [u8;3]`, `payout: u64`, `status` |
| `PlayerVault` | `["vault", "player"]` | PDA 拥有的 SPL token account（托管全部玩家 credit） |
| `HouseVault` | `["vault", "house"]` | PDA 拥有的 SPL token account（庄家准备金） |

> 空间优化：strip 存区间表（每符号 3 个 u16 = 54 字节）比存 1000 字节省 ~95% 空间与 rent；代价是落点换算需一次扫描（CU 可忽略）。

### 6.9 指令

```
initialize_house(...)              # 部署初始化
init_player()                      # 首次进入创建 PlayerState
deposit(amount)                    # 用户 → PlayerVault，credit += amount
request_spin(bet, randomness)      # 冻结 bet，创建 Round，绑定 randomness 账户
settle_spin(round)                 # permissionless：读 reveal 值 → 算符号 → 结算（Go crank 发起）
expire_round(round)                # 超时未结算：bet 充公入 house，Round 关闭
withdraw(amount)                   # credit -= amount，PlayerVault → 用户
house_withdraw / pause / update    # 管理端（Squads 多签）
```

### 6.10 随机源选型

| 方案 | 安全性 | 成本/局 | 延迟 | 结论 |
| --- | --- | --- | --- | --- |
| **Switchboard Randomness (VRF, On-Demand)** | 高（密码学可验证，抗验证者操纵） | ~0.001–0.002 SOL（创建 randomness 账户） | 1–3 slot | **推荐** |
| ORAO VRF | 高 | 类似 | 1–2 slot | 备选 |
| SlotHashes commit-reveal | 中（验证者可小额操纵，仅适合小额） | 0 | 1 slot | 仅作"快速小额模式"备选 |
| 区块哈希 / 时间戳 | 低 | 0 | 0 | **禁止使用** |

> 注：Solana 无原生 VRF precompile，必须依赖外部随机源或 commit-reveal。

### 6.11 关键安全不变量

1. **不可预测**：`reveal_slot > request_slot`，随机数在下注之后才产生，杜绝"看到结果再下注"。
2. **不可重放**：一个 randomness 账户只能被消费一次（Round PDA seed 含 randomness pubkey + 用后置 `used` 标记；关闭 randomness 账户回收 rent）。
3. **不可挑局**：`expire_round` 让"输了就不结算"的玩家被没收注额，Go crank 主动 settle 是常态。
4. **CU 预算**：Switchboard CPI 较贵，settle 指令需 profile，必要时拆分或使用 `compute budget` 提额。
5. **算术**：所有金额用 `checked_add/checked_mul`，符号查表用 `u32` 取模。
6. **权限**：`settle_spin` 任何人可调（无 signer 依赖资金权限），但资金只能进 `PlayerState.credit`，不会外流。
7. **暂停开关**：`paused` 时禁止 request_spin，但允许 settle/withdraw。
8. **PDA 隔离**：所有 vault seed 区分 player/house，禁止跨账户 CPI 到未校验程序。
9. **crank 密钥最小化**：crank 用的热钱包只能发起 `settle_spin` / `expire_round`，不得持有任何 authority；多签（Squads）单独管理金库与参数。
10. **后端不可信原则**：Go 后端只做转发与预测，任何金额展示最终以链上账户为准；前端不得采信后端单独给出的"赔付数字"作为唯一来源。

### 6.12 工程目录建议

```
repo/
├─ programs/slot/              # Rust + Anchor（链上真值）
│  └─ src/{lib.rs,state/,instructions/,math/}
├─ backend/                    # Go 模块化单体
│  ├─ cmd/{api,crank,worker}
│  ├─ internal/{api,txfactory,crank,indexer,risk,stat,ws}
│  └─ internal/gen/anchor/     # IDL → Go bindings（代码生成产物，勿手改）
├─ client/                     # Cocos Creator 3.x 工程
│  └─ assets/scripts/{net,wallet,game,audio}
├─ webauth/                    # 方案 A 的 WebView 钱包签名页（可用 React，独立构建）
├─ shared/
│  ├─ idl/slot.json            # 唯一来源
│  └─ paytable.json            # 链上 / Go 对账 / 前端展示 三方同源
└─ math/                       # Python 蒙特卡洛验证脚本（§4.5）
```

### 6.13 备选架构 X：中央账本（Off-chain Ledger）+ 批量上链

> 需求来源：玩家不连钱包、不逐局签名，充值记在中心服务器账本上，游戏链下跑，资金**批量**发往链上。

#### 6.13.1 结论先行

**技术上完全可行，而且是行业绝大多数链上赌博站的实际做法。但它会改变产品的性质：**
从"链上非托管、每局可验证的 slot"变成"链下赌博平台，链上只做充提通道 + 锚定"。

由此带来三个连锁反应，必须在决定前接受：

1. **信任模型**：核心卖点从"每局链上可验证"降级为"Provably Fair + 链上锚定"。可用 §6.13.5 补到接近，但做不到等价。
2. **合规性质**：服务器持有用户资金 = **托管（custodial）**。原本"我们不掌握用户资金"的说法不再成立，博彩牌照 + KYC/AML 从"建议"变成"必需"。
3. **被黑后果**：从"合约漏洞损失有限"变成"热钱包 + 数据库，可能全损"。

**如果这三条都能接受，账本模式在体验与成本上是明显更优的。**

#### 6.13.2 三种"批量上链"的粒度（先选这个）

| 粒度 | 批什么上链 | 成本 | 可验证性 | 说明 |
| --- | --- | --- | --- | --- |
| **L1 只批充提** | 仅提现批量转账、充值归集 | 最低 | 无 | 最简单；链上完全看不到游戏数据，玩家 100% 信任服务器 |
| **L2 批局摘要** | 每 N 局 / 每 T 提交一个 batch 的**结果承诺 + 净额** | 低 | 中 | 能证明"这批局的聚合结果"，但单局无法独立自证 |
| **L3 批余额快照（推荐）** | 每 5–15 分钟提交一次**全站账户树的 Merkle root + 负债总额 + 局数** | 极低（≈0.000005 SOL/次，一天 < 0.002 SOL） | 较高 | 玩家可拿 Merkle proof 自证"我的余额被包含在这个 root 里"，接近 rollup 的信任模型 |

> **推荐 L1 + L3 组合**：资金动作用 L1，信任锚定用 L3。成本几乎为零，却能把"纯信托"提升到"可自证 + 事后可审计"。

#### 6.13.3 架构

```
玩家（无需钱包，邮箱/社交登录）
        │
        ▼
   Cocos 前端 ──── HTTPS + WS ────┐
                                  │
                          ┌───────▼──────── Go 后端 ─────────┐
                          │ Ledger      （Postgres 双分录账本）│ ← 余额真值（链下）
                          │ GameEngine  （commit-reveal RNG） │
                          │ DepositWatcher（链上入账监听）    │
                          │ WithdrawBatcher（批量打包提现）   │
                          │ RootPublisher（周期提交 Merkle）  │
                          │ Reconciler  （链上⇄账本对账）      │
                          │ Risk        （限频/限额/风控）     │
                          └───────┬───────────────────────────┘
                                  │ JSON-RPC / WS
                          ┌───────▼───────────────────────────┐
                          │ Solana                             │
                          │ · 每用户 deposit address（HD 派生）│
                          │ · 提现批处理（热钱包，v0 + ALT）   │
                          │ · Anchor 账户：Merkle root 序列    │
                          │ · 冷仓（Squads 多签）              │
                          └────────────────────────────────────┘
```

此模式下 §6.5 的钱包桥、§6.9 的指令、§6.10 的 VRF **整体不再需要**；链上合约退化为一个很小的"锚定 + 充提"程序。

#### 6.13.4 资金流与账本设计（最容易出事的地方）

**充值**
- 每个用户分配一个 deposit address（服务端 HD 派生，**私钥永不在线**，或用集中地址 + 唯一 memo）。
- `DepositWatcher` 监听 → **达到确认阈值**（建议 ≥ 1 确认 / 32 slot，大额更长）→ 入账。
- **幂等**：以 `tx signature` 建唯一索引，防止回调重放重复入账。

**账本（必须是 append-only 双分录）**

```
ledger_entries(
  id, user_id, kind,            -- deposit|withdraw|bet|payout|bonus|fee|adjust
  asset, amount_delta,          -- 有符号，绝不用 UPDATE 覆盖余额
  ref_type, ref_id,            -- 关联 round / tx
  balance_after, created_at
)
```

- 余额 = `SUM(amount_delta)`（配物化视图或 Redis 缓存加速），**权威值永远是账本求和**。
- 下注原子性：`BEGIN; SELECT ... FOR UPDATE;` 扣余额并写 round，或用 Redis Lua 原子扣减 + 异步落库。**任何"先改缓存再落库"的写法都必须在并发下证明幂等**，否则必被薅。
- 禁止负余额：`CHECK` 约束或事务内断言。

**提现**
- 用户提交 → 进入 `withdraw_queue`（状态机：pending → batched → signed → confirmed / failed）。
- 批处理：每 30–60s 或满 N 笔打包；单交易上限 1232 字节，用 **v0 + ALT** 大约可塞 20–30 笔 SPL 转账，超量自动分批。
- 失败交易：重试 + 优先级费 bump；连续失败转入人工队列。

#### 6.13.5 随机性：用 Provably Fair 补回信任

失去链上 VRF 后，必须换一套可验证方案，否则产品没有任何可信度：

```
① 开局前：公开 server_seed_hash = sha256(server_seed)（按场次/按用户轮换）
② 客户端提供 client_seed（可让玩家自定义或点击随机）
③ 结果 = HMAC_SHA256(server_seed, client_seed:nonce) → 派生 3 个 reel 落点
④ 局后揭示 server_seed，玩家可自行复算整局
⑤ 每批的 seed 承诺 / 结果承诺随 L3 的 Merkle root 一起上链
```

| 对比 | 链上 VRF（原方案） | Provably Fair + L3 锚定 |
| --- | --- | --- |
| 单局延迟 | 0.8–1.5s | **< 50ms** |
| 单局成本 | ~0.002 SOL | **~0** |
| 抗服务器作弊 | 密码学保证 | 承诺后不可改，但理论可"不揭示"→ 靠定期上链 + 揭示超时惩罚缓解 |
| 玩家理解门槛 | 低（链上就是证据） | 中（需要 UI 讲清楚） |

> 补救措施：**server_seed 揭示超时（如 24h）自动判负并赔付**，写入 ToS；每批 root 上链使"事后改历史"在计算上不可行。

#### 6.13.6 风控与对账（账本模式的生命线）

| 措施 | 做法 |
| --- | --- |
| **热钱包限额** | 热钱包余额 ≤ 日提现峰值 × 2；冷仓（Squads 多签）定时补仓。被黑损失上限定死 |
| **大额人工审核** | 超过阈值的提现进人工队列 + 延迟到账 |
| **对账** | 每 10 分钟核对 `链上资金 == 账本负债 + 储备`；漂移超阈值 → **自动暂停提现并告警**（宁可停服，不可错付） |
| **储备金证明** | 用 L3 的 Merkle root 做 PoR 页面，玩家可自证余额被包含 |
| **并发/双花** | 事务锁 + 唯一索引 + 幂等键 |
| **数据库** | 主从 + PITR 备份；账本表 append-only，禁止物理删除（改做冲正分录） |

#### 6.13.7 与链上方案的对比（决策表）

| 维度 | 链上合约（§6 原方案） | 中央账本 + 批量上链 |
| --- | --- | --- |
| 玩家门槛 | 需钱包、每局签名 | 邮箱/社交登录即可，**转化率显著提升** |
| 单局延迟 | 0.8–1.5s（含 VRF） | < 50ms |
| 单局成本 | ~0.002 SOL | ~0 |
| 可验证性 | 每局链上可验证（最强） | Provably Fair + Merkle 锚定（中） |
| 资金安全 | **非托管** | **托管（服务器持有）** |
| 合规 | 灰色，可辩"非托管" | **必须持牌 + KYC/AML + 地理围栏** |
| 被黑后果 | 合约漏洞，损失有限 | 热钱包 + DB，可能全损 |
| 工程重心 | 合约 + Go crank + Cocos | 账本/对账/批处理/风控（无合约） |
| 工期影响 | 合约 2 周 + VRF 1 周 | 省合约与 VRF，但账本+充提+对账+批处理约 +2.5 周，且**合规是外部成本** |
| 代币叙事 | "链上游戏"，强 | 弱；筹码建议直接用 USDC/SOL，代币退为权益层（正好落在 §5 方案 B） |

#### 6.13.8 建议路径

1. **若目标是大众用户与游戏体验** → 走账本模式，但务必同时做：L3 Merkle 锚定、Provably Fair、热钱包限额、自动对账止损、持牌与 KYC。
2. **若目标是加密原生用户与代币溢价** → 保持链上方案，"每局可验证"才是卖点。
3. **折中：双轨制（推荐考虑）** —— 同一套数学、同一套前端、两套资金轨：
   - `链上轨`（非托管，加密用户，额度高）
   - `账本轨`（快速，大众用户，额度低，需持牌）
   代价是后端要维护两套结算与对账，合规成本按最严的一轨算。
4. **MVP 顺序建议**：先用**账本模式**验证玩法与留存（快、便宜、无钱包门槛），跑通数据后再上链上非托管版本作为"Pro 模式"。反过来的顺序（先做链上）会因钱包门槛导致早期数据很差，难以判断玩法本身是否成立。

### 6.14 嵌入式钱包 + 会话密钥：体验与信任的折中解

> 需求来源：应用**自动为用户创建钱包**（用户不装 Phantom），资金在链上、用户可自行查看，金额以链上为准。这样信任度会不会降低？

#### 6.14.1 先把三件事拆开（很多争论都在混为一谈）

| 维度 | 回答的问题 | 由什么决定 |
| --- | --- | --- |
| **① 资金可见性** | 用户能不能在 explorer 看到自己的钱？ | 资金是否在链上公开账户 |
| **② 资金控制权** | 谁能**单方面**把钱转走？ | **私钥归谁 / 转移是否必须用户本人授权** |
| **③ 结果可信度** | 这局是不是真的随机、没被改？ | 随机数是否链上可验证 |

**你描述的方案解决了 ①，但没有自动解决 ② 和 ③。** 而"托管 vs 非托管"的判定、以及合规与第三方评级机构的评级，**只看 ②**。

一句话：**"能在链上看见" ≠ "别人动不了"。** 一个持有全部私钥的服务器，转走用户资金时链上同样"看得见"——威慑力远小于直觉。

#### 6.14.2 信任分级（L0 → L4）

| 档 | 形态 | 私钥控制 | 资金在哪 | 谁能单方面转走 | 评级定位 |
| --- | --- | --- | --- | --- | --- |
| **L0** | 中央账本（§6.13） | 无链上私钥 | 服务器 DB | 平台 | 纯托管 |
| **L1** | **自动创建钱包，私钥由平台托管** | **平台独有** | **链上，用户可查** | **平台** | **仍是托管**（资金可见性提升，控制权未变） |
| **L2** | MPC / TSS 分片（平台 + 用户各持一片） | 双方协作 | 链上 | 需双方合作 | 接近非托管 |
| **L3** | **Passkey 智能钱包（PDA smart wallet）** | **用户设备 Secure Enclave** | 链上 PDA | **只有用户本人** | **非托管** |
| **L4** | 用户自装 Phantom 等 | 用户 | 用户账户 | 只有用户本人 | 非托管（门槛最高） |

**关键结论：**
- 如果只是"应用自动创建钱包、私钥存在服务器"，你落在 **L1**。相比 L0 是提升（资金透明、可审计），但**性质仍是托管**——用户拿不到私钥，平台随时可转移资金。
- 要既保留"零门槛"又拿到非托管评级，必须走 **L2 或 L3**。L3 在 Solana 上已经成熟可用。

#### 6.14.3 推荐解：Passkey 智能钱包 + 会话密钥

Solana 已具备实现条件：**secp256r1 验签程序 + instruction introspection + PDA 智能钱包 + relayer**。已有开源实现可参考（如 LazorKit：passkey 认证 + session key delegation + gas sponsorship + on-chain RBAC 权限）。

**核心机制**

```
用户登录（邮箱/社交）→ 首次触发 WebAuthn 创建 Passkey
  · 私钥生成并保存在用户设备 Secure Enclave，永不出设备
  · 链上同时创建一个 PDA "智能钱包"，把 passkey 公钥登记为授权根
  · 平台没有私钥 → 无法单方面转账 → 非托管成立

资金流：用户 → 自己的 PDA 金库（链上，用户随时可在 explorer 查看）
手续费：relayer / paymaster 代付（用户无需持有 SOL，无 gas 感知）

游戏：一次 Face ID 授权 session key（会话密钥）
  → session key 只能调 request_spin，带额度上限 + 过期 slot
  → 之后每局由后端用 session key 代签，玩家零打扰
  → 提现：不在 session key 权限内，必须重新 Face ID
```

架构落点（推荐自研，把资金主体放自己程序里）：

```rust
// PlayerWallet（PDA："wallet", user）
owner_root:        Secp256r1Pubkey,   // passkey 公钥（授权根）
session_key:       Pubkey,            // 会话密钥（可轮换）
session_expiry:    u64,               // 过期 slot
session_limit:     u64,               // 累计可下注额度
session_spent:     u64,
locked:            bool,              // 风控冻结

// 指令权限矩阵（链上强制，不靠后端自觉）
authorize_session  ← 必须 owner_root（passkey 验签）
request_spin       ← session_key 或 owner_root，且受 expiry / limit 约束
withdraw           ← 必须 owner_root（提现永不交给 session key）
deposit            ← 任何人（只能进，不能出）
```

**安全边界（写清楚这些，评级才有说服力）**

| 场景 | 结果 |
| --- | --- |
| 平台后端被入侵、session key 泄露 | 攻击者最多在**额度内反复下注**，无法提现、无法转给第三方；损失有上限 |
| 平台跑路 | 用户可自行用 passkey 发起 `withdraw`，取回链上资金 |
| 平台下线/域名失效 | 用户仍可换任意 RPC + 官方开源 ABI 提现（**必须提供这个文档**） |
| 设备丢失 | 需设计恢复机制（第二 passkey / 备份设备 / 延迟社交恢复），否则用户资金永久锁死——**这是 L3 最大的产品风险，不是安全风险** |

#### 6.14.4 三个"真值"必须分开声明

| 真值 | 是否上链 | 说明 |
| --- | --- | --- |
| **资金余额** | ✔ 链上 | 你说的"以链上为准"，成立 |
| **资金控制权** | ✔ 链上（PDA 权限矩阵） | 由 passkey 与 session key 约束，成立 |
| **游戏结果随机性** | ✘ 每局链下 | **不能靠"金额以链上为准"来补** |

⚠ **重要**：如果每局不逐局上链，随机性就退化为 **Provably Fair（§6.13.5）**：server_seed 承诺 + client_seed + nonce，局后揭示可复算，并把每批承诺锚定到链上。这是可接受的方案，但必须显式告知玩家，不要宣传成"链上 VRF 可验证"——那会是虚假宣传。

#### 6.14.5 三种架构的最终对比

| 维度 | 账本模式（§6.13） | **Passkey 智能钱包 + 会话密钥（本节）** | 链上 + 外部钱包（§6） |
| --- | --- | --- | --- |
| 用户门槛 | 最低（邮箱） | **最低（邮箱 + Face ID）** | 高（装 Phantom、助记词） |
| 单局延迟 | < 50ms | < 50ms（session key 代签） | 0.8–1.5s |
| 单局成本 | ~0 | ~0（session key 代签，可批量上链） | ~0.002 SOL |
| 资金控制权 | 平台 | **用户本人** | 用户本人 |
| 资金可见性 | 无 | **链上可查** | 链上可查 |
| 结果可信度 | Provably Fair | Provably Fair（可升级为定期上链） | 每局链上 VRF |
| 被黑损失上限 | 全部 | **session 额度内** | 合约漏洞 |
| 合规定位 | 托管，必须持牌 | **非托管（评级显著优于 L0/L1）** | 非托管 |
| 工程成本 | 中 | **高**（passkey + 权限矩阵 + relayer，约 +2 周） | 中 |

#### 6.14.6 选型建议

- **只想解决"用户嫌装钱包麻烦"** → 三者都能解决，账本模式最省。但你要放弃非托管叙事。
- **想同时要"零门槛"和"资金不被平台控制"** → **Passkey 智能钱包是当前唯一正解**，推荐作为目标架构，但**不要放进 MVP**。
- **落地顺序建议**：
  1. MVP 用账本模式或 §6 链上模式跑通玩法与留存；
  2. 第二阶段叠加 Passkey 智能钱包（可复用同一套 PDA 金库合约，把 `authorize_session` 与 `session_key` 加进指令集）；
  3. 第三阶段再评估逐局上链（VRF）作为"高额度 Pro 档"。
- **自研 vs 用现成方案**：建议自研权限矩阵（我们的游戏程序本来就要写，且不想把资金安全性外包给第三方程序的存续与升级），仅在客户端 passkey 交互与 relayer 上参考开源实现。

---

## 7. 风控

### 7.1 限注与敞口

```
top_multiplier      = 5000          # 钻石三连
max_exposure_bps    = 500           # 单局最大赔付 ≤ 金库的 5%
max_bet_dynamic     = house_vault_balance * max_exposure_bps / 10000 / top_multiplier
max_bet             = min(config.max_bet_hard, max_bet_dynamic)
```

例：金库 1,000,000 代币 → `max_bet = 1,000,000 × 0.05 / 5000 = 10 代币`。金库增长则档位自动解锁。

### 7.2 金库监控

- 金库低于 `1000 × 平均单注` 时，链上自动降级 `max_bet`；前端顶部显示"金库健康度"。
- 每日快照：金库余额、未结算 Round 数、净流水、实测 RTP（滚动 100 万局）。

### 7.3 反滥用

| 风险 | 对策 |
| --- | --- |
| 机器人批量刷量 | 单地址每分钟流水上限；可选轻 KYC / 人机验证 |
| 选择性结算 | expire_round 惩罚 + crank 主动结算 |
| 大注冲击 | 动态 max_bet（§7.1） |
| 验证者/预言机串谋 | VRF 密码学保证；单局敞口上限使攻击收益 < 成本 |
| 前端钓鱼 | 开源前端 + 合约地址在 UI 显著位置；提供"仅用 CLI 校验"的验证页 |

### 7.4 累进 Jackpot（V2）

- 每笔流水的 1% 进入 Jackpot 池（独立 PDA），钻石三连时触发池子全额或部分发放。
- 需重新校准：基础 RTP 降至 95%，Jackpot 贡献 1%，总 RTP 仍 96%。
- Jackpot 池单独核算，不计入 house bankroll 风险敞口。

---

## 8. UI / UX

### 8.1 主界面

```
┌──────────────────────────────────────────┐
│  $SPIN SLOT           余额 1,250  [充值]  │
│  ┌─────────┬─────────┬─────────┐         │
│  │   🍋    │   🔔    │   🍉    │  Row1   │
│  │ ══🍒════╪═══🍒════╪═══🍒════╪═ PAYLINE│  ← 中间行金色高亮线
│  │   🍊    │   🍉    │   🍋    │  Row3   │
│  └─────────┴─────────┴─────────┘         │
│   注额 [1][5][25][100][500]   下注 25    │
│         [        SPIN        ]           │
│  RTP 96% · 上次赢得 0 · [开奖记录]       │
└──────────────────────────────────────────┘
```

### 8.2 Cocos 工程实现

```
SlotScene
├─ CanvasReels ── ReelController（3 × ReelColumn）
│                 └─ SymbolCellPool（对象池，Sprite + Tween）
├─ PaylineLayer     ← 中间行金色横线 + 中奖时连线高亮
├─ HudPanel         ← 余额 / 注额档位 / SPIN 按钮 / 金库健康度
├─ NetBridge (TS)   ← HttpClient + WsClient（纯 TS，不引 solana 依赖）
├─ WalletBridge     ← 调用 WebView 签名页 / 原生桥（§6.5 方案 A/B/C）
└─ AudioMgr         ← 滚动 · 停止 · 中奖 · 大奖 音效
```

实现要点：

- **对象池 + Mask 裁剪**：每列 5~7 个 symbol 节点循环复用即可实现无限滚动，避免逐个创建节点；用 Mask 组件裁一列窗口。
- **运动模糊**：滚动期用 Cocos 材质/shader 做纵向拉伸模糊，或直接用 Y 轴 scale 压缩近似，成本最低。
- **数据与表现分离**：9 格数据来自 WS 的 `window` 字段（列优先，含上中下三行），落点已由链上确定；**Cocos 侧不做任何符号生成或赔付计算**。
- **60fps 预算**：场景 DrawCall 控制在 ~20 以内；符号图集合并（TexturePacker / Cocos Auto Atlas）。
- **弱网容错**：WS 心跳 + 自动重连；`round.pending` 超过 5s 未收 `settled` 则轮询 `/v1/rounds` 补拉，UI 显示"结果确认中"，绝不本地编造结果。

### 8.3 动画规格

| 阶段 | 时长 | 表现 |
| --- | --- | --- |
| 启动 | 0.1s | 拉杆下拉 + 咔哒音 |
| 滚动 | 0.6 / 0.8 / 1.0s | 三列依次启动，向下循环滚动带运动模糊 |
| 停止 | 每列 0.15s 回弹 | 停止音（音高逐列升高） |
| Near-miss | 0.4s | 第三列停在差一格时抖动 + 悬停 |
| 中奖 | 0.8s | payline 三格脉冲发光 + 金色连线 + 赔付数字滚动计数 |
| 大奖 | 2.5s | 全屏粒子 + 音阶上行 + 分享卡片 |

### 8.4 体验要点

- **一次签名一局**：下注只签一次，settle 由 Go crank 代付，reveal 等待期显示"结果确认中"并被滚动动画自然覆盖。
- **余额先行**：链上 credit 由 Go indexer 实时推送 `balance.changed`，UI 即刻反映。
- **信任面板**：每局可展开查看 `Round PDA`、`randomness pubkey`、随机数原始字节、符号推导过程 —— 这是链上 slot 相对传统线上赌场的核心卖点。
- **可验证入口**：面板里直接给 explorer 链接 + 一份"自己复算"的说明（strip hash + 3 个随机数 → 符号），任何人可离线核对。
- **降级路径**：Go 后端不可用时，UI 提示"可直接向链上发起提现"并给出 fallback 页面链接，兑现非托管承诺。

---

## 9. 数据与运营

| 指标 | 定义 | 目标 |
| --- | --- | --- |
| 流水 Volume | 日均下注代币量 | 上线 30 天 ≥ 金库 3× |
| 实测 RTP | 滚动统计赔付/流水 | 长期收敛到 96% ± 0.5% |
| Spin/User/DAU | 人均日旋转次数 | ≥ 40 |
| 充值转化 | 连接钱包 → 首次充值 | ≥ 25% |
| 留存 D7 | — | ≥ 20% |
| 金库健康度 | 金库 / (1000 × 平均注) | ≥ 1.0 |

运营动作：新用户首充 bonus（需计入营销成本，不改 RTP）、流水排行榜（周赛）、质押者分红公示（每周）、回购销毁公告（链上可查链接）。

---

## 10. 路线图

| 里程碑 | 周期 | 交付物 |
| --- | --- | --- |
| **M0 数学** | 1 周 | strip 生成器 + 蒙特卡洛 1e8 验证脚本 + `shared/paytable.json` 定稿 |
| **M1 合约** | 2 周 | Anchor 程序 v1（deposit/spin/settle/withdraw），litesvm 单测覆盖结算逻辑与赔率表 |
| **M2 随机源** | 1 周 | 接入 Switchboard VRF，完成不可预测性 / 重放防护测试 |
| **M3 Go 后端** | 2 周 | IDL→Go codegen + txfactory + indexer + WS + 基础 API；`solana-test-validator` 集成测试跑通全流程 |
| **M4 Crank 与风控** | 1 周 | 自动 settle worker（并发/重试/费用 bump）、`expire_round` 兜底、risk 限频、金库健康度告警 |
| **M5 Cocos 前端** | 2–3 周 | 3×3 卷轴动画（对象池 + Mask）、WalletBridge（先方案 A）、HUD、信任面板、音效 |
| **M5.5 方案 B（可选）** | 1–1.5 周 | esbuild 预打包 wallet-standard SDK + AdapterRouter 自动降级 + TxInspector 盲签校验与恶意交易测试（§6.5） |
| **M6 联调压测** | 1 周 | devnet 端到端压测（并发 spin、crank 吞吐）、CU profiling、Fuzz（Trident）+ Go/rust 逻辑一致性对拍 |
| **M7 审计与发币** | 2–3 周 | 第三方审计 + 代币发行 + LP + Squads 多签接管权限 |
| **M8 上线运营** | 持续 | Web/PWA 首发 → Android；质押/回购；V2（WILD / SCATTER / Jackpot） |
| **M9 零门槛升级（可选）** | 2–3 周 | Passkey 智能钱包 + 会话密钥（§6.14）：`authorize_session` 权限矩阵、passkey 验签、relayer 免 gas、账户恢复设计 |

> **建议的技术试错**：M1 开始前先做 2 天 spike —— 用 Go 对一个最简单的 Anchor 程序做"构造指令 → 发交易 → 解析账户数据"，把 discriminator 与序列化方案一次性验证掉，避免 M3 翻车。

---

## 11. V2 玩法扩展（预留）

1. **WILD（钻石变 Wild）**：替代任意符号成三连；需重新用蒙特卡洛校准 RTP（Wild 会显著抬高 RTP，通常需下调低符号赔付）。
2. **SCATTER（幸运星）**：中间行任意位置出现 3 个 scatter 触发 10 次免费旋转（同注额）；免费旋转同样走链上随机源，注意 CU 与账户复用。
3. **Gamble / Double-or-Nothing**：中奖后可单次加倍（红色/黑色 50/50），链上实现简单，但会拉高波动率，金库需额外计提。
4. **累进 Jackpot**：见 §7.4。
5. **会话密钥 / Ephemeral Rollup**：高频旋转场景可评估 MagicBlock 或委托会话，降低确认延迟到毫秒级（会引入额外信任假设，需单独论证）。

---

## 12. 风险与合规

| 类别 | 说明 |
| --- | --- |
| **法律** | 链上赌博在多数司法辖区受监管（美国各州、欧盟、部分亚洲地区）。上线前必须做**地理围栏 + IP 封锁 + ToS 声明**，并在 UI 强制 18+ 提示。本策划案不提供法律意见，正式上线需当地法务确认。 |
| **发布渠道** | 微信/抖音等小游戏平台禁止加密货币支付，**不可作为渠道**；iOS App Store 对赌博类 App 审核极严。主推 Web/PWA，Android 次之（§6.5）。 |
| **后端中心化** | Go 后端虽不托管资金，但仍承担可用性：crank 停摆会导致玩家等待超时被 `expire_round`（虽 punish 极少发生）。需 crank 多活 + 监控告警 + 任何人可 settle 的兜底。 |
| **托管风险（§6.13 架构下）** | 账本模式下服务器持有用户资金：必须持牌 + KYC/AML；热钱包限额、冷仓多签、自动对账止损、储备金证明（PoR）缺一不可；ToS 必须写明提现通道与资金归属。 |
| **智能合约** | 未审计合约不得承载真实资金；建议至少 1 家审计 + 漏洞赏金。 |
| **代币** | 代币价格波动风险（§5.1）；发币涉及证券法风险，分红设计需谨慎。 |
| **预言机** | 随机源是单点依赖；需准备 fallback（暂停 + 已下注局退回）。 |
| **流动性** | 代币 LP 深度不足会导致回购/卖出滑点剧增；初期需做市。 |
| **舆论** | Meme 化叙事与赌博结合需谨慎表述，避免" guaranteed returns"类宣传。 |

---

## 附录 A：关键参数速查

```
N (strip 长度)        = 1000
符号数                = 9
Payline               = 中间行（Row 2）
结算规则              = 三连 / 恰好两连 / 无
理论 RTP              = 95.95%
House Edge            = 4.05%
Hit Frequency         = 42.32%
σ / bet               = 4.44
Top Prize             = 5000× (DIAMOND ×3)
max_exposure_bps      = 500 (5% of bankroll)
建议 Bankroll         ≥ 1000 × 平均单注
随机源                = Switchboard Randomness (VRF)
结算模式              = request_spin → (Go crank 代付) settle_spin
超时惩罚              = expire_round，注额充公
```

**技术栈速查**

```
前端    Cocos Creator 3.x / TypeScript / 发布 Web·iOS·Android
后端    Go 1.23+ / gin / gorilla-ws（或 nhooyr）/ Postgres / Redis
链上    Rust + Anchor 0.30+ / SPL Token / Switchboard VRF
Go SDK  github.com/solana-foundation/solana-go
通信    Cocos ⇄ Go: REST + WebSocket；Go ⇄ Solana: JSON-RPC + WS subscriptions
发布渠道  Web/PWA（主）、Android（次）；⚠ 小游戏平台与 iOS 赌博类审核受限
```

## 附录 B：待确认决策点

1. 筹码计价：SOL / USDC / $SPIN 单代币（§5 方案 A vs B）—— **决定后续所有经济模型**
2. 目标 RTP：96% 是否合适（竞品通常 94–97%，高 RTP 意味着更薄的利润）
3. 是否上线即发币，还是先跑无币版本验证玩法
4. 随机源预算：每局 ~0.002 SOL 的 VRF 成本是否可接受（可用小额模式走 slot-hash 降低成本）
5. 是否需要 jackpot 池与质押分红在 MVP 就上线
6. Cocos 的钱包桥先走哪条：**A（WebView，全端通用，MVP 必做）**、B（H5 直连 wallet-standard，需接受自己承担盲签安全，§6.5）、还是两者都做并用 AdapterRouter 自动降级（推荐）
7. 首发渠道：只做 Web/PWA，还是同步 Android（涉及额外的双端原生工作量）
8. 是否接受"每局弹一次钱包"的方案 B 体验；若要消除，是否投入 V2 会话密钥（§6.5.9，约增加 1 周合约工作量）
9. **【最优先】采用哪种资金架构**（四选一，决定合规路径、工期与产品定位，其他决策都排在它后面）：
   - a. 链上非托管 + 外部钱包（§6）—— 门槛高、叙事最强
   - b. 中央账本 + 批量上链（§6.13）—— 门槛最低、**定性为托管，须持牌**
   - c. **Passkey 智能钱包 + 会话密钥（§6.14）—— 零门槛 + 非托管，推荐为目标架构**
   - d. 双轨（账本快速档 + 链上 Pro 档）
10. 若选 c：是否接受约 +2 周工程成本；设备丢失的账户恢复设计（第二 passkey / 社交恢复）是否在 MVP 内
11. 若每局不上链：是否接受把随机性降级为 **Provably Fair** 并在 UI 如实说明（不可宣传为链上 VRF）
