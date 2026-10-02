# 阶段 0 详细实现 Spec：决策冻结、数学验证与工程基线

> 文档版本：v1.0  
> 状态：待执行  
> 阶段周期：2026-10-05 ～ 2026-10-16  
> 负责人：单人全栈开发者  
> 项目根目录：/Users/wuruofan/SolanaSlot  
> 上游文档：[项目排期](./solana-fullstack-project-schedule.md) · [策划案](./Solana链上老虎机策划案.md) · [技术实现文档](./Solana老虎机-技术实现文档.md)

## 1. 目的

阶段 0 不实现生产链上合约、Go 服务或 Cocos 游戏。它负责消除会引起三端返工的基础不确定性，并产生阶段 1 可以直接使用的、可机器验证的工程输入。

阶段完成后必须回答：

1. MVP 用什么资产、托管模式、渠道、钱包路径和随机源；
2. 游戏规则、赔付语义和精确数学指标是什么；
3. Strip、Paytable、风险参数如何生成、版本化和校验；
4. Switchboard 的真实成本和延迟是否支持目标下注档位；
5. Rust、Go、TypeScript 将遵守什么共同数据契约；
6. 开发环境、命令和 CI Gate 是否可重复执行。

## 2. 规范用语与文档优先级

本文中的“必须”“禁止”“应当”“可以”具有规范含义：

- 必须：阶段验收不可缺少；
- 禁止：实现不得出现；
- 应当：除非 ADR 记录充分理由，否则必须执行；
- 可以：可选，不影响阶段验收。

文档冲突时按以下优先级处理：

1. 已批准 ADR；
2. 本 Spec；
3. 技术实现文档 v1.0；
4. 项目排期 v2.0；
5. 策划案 v0.2。

阶段结束时，必须把已确认结论回写到策划案和技术实现文档，不能长期依靠优先级掩盖冲突。

## 3. 范围

### 3.1 In Scope

- MVP 架构与产品决策冻结；
- 经典三列水果机的数学资产；
- 精确解析、组合穷举和蒙特卡洛验证工具；
- 可重现 Strip 生成；
- JSON Schema、二进制资产及 SHA-256 Manifest；
- 随机数到 Reel Position 的无偏映射规范；
- Switchboard Devnet 成本/延迟探针；
- Monorepo 目录骨架；
- 工具链版本锁定；
- 最小 Makefile 和 CI；
- 阶段报告、ADR、风险和移交清单。

### 3.2 Out of Scope

- 生产 Anchor Program；
- 真实 deposit、request_spin、settle_spin、withdraw；
- Go API、Indexer 或 Crank；
- Cocos 正式界面和动画；
- Mainnet 部署；
- $SPIN 发行、质押、分红或回购；
- Passkey、会话密钥和中央账本；
- 第二款游戏；
- 智能合约审计；
- 博彩牌照和正式法律意见。

## 4. 阶段默认方案

以下是执行起点，不代表可以跳过 ADR。阶段 0 结束时必须确认或正式推翻：

| 决策项 | 默认值 |
| --- | --- |
| MVP 游戏 | classic_fruits |
| 游戏结构 | 3 Reel × 3 Row，只有中间行结算 |
| 资产 | 原生 SOL，金额统一使用 lamports 整数 |
| $SPIN | 不进入 MVP 游戏金库 |
| 资金模式 | 链上非托管 |
| 链上拓扑 | 单 Anchor Program + game_id 分派 |
| 随机源 | Switchboard Randomness |
| 前端 | Cocos Creator 3.x，首发 Web/PWA |
| 钱包 | 外部钱包；MVP 以 Web 钱包层为主路径 |
| 后端 | Go 模块化单体 |
| 首发游戏数 | 1 |
| 结算真值 | 链上 |

## 5. 交付物

阶段 0 结束时，仓库至少包含：

~~~text
SolanaSlot/
├─ Makefile
├─ README.md
├─ .gitignore
├─ .tool-versions
├─ docs/
│  ├─ adr/
│  │  ├─ 0001-mvp-scope-and-asset.md
│  │  ├─ 0002-custody-and-program-topology.md
│  │  ├─ 0003-randomness-and-cost-gate.md
│  │  ├─ 0004-wallet-and-release-channel.md
│  │  ├─ 0005-payout-and-round-expiry-semantics.md
│  │  └─ 0006-assets-versioning-and-rng-mapping.md
│  ├─ reports/
│  │  ├─ classic-fruits-math-v1.md
│  │  └─ switchboard-devnet-probe-2026-10.md
│  └─ phase-0-detailed-implementation-spec.md
├─ shared/
│  └─ games/classic_fruits/
│     ├─ game.json
│     ├─ paytable.json
│     ├─ strip.json
│     ├─ strip.bin
│     ├─ manifest.json
│     └─ schema/
│        ├─ game.schema.json
│        ├─ paytable.schema.json
│        ├─ strip.schema.json
│        └─ manifest.schema.json
├─ tools/
│  ├─ math/
│  │  ├─ pyproject.toml
│  │  ├─ generate_assets.py
│  │  ├─ verify_exact.py
│  │  ├─ verify_monte_carlo.py
│  │  └─ tests/
│  └─ vrf-probe/
│     ├─ package.json
│     ├─ pnpm-lock.yaml
│     ├─ src/
│     └─ README.md
├─ programs/slot/
├─ backend/
├─ client/
└─ webauth/
~~~

空目录不得只依赖 .gitkeep；每个未来工程目录应放 README，说明职责、阶段 1 入口和禁止提前实现的范围。

## 6. 数学模型规范

### 6.1 符号定义

Strip 长度固定为 1000。三条 Reel 在 MVP 使用同一份 Strip，但每条 Reel 使用独立随机位置。

| ID | Code | Weight | Pair Multiplier | Triple Multiplier |
| ---: | --- | ---: | ---: | ---: |
| 0 | CHERRY | 200 | 1 | 8 |
| 1 | LEMON | 200 | 1 | 10 |
| 2 | ORANGE | 180 | 1 | 15 |
| 3 | PLUM | 150 | 1 | 20 |
| 4 | WATERMELON | 120 | 2 | 40 |
| 5 | BELL | 90 | 3 | 80 |
| 6 | BAR | 45 | 5 | 250 |
| 7 | SEVEN | 12 | 15 | 1000 |
| 8 | DIAMOND | 3 | 60 | 5000 |

必须满足：

- ID 从 0 连续递增；
- Code 唯一且仅使用大写 ASCII 和下划线；
- Weight 总和严格等于 1000；
- 所有 Multiplier 为非负整数；
- topMultiplier 严格等于 5000；
- 三端禁止维护第二份手写赔率常量。

### 6.2 结果判定

设中间行三个符号为 a、b、c：

1. 若 a = b = c，支付该符号的 Triple Multiplier；
2. 否则，若恰好两个相同，支付重复符号的 Pair Multiplier；
3. 否则支付 0；
4. Triple 不得叠加 Pair；
5. Multiplier 表示 Gross Payout，包含返还的本金；
6. 1× 表示 Push：下注先冻结，结算时原额返还；
7. payoutLamports = betLamports × multiplier，必须做 u64 checked multiplication；
8. 手续费、Oracle 成本和租金不得混入 RTP Payout。

### 6.3 Window 与 Payline

对每条 Reel 的中间位置 pos：

| 行 | Strip Index |
| --- | --- |
| Top | (pos + 999) mod 1000 |
| Middle | pos |
| Bottom | (pos + 1) mod 1000 |

Payline 固定为三条 Reel 的 Middle。

共享结果的 Window 顺序冻结为 Column-major：

~~~text
[R0Top, R0Middle, R0Bottom,
 R1Top, R1Middle, R1Bottom,
 R2Top, R2Middle, R2Bottom]
~~~

任何 Near-miss 只影响前端表现，禁止改变 Pos、Window、Payout 或后续随机数消费。

### 6.4 精确概率

设第 i 个符号权重为 wi，Strip 长度 L = 1000：

- Triple Count(i) = wi³；
- Exact Pair Count(i) = 3 × wi² × (L − wi)；
- Probability = Count / L³；
- RTP Numerator = Σ(Triple Count × Triple Multiplier + Pair Count × Pair Multiplier)；
- RTP Denominator = L³ = 1,000,000,000。

当前表的阶段 0 精确基准：

| 指标 | 精确/基准值 |
| --- | ---: |
| Triple RTP Numerator | 451,064,250 |
| Pair RTP Numerator | 508,475,505 |
| Total RTP Numerator | 959,539,755 |
| RTP | 0.959539755 |
| House Edge | 0.040460245 |
| Hit Frequency | 0.423220240 |
| Push Frequency | 0.329079000 |
| Net-win Frequency | 0.094141240 |
| Lose Frequency | 0.576779760 |
| E[X²] | 20.521055125 |
| Variance | 19.60033858357454 |
| Sigma | 4.427226963187515 |

当前策划案中的 Hit Frequency、E[X²]、Variance 和 Sigma 有轻微偏差。阶段 0 必须使用工具重新生成表格并回写上游文档，禁止复制旧近似值作为测试常量。

### 6.5 两个独立精确验证器

必须实现两条独立路径：

1. Closed-form：按 6.4 公式使用整数或 Fraction 计算；
2. Exhaustive-symbol：遍历 9³ = 729 个符号组合，以 wa × wb × wc 作为组合计数，并调用实际 Payout Rule。

验收条件：

- 两条路径的 RTP Numerator 必须逐整数相等；
- Hit、Push、Net-win、Lose Count 必须逐整数相等；
- 四种 Outcome Frequency 之和必须等于 1；
- RTP + House Edge 必须等于 1；
- 禁止用浮点近似决定精确路径是否通过。

### 6.6 蒙特卡洛验证

蒙特卡洛只作为随机采样和实现回归测试，不是赔率正确性的唯一 Oracle。

| Profile | Spins | 使用场景 |
| --- | ---: | --- |
| smoke | 1,000,000 | 本地快速验证 |
| ci | 10,000,000 | Pull Request / CI |
| release | 100,000,000 | 资产发布前 |

每次输出：

- Seed 和 Spins；
- Sample RTP、Standard Error 和 z-score；
- Hit/Push/Net-win/Lose Frequency 及 z-score；
- 每种 Pair/Triple 命中数；
- 最大单次 Multiplier；
- 运行时间和工具版本。

验收规则：

- abs(rtpZScore) ≤ 5；
- 每个预期命中数 ≥ 25 的事件，abs(eventZScore) ≤ 5；
- 未命中的极低概率事件不得单独判失败；
- 至少使用 3 个固定 CI Seed；
- Release 报告保存完整参数及输出。

禁止把“1 亿局后绝对误差必须 ≤ 0.0001”作为 Gate。以 Sigma 约 4.427 计算，1 亿局 RTP 的单倍标准误约为 0.000443，固定 ±0.0001 会高概率误报。误差必须结合样本数和标准误解释。

## 7. Strip 生成规范

### 7.1 原则

- Strip 必须由公开 Seed 和确定性算法生成；
- 禁止生成后人工交换位置；
- 如需视觉约束，必须先写成算法、ADR 和测试，再升 Assets Version；
- 每个 Symbol ID 的计数必须严格等于 Weight；
- 三端只消费生成后的 Strip，不得自行 Shuffle；
- Seed 不承担运行时随机安全职责，可以公开。

### 7.2 确定性生成

采用 Fisher-Yates Shuffle。初始数组按 Symbol ID 升序展开。

随机字节流：

~~~text
block(counter) =
  SHA256(
    "solanaslot/classic_fruits/strip/v1" ||
    0x00 ||
    seed32 ||
    counter_u64_big_endian
  )
~~~

- seed32 必须是 32 字节公开值；
- counter 从 0 开始；
- 每个 SHA-256 Block 按 8 个 big-endian u32 消费；
- Fisher-Yates 从 i = 999 递减到 1；
- 生成 j ∈ [0, i] 时使用 Rejection Sampling，禁止直接 u32 mod (i + 1)；
- 同一输入在不同机器上必须得到相同 strip.bin。

### 7.3 Strip 验证

必须检查：

- 长度为 1000；
- 所有值在 0～8；
- 每个 ID 计数等于 Weight；
- strip.json 和 strip.bin 内容一致；
- 重复生成得到字节级相同结果；
- Top/Middle/Bottom 在 0 和 999 正确 Wrap；
- 生成邻接统计，披露每个 Symbol 前后邻居分布；
- 不存在未披露的 Near-miss 人工规则。

## 8. 运行时 RNG 映射契约

VRF Bytes 到 Reel Pos 的映射必须在阶段 0 冻结，否则 Rust、Go 和前端复算器可能产生不同结果。

### 8.1 Domain-separated Stream

~~~text
rngBlock(counter) =
  SHA256(
    "solanaslot/spin-rng/v1" ||
    0x00 ||
    gameId16 ||
    roundPda32 ||
    randomness32 ||
    counter_u64_big_endian
  )
~~~

- gameId16 为固定 16 字节；
- classic_fruits 使用 ASCII 后补 0x00；
- 每个 Block 按 8 个 big-endian u32 消费；
- Counter 从 0 开始；
- 可扩展 Stream 允许 Rejection 后继续取值。

### 8.2 Uniform Bounded Integer

~~~text
limit = floor(2^32 / 1000) × 1000 = 4,294,967,000
if value >= limit:
  reject and consume next u32
else:
  pos = value mod 1000
~~~

三条 Reel 依次消费三个成功样本。禁止简单使用 u32 mod 1000，因为 2³² 不能被 1000 整除。

阶段 0 必须产出至少 20 个 Golden Vectors，包含：

- gameId；
- roundPda；
- randomness；
- Counter 消费过程；
- 被 Rejection 的值；
- 三个 Pos；
- 9 格 Window；
- Multiplier；
- 给定 Bet 下的 Payout。

Golden Vectors 后续由 Rust、Go 和 TypeScript 共用。

## 9. 共享资产格式

### 9.1 通用规则

- JSON 使用 UTF-8；
- Key 使用 lowerCamelCase；
- 金额使用十进制字符串或整数，禁止 JSON 浮点金额；
- 派生概率作为十进制字符串写入报告，不进入链上配置源；
- 生成文件固定为 2 空格缩进、Key 排序、LF 和结尾换行；
- Schema 禁止未知字段；
- 所有文件必须由工具生成或验证。

### 9.2 game.json

~~~json
{
  "schemaVersion": 1,
  "gameId": "classic_fruits",
  "gameIdBytesHex": "636c61737369635f6672756974730000",
  "assetsVersion": 1,
  "currency": {
    "kind": "nativeSol",
    "decimals": 9
  },
  "reelCount": 3,
  "visibleRows": 3,
  "paylineRow": 1,
  "stripLength": 1000,
  "payoutSemantics": "grossIncludingStake",
  "topMultiplier": 5000,
  "maxExposureBps": 500,
  "globalOpenExposureBps": null,
  "minBetLamports": null,
  "maxBetHardLamports": null,
  "betTiersLamports": []
}
~~~

阶段开始时风险金额字段可以为 null；阶段退出前必须根据 VRF 和金库报告填入，或由 ADR 明确延期。生产消费者禁止接受 null。

### 9.3 paytable.json

~~~json
{
  "schemaVersion": 1,
  "gameId": "classic_fruits",
  "assetsVersion": 1,
  "symbols": [
    {
      "id": 0,
      "code": "CHERRY",
      "weight": 200,
      "pairMultiplier": 1,
      "tripleMultiplier": 8
    }
  ]
}
~~~

实际文件必须包含全部 9 个符号。

### 9.4 strip.json 与 strip.bin

~~~json
{
  "schemaVersion": 1,
  "gameId": "classic_fruits",
  "assetsVersion": 1,
  "algorithm": "sha256-fisher-yates-rejection-v1",
  "seedHex": "<64 lowercase hex chars>",
  "symbols": [0, 1, 0]
}
~~~

实际 symbols 长度必须为 1000。strip.bin 是相同 Symbol ID 序列的一字节编码，固定为 1000 字节。

### 9.5 manifest.json

~~~json
{
  "schemaVersion": 1,
  "gameId": "classic_fruits",
  "assetsVersion": 1,
  "files": {
    "game.json": {
      "bytes": 0,
      "sha256": "<64 lowercase hex chars>"
    },
    "paytable.json": {
      "bytes": 0,
      "sha256": "<64 lowercase hex chars>"
    },
    "strip.bin": {
      "bytes": 1000,
      "sha256": "<64 lowercase hex chars>"
    },
    "strip.json": {
      "bytes": 0,
      "sha256": "<64 lowercase hex chars>"
    }
  },
  "assetsSha256": "<64 lowercase hex chars>"
}
~~~

聚合 Hash：

~~~text
assetsSha256 = SHA256(
  "solanaslot/assets/v1" || 0x00 ||
  u32be(len(gameBytes))     || gameBytes ||
  u32be(len(paytableBytes)) || paytableBytes ||
  u32be(len(stripBin))      || stripBin
)
~~~

manifest 不参与自身 Hash。Game、Paytable 或 Strip 变化必须提升 assetsVersion，禁止原地覆盖同版本资产。

## 10. 数学工具接口

### 10.1 generate_assets.py

~~~text
python tools/math/generate_assets.py \
  --game classic_fruits \
  --version 1 \
  --seed <64-hex> \
  --out shared/games/classic_fruits
~~~

要求：

- 读取通过 Schema 验证的 Paytable；
- 生成确定性 Strip；
- 写 canonical JSON、strip.bin 和 manifest；
- 默认禁止覆盖已有版本；
- --check 只比较，不修改；
- 错误时非零退出。

### 10.2 verify_exact.py

~~~text
python tools/math/verify_exact.py \
  --assets shared/games/classic_fruits \
  --report docs/reports/classic-fruits-math-v1.md
~~~

要求：

- 校验 Schema 和 Hash；
- 运行 Closed-form 和 729 组合；
- 输出精确 Numerator/Denominator；
- 输出每个 Symbol 的 Pair、Triple、RTP 和二阶矩贡献；
- 输出 Hit、Push、Net-win、Lose；
- 输出 ROR 近似并标明它不是偿付保证；
- 与 6.4 基准不一致时失败。

### 10.3 verify_monte_carlo.py

~~~text
python tools/math/verify_monte_carlo.py \
  --assets shared/games/classic_fruits \
  --spins 10000000 \
  --seed 20261005 \
  --chunk-size 1000000
~~~

要求：

- 分块运行，内存不随 Spins 线性增长；
- 结果可由 Seed 重现；
- 通过 Strip Index 抽样，禁止直接按 Weight 抽 Symbol；
- 使用同一 Payout Rule；
- 输出机器可读 JSON 和人类可读摘要；
- 支持 smoke、ci、release Profile。

## 11. Switchboard Devnet 探针

### 11.1 目的

策划案估算每局成本约 0.001～0.002 SOL。该成本可能使小额 Spin 不具经济性，必须在合约开发前验证。

### 11.2 样本

- 至少 100 次 Devnet Randomness 请求；
- 分布在至少 3 个时间窗口；
- 记录成功、失败、超时和重试；
- 测量账户创建、Commit、Reveal、Close/回收全过程。

### 11.3 单次记录

- Cluster、RPC 和工具版本；
- Transaction Signature；
- Request/Commit/Reveal/Close Slot；
- Confirmed 和 Finalized 延迟；
- Base Fee、Priority Fee、Oracle Fee；
- 创建账户前后余额；
- 可回收和实际回收 Rent；
- 净不可回收成本；
- 交易字节数及 Compute Units；
- 错误码和重试次数。

禁止把最终回收的 Rent 计入长期成本，也禁止忽略玩家需要预先持有的峰值余额。

### 11.4 汇总

- Success Rate；
- Confirmed/Reveal Latency P50/P95/P99；
- Net Cost P50/P95/P99；
- Peak Temporary Balance Requirement；
- Failure Category；
- 每日 10k、100k、1M Spins 成本推算。

### 11.5 成本 Gate

~~~text
costRatio = p95NetPerSpinCostLamports / minBetLamports
~~~

- costRatio ≤ 2%：通过；
- 2% < costRatio ≤ 5%：需要产品和技术 ADR 接受；
- costRatio > 5%：Stop-ship，禁止冻结当前下注档位。

若失败，必须选择：

1. 提高最小下注；
2. 平台补贴并重算单位经济；
3. 更换 Oracle；
4. 为小额档设计不同随机方案并披露信任等级；
5. 取消逐局上链架构。

不能用“后面再优化”作为阶段退出结论。

## 12. 风险与金库报告

报告至少包含：

- RTP、Variance、Sigma；
- 最高赔付 5000×；
- 不同平均 Bet 下的 1%/0.1% ROR 近似；
- ROR 公式适用范围和局限；
- 10k、100k、1M Spins 的期望利润与波动区间；
- 单个头奖、连续头奖和 Pending Exposure 情景；
- maxExposureBps = 500 下的动态 maxBet 示例；
- 建议初始 House Bankroll；
- 建议 minBetLamports、maxBetHardLamports 和 Bet Tiers；
- VRF 成本对 House Edge 与用户成本的影响；
- Devnet/Mainnet 初始灰度限额。

ROR 近似不得描述为资产安全保证。Mainnet 前必须做更完整的路径模拟和压力测试。

## 13. ADR 清单

每份 ADR 包含 Context、Decision、Alternatives、Consequences、Risks、Rollback Trigger 和 Approval。

### ADR-0001：MVP 范围与资产

冻结原生 SOL、不发行 $SPIN、classic_fruits、Web/PWA，以及不含原生 App、Jackpot 和 Feature Game。

### ADR-0002：托管与程序拓扑

冻结链上非托管、单 Program + game_id、玩家独立提现，以及暂停时仍允许 settle 和 withdraw。

### ADR-0003：随机源与成本 Gate

记录 Switchboard 实测、Cost Ratio、最小下注、Oracle 故障恢复及切换条件。

### ADR-0004：钱包与渠道

冻结浏览器/钱包矩阵、Cocos 与钱包边界、每局签名 UX、Wallet Standard 是否进入 MVP，以及不支持环境的提示。

### ADR-0005：赔付与 Round Expiry

冻结 Gross Payout 语义。

建议异常局规则：

- Randomness 未 Reveal 且超过 Oracle Timeout：退还 Bet；
- Randomness 已 Reveal：只能 Permissionless Settle，不允许退款或作废；
- 平台暂停：禁止新 Spin，但允许 Settle、Refund 和 Withdraw；
- 不把纯 Oracle 故障直接定义为玩家 Bet 充公。

如果仍采用 expire_round 充公，ADR 必须提供可验证的玩家过错条件、攻击分析及产品/法律批准。

### ADR-0006：资产版本与 RNG

冻结 gameId16、JSON/Binary、Strip Generator、Aggregate Hash、assetsVersion、VRF 到 Pos 算法及 Golden Vector。

## 14. 工程基线

### 14.1 工具版本

阶段 0 不假定“最新版”兼容。必须验证并锁定：

- Rust stable；
- Solana CLI；
- Anchor/AVM；
- Go；
- Node.js 和 pnpm；
- Python 及数学依赖；
- Cocos Creator。

锁文件使用精确版本，禁止 latest、星号和未锁定范围。README 包含版本检查命令。

### 14.2 Makefile

最少目标：

~~~text
make bootstrap
make format
make lint
make schema-check
make math-generate
make math-check
make math-exact
make math-smoke
make math-ci
make math-release
make vrf-probe
make test
make phase0-check
~~~

phase0-check 只执行确定性和 CI 可完成的 Gate；100M Monte Carlo 和 100 次 Devnet Probe 可独立执行，并以报告验收。

### 14.3 CI Gate

每个 Pull Request 必须：

1. 校验 JSON Schema；
2. 在临时目录重新生成资产并 Byte Compare；
3. 校验 Manifest Hash；
4. 执行两条精确数学路径；
5. 执行 Monte Carlo CI Profile；
6. 执行数学工具单测；
7. 检查生成文件无未提交差异；
8. 检查 ADR、报告和内部链接。

网络探针不作为每次 PR Gate，但阶段 0 结束前必须有一次通过报告。

## 15. 测试用例

### 15.1 Payout Rule

- CHERRY/CHERRY/CHERRY → 8；
- DIAMOND/DIAMOND/DIAMOND → 5000；
- CHERRY/CHERRY/LEMON → 1；
- CHERRY/LEMON/CHERRY → 1；
- LEMON/CHERRY/CHERRY → 1；
- DIAMOND/SEVEN/DIAMOND → 60；
- CHERRY/LEMON/ORANGE → 0；
- Triple 不重复支付 Pair；
- Bet 为 0 被配置层拒绝；
- Bet × 5000 溢出被拒绝。

### 15.2 Strip

- Weight Count；
- 长度和 ID Range；
- 固定 Seed Golden Hash；
- 同 Seed 同结果、不同 Seed 不同结果；
- Rejection Sampling 边界；
- Pos 0/999 的 Window Wrap；
- JSON/Binary 一致；
- 修改任一字节后 Manifest 校验失败。

### 15.3 Exact Math

- 当前表精确 RTP Numerator；
- 729 组合与公式一致；
- 修改 Multiplier 会触发差异；
- Weight Sum 不为 1000 时失败；
- 重复 ID/Code 时失败；
- topMultiplier 不一致时失败。

### 15.4 Monte Carlo

- Seed 可复现；
- Chunk Size 不改变结果；
- Spins 非法值失败；
- 报告字段完整；
- 错误 Payout Rule 能被统计 Gate 发现。

## 16. 两周实施计划

### 第 1 周

| 日期 | 工作 | 当日输出 |
| --- | --- | --- |
| 10-05 | 建目录、文档优先级、ADR 模板 | Repo Skeleton、ADR Template |
| 10-06 | 冻结游戏规则和 Payout | ADR-0001、ADR-0005 Draft |
| 10-07 | JSON Schema、Canonical Encoding、版本 | Schema、ADR-0006 Draft |
| 10-08 | 精确公式和 729 组合 | verify_exact、单测 |
| 10-09 | Strip、Binary 和 Manifest | generate_assets、Golden Hash |

第 1 周演示：从 Paytable 和 Seed 一键生成资产，两个精确验证器输出相同 RTP。

### 第 2 周

| 日期 | 工作 | 当日输出 |
| --- | --- | --- |
| 10-12 | 分块 Monte Carlo 和报告 | verify_monte_carlo |
| 10-13 | 100M Release Run，修正文档 | Math Report、上游文档更新 |
| 10-14 | Switchboard Devnet Probe | Raw Data、Probe Report |
| 10-15 | 成本/风险分析、Bet Tiers 和 ADR | ADR-0002～0004、Risk Report |
| 10-16 | CI、phase0-check、评审和移交 | Phase 0 Release Tag |

第 2 周演示：展示数学资产、Release 报告、VRF 成本与延迟、最终 ADR 和 CI Gate。

日期可以按实际开工日平移，但任务依赖顺序不得颠倒。

## 17. Definition of Done

- [ ] 6 份 ADR 已批准，无影响阶段 1 的 TBD；
- [ ] 资产、托管、渠道、钱包和随机源唯一确定；
- [ ] Paytable、Strip、Manifest 和 Schema 已提交；
- [ ] Strip 可由固定 Seed 字节级重现；
- [ ] Closed-form 与 729 组合的整数指标完全一致；
- [ ] 精确 RTP 为 959,539,755 / 1,000,000,000；
- [ ] 100M Monte Carlo 通过统计 Gate 并保存报告；
- [ ] 上游文档数学偏差已修正；
- [ ] RNG 算法和至少 20 个 Golden Vectors 已冻结；
- [ ] Switchboard 至少 100 次 Devnet 样本已完成；
- [ ] VRF Cost Ratio 已通过或有架构变更 ADR；
- [ ] minBet、maxBetHard、Bet Tiers 和金库建议已填写；
- [ ] 工具链使用精确版本锁定；
- [ ] make phase0-check 在全新环境通过；
- [ ] CI 能发现资产漂移、Hash 变化和数学回归；
- [ ] README 能让另一名开发者在 30 分钟内复现结果；
- [ ] 阶段 1 移交记录完整。

## 18. Stop-ship 条件

- VRF P95 成本超过最小下注 5%，且无替代方案；
- 数学精确路径不一致；
- Strip 无法从公开 Seed 重现；
- Gross/Net Payout 语义仍不明确；
- SOL、USDC、$SPIN 仍同时作为 MVP 候选；
- Oracle 失败时玩家资金规则不明确；
- RNG 使用简单 u32 mod 1000 且未接受偏差；
- 任一端计划维护手写 Paytable；
- 工具链版本未锁定；
- 阶段 1 仍需重定账户、资产或 RNG 基础契约。

## 19. 阶段 1 移交包

1. 已批准 ADR；
2. classic_fruits Assets Version 1；
3. Manifest 和 Aggregate Hash；
4. 精确数学报告；
5. Monte Carlo Release 报告；
6. Switchboard Devnet 报告；
7. 至少 20 个 RNG Golden Vectors；
8. 风险参数和 Bet Tiers；
9. 工具链锁文件；
10. make phase0-check；
11. 已知风险、非 MVP 范围和变更流程。

阶段 1 只能消费这些冻结资产，不得在合约中重新定义权重、赔率或 RNG 映射。如必须修改，应提升 Assets Version、重跑阶段 0 Gate 并追加 ADR。
