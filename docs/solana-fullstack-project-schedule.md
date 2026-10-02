# $SPIN Solana 链上老虎机——单人全栈项目排期

> 版本：v2.0  
> 制定日期：2026-10-01  
> 计划启动日期：2026-10-05  
> 项目根目录：/Users/wuruofan/SolanaSlot  
> 配套文档：[Solana 链上老虎机策划案](./Solana链上老虎机策划案.md) · [Solana 老虎机技术实现文档](./Solana老虎机-技术实现文档.md)

## 1. 排期结论

本项目当前处于“设计基本完成、工程尚未启动”阶段。目录内已有策划案和技术实现文档，但尚无 Cocos、Go、Rust/Anchor 工程，因此前端、后端和链上均需从零建设。

推荐开发顺序：

1. 冻结资金架构、筹码、随机源及 MVP 边界；
2. 完成数学验证和三端共享数据；
3. 学习 Rust/Solana，并验证 Anchor、Go SDK 和 Switchboard；
4. 优先实现链上程序，因为账户、指令和事件是另外两端的上游契约；
5. 实现 Go 交易工厂、Indexer、Crank、风控和 WebSocket；
6. 实现 Cocos 功能前端及钱包；
7. 完成 Devnet 压测、安全加固和审计。

按“单人全职、从 Rust/Solana 基础学起、首版只做一款三列老虎机”估算：

| 目标 | 预计工期 | 预计完成时间 |
| --- | ---: | --- |
| 本地纵向闭环 | 16～20 周 | 2027 年 1～2 月 |
| Devnet 功能 MVP | 28～34 周 | 2027 年 4～5 月 |
| Devnet 公开 Beta | 34～40 周 | 2027 年 6～7 月 |
| Mainnet Candidate | 40～48 周 | 2027 年 7～9 月 |

如果每周投入 15～20 小时，建议按 16～24 个月规划。第三方审计的排队时间不完全受开发者控制，应另留 4～8 周日历时间。

## 2. 估算基线

本排期以技术实现文档 v1.0 为主，并采用以下范围：

- 前端：Cocos Creator 3.x + TypeScript，首发 Web/PWA；
- 后端：Go 1.23+ 模块化单体；
- 链上：Rust + Anchor，负责资金、随机数、符号和赔付；
- 资产：原生 SOL（lamports），$SPIN 暂只保留权益层设计；
- 随机源：Switchboard Randomness；
- 游戏：仅 classic_fruits 三列老虎机；
- 钱包：外部钱包；
- 数据：PostgreSQL + Redis；
- 发布：Localnet → Devnet → 审计后再考虑 Mainnet；
- 多游戏平台只保留必要扩展点，不在 MVP 开发第二款游戏；
- Passkey、会话密钥、质押、回购、发币、Jackpot 和原生 App 不进入 MVP。

范围变化对工期的影响：

| 变化 | 额外工期参考 |
| --- | ---: |
| 同步发布 iOS/Android | 4～8 周 |
| Wallet Standard 直连及完整降级链 | 2～3 周 |
| Passkey 智能钱包 + 会话密钥 | 4～8 周 |
| $SPIN 发币、质押、分红和回购 | 6～10 周，不含法律工作 |
| 第二款游戏 | 4～8 周，另需增量审计 |
| 中央账本 + 批量上链 | 4～8 周，并提高托管与合规成本 |

## 3. 当前状态

### 3.1 已完成

- 产品定位、玩法和赔率表草案；
- 目标 RTP 及基础金库风险模型；
- Cocos、Go、Anchor 三层职责划分；
- 链上账户、指令和多游戏扩展设计；
- Switchboard 随机源及两阶段结算设计；
- Cocos 钱包桥、交易检查器和状态机设计；
- Go TxFactory、Indexer、Crank、风控和 WS 模块设计；
- 测试分层、监控及发布流程草案。

### 3.2 尚未完成

- Rust/Anchor 工程及链上程序；
- Go 后端、数据库迁移和运行环境；
- Cocos 工程、资源、动画及钱包接入；
- 数学脚本、共享赔率数据和跨语言代码生成；
- Switchboard 的真实费用、延迟及失败行为验证；
- Localnet/Devnet 集成测试、CI/CD 和监控；
- 安全审计、Mainnet 部署及合规确认。

策划案中原有约 12～15 周的路线图，可视为熟悉全套技术栈的多人团队理想工期，不适合作为单人学习型开发排期。

## 4. 开发原则

### 4.1 先链上，再后端，再正式前端

链上 Account、PDA、Instruction、IDL 和事件结构决定后端如何构造交易，也决定前端需要签什么、展示什么。如果先完成正式前端或完整后端，合约修改会导致两端一起返工。

前期可以做低保真界面和卷轴动画实验，但不应在链上接口冻结前投入大量正式 UI 工作。

### 4.2 尽早打通纵向闭环

第一条闭环：

连接钱包 → SIWS 登录 → deposit → request_spin → Switchboard reveal → Go Crank 调用 settle_spin → 链上结算 → WS 推送 → Cocos 停轮 → withdraw。

每完成一层就为整条链路补测试，不等三端全部完成才联调。

### 4.3 MVP 只保留一个技术答案

本排期默认采用：

- 原生 SOL；
- 链上非托管；
- Switchboard Randomness；
- 外部钱包；
- Web/PWA；
- 一款游戏；
- 不发行 $SPIN。

## 5. 总体里程碑

以下日期按 40 周 Devnet Beta 基线制定：

| 里程碑 | 日期 | 交付物 |
| --- | --- | --- |
| M0：范围与数学冻结 | 2026-10-16 | ADR、赔率数据、数学验证报告 |
| M1：基础学习与风险 PoC | 2026-11-20 | Anchor、Go 客户端、钱包及随机源 PoC |
| M2：链上核心程序 | 2027-01-01 | Deposit、Spin、Settle、Withdraw 本地测试 |
| M3：Switchboard 完整接入 | 2027-01-29 | 随机、重放防护及超时路径 |
| M4：Go 后端闭环 | 2027-03-12 | TxFactory、Indexer、Crank、WS、数据库 |
| M5：Cocos 功能 MVP | 2027-04-30 | 钱包、下注、卷轴、结果及历史 |
| M6：Devnet MVP | 2027-05-28 | 三端端到端测试和部署 |
| M7：Devnet Beta | 2027-07-09 | 压测、安全加固和审计材料 |
| M8：Mainnet Candidate | 2027-08-20 | 审计修复、多签和灰度准备 |

## 6. 分阶段排期

### 阶段 0：决策冻结与数学验证

时间：2026-10-05 ～ 2026-10-16，共 2 周。

详细实现规范：[阶段 0 详细实现 Spec](./phase-0-detailed-implementation-spec.md)。

主要工作：

- 确认原生 SOL、链上非托管及 Web/PWA；
- 冻结玩法、下注档位、最大赔付和 RTP；
- 实现 strip 生成器、解析计算和蒙特卡洛验证；
- 生成 shared/paytable.json、strip、版本及哈希；
- 实测 Switchboard 单局成本和延迟；
- 记录资金、随机、结算、超时和提现 ADR；
- 建立 monorepo、CI 和开发环境说明。

退出标准：

- 解析 RTP 与蒙特卡洛误差不超过 0.05%；
- Rust、Go、Cocos 可以消费同一份共享数据；
- MVP 不再同时保留 SOL、USDC、$SPIN 多种筹码选项；
- VRF 成本与下注档位具有经济可行性。

### 阶段 1：Rust/Solana 学习与风险 PoC

时间：2026-10-19 ～ 2026-11-20，共 5 周。

学习及 PoC：

- Rust 所有权、借用、生命周期、Trait、错误处理和测试；
- Solana Account、Instruction、Transaction、PDA、CPI 和租金；
- Anchor 账户约束、事件、错误码和 IDL；
- Local validator、Devnet 部署和交易调试；
- Go 根据 IDL 构造指令、发送交易并解析账户；
- TypeScript 签署后端构造的测试交易；
- Switchboard 请求、揭示、消费及失败路径；
- 锁定 Rust、Solana CLI、Anchor、Go、Node 和 Cocos 版本。

退出标准：

- Anchor、Go SDK、钱包及 Switchboard 均有可重复运行的 PoC；
- discriminator、Borsh 编码及错误码方案确定；
- 根据实测决定是否保留当前随机源方案。

### 阶段 2：链上核心程序

时间：2026-11-23 ～ 2027-01-01，共 6 周。

主要工作：

- 实现 PlatformConfig、GameRegistry、GameAssets、PlayerState、Round；
- 实现初始化、游戏注册、参数更新及暂停；
- 实现 deposit、request_spin、settle_spin、expire_round、withdraw；
- 实现动态限注、金库暴露控制及 rent 保护；
- 实现 classic_fruits 纯函数 evaluate；
- 将赔率、strip 和版本哈希绑定到链上配置；
- 定义稳定事件供 Go Indexer 使用；
- 测试越权、溢出、重复结算、错误账户及最大赔付。

退出标准：

- Localnet 跑通充值、下注、结算和提现；
- 后端无法决定符号或赔付；
- 玩家可以绕过后端直接提现；
- 所有资金移动均由链上不变量保护；
- IDL v0.1 冻结。

### 阶段 3：随机源与结算安全

时间：2027-01-04 ～ 2027-01-29，共 4 周。

主要工作：

- 正式接入 Switchboard Randomness；
- 完成 randomness 账户创建、commit、reveal 及消费；
- 防止旧随机数、跨 Round 随机数和重复 reveal；
- 实现 permissionless settle；
- 明确超时局的退款或惩罚规则；
- 测试预言机延迟、不可用及交易过期；
- 完成 Compute Unit 和交易大小 profiling；
- 设计随机源不可用时的暂停及恢复。

退出标准：

- Devnet 连续完成至少 1,000 局自动结算；
- 任何人不能看到结果后选择性取消下注；
- 随机源故障不会使资金永久锁定；
- 每局成本和延迟满足 MVP 指标。

### 阶段 4：Go 后端

时间：2027-02-01 ～ 2027-03-12，共 6 周。

模块范围：

- API：认证、状态、游戏配置、历史和交易；
- TxFactory：v0 交易、ComputeBudget 及 blockhash；
- Submitter：提交已签名交易和错误归类；
- Indexer：日志订阅、账户订阅、断点续扫及幂等落库；
- Crank：并发 settle、重试、费用 bump 和多实例互斥；
- Risk：限频、软限注和金库健康度；
- WS：pending、settled、余额和风险事件；
- Admin：暂停、配置及只读运营查询；
- Codegen：IDL → Go 指令、账户和错误码。

基础设施：

- PostgreSQL migration；
- Redis 限频、去重和短期状态；
- Local validator 集成测试；
- 指标、结构化日志、链路 ID 和健康检查；
- Crank 密钥与金库 authority 隔离。

退出标准：

- 服务重启后 Indexer 能从游标继续；
- 重复日志或提交不会产生重复记录和结算；
- Crank 停止后第三方仍可 settle；
- Go 与 Rust 对至少 10 万组输入逐字段一致；
- REST 和 WS 契约冻结为 v0.1。

### 阶段 5：Cocos 功能 MVP

时间：2027-03-15 ～ 2027-04-30，共 7 周。

主要工作：

- 建立大厅、Core Bundle 和 classic_fruits Asset Bundle；
- 实现 GameSdk、ISlotGame、NetBridge 和 WalletBridge；
- 完成钱包连接、SIWS、充值、下注及提现；
- 完成三列卷轴、对象池、Mask、中奖线及基础音效；
- 使用 WS pending/settled 驱动动画；
- 实现断线重连、Round 补拉及刷新恢复；
- 实现交易预览、网络检测、错误提示及加载状态；
- 实现 TxInspector 白名单校验；
- 实现历史、信任面板及 Round 复算；
- 完成桌面和移动浏览器适配。

退出标准：

- 用户能从空钱包完成充值、旋转、查看结果和提现；
- 前端不生成符号、不决定最终赔付；
- 恶意交易、未知程序及越权转账在签名前被拒绝；
- WS 断开、拒签和 blockhash 过期均可恢复；
- 主要游戏场景稳定达到 60 FPS。

### 阶段 6：Devnet 联调与功能 MVP

时间：2027-05-03 ～ 2027-05-28，共 4 周。

主要工作：

- 三端 Devnet 全链路回归；
- 并发 Spin、Crank 吞吐和 RPC 限流测试；
- 三端共享版本及结果一致性检查；
- 注入 RPC、Redis、PostgreSQL、WS 和 Crank 故障；
- 测试钱包切换、重复签名、延迟确认及过期交易；
- 优化 CU、交易大小、优先费及端到端延迟；
- 建立 Devnet 自动部署和回滚。

退出标准：

- 核心场景无 P0/P1 缺陷；
- 连续 10,000 局无资金或结果不一致；
- 服务重启和 RPC 切换不会造成重复结算；
- Devnet 可以一键部署并完成冒烟测试。

### 阶段 7：Beta、安全与运维

时间：2027-05-31 ～ 2027-07-09，共 6 周。

主要工作：

- Trident/Fuzz 和链上攻击路径测试；
- TxInspector 恶意样本测试；
- 金库暴露、动态限注及极端赔付压力测试；
- 多 RPC、多 Crank 及告警演练；
- 建立指标面板、Pager 告警和运行手册；
- 管理员权限转移至 Squads 多签；
- 冻结审计 commit 并整理威胁模型；
- 邀请受控用户进行 Devnet Beta；
- 修复 Beta 反馈及审计前自查问题。

退出标准：

- Beta 环境连续稳定运行至少 14 天；
- 权限、密钥、暂停及升级均有多签方案；
- 所有异常 Round 都能监控、定位及恢复；
- 审计材料完整，代码进入冻结状态。

### 阶段 8：审计与 Mainnet Candidate

时间：2027-07-12 ～ 2027-08-20，预计 6 周，审计排队时间另计。

主要工作：

- 第三方合约审计、修复及复审；
- Mainnet RPC、优先费和成本验证；
- 设置金库限额、灰度用户及单日流水上限；
- 演练暂停、升级、退款及回滚；
- 完成适用司法辖区的法律与合规评估；
- 仅在审计和合规通过后准备小额灰度。

退出标准：

- 无未关闭的 Critical/High 审计问题；
- 未审计版本不能通过发布流水线；
- Mainnet 资金上限及自动暂停阈值生效；
- 运营、监控及应急责任明确。

## 7. MVP 明确不做

- $SPIN 发币、LP、质押、分红和回购；
- Passkey 智能钱包和会话密钥；
- 中央账本及链上/链下双轨；
- 第二款及更多游戏；
- WILD、SCATTER、免费旋转、Gamble 和 Jackpot；
- iOS/Android 原生桥；
- 第三方游戏开发者接入；
- 自建 Solana RPC；
- 完整运营后台和复杂数据分析。

可以保留扩展点，但不为未经验证的未来需求提前实现完整框架。

## 8. 单人工作节奏

| 工作类型 | 建议比例 |
| --- | ---: |
| 核心开发 | 55% |
| 自动化测试 | 25% |
| 学习、设计及文档 | 10% |
| 缺陷及工期缓冲 | 10% |

执行规则：

- 每两周必须有一次可运行演示；
- 每个里程碑都必须满足退出标准；
- 资金路径代码与测试同步提交；
- 不连续超过四周只开发单层而没有端到端验证；
- 每周留半天处理依赖、环境和文档；
- 以测试和故障恢复通过作为完成标准。

## 9. 关键风险

| 风险 | 影响 | 应对方式 |
| --- | --- | --- |
| Rust/Solana 学习曲线 | 合约延期 | 先做 5 周 PoC |
| Switchboard 成本过高 | 小额下注不经济 | 阶段 0 实测并调整下注或随机方案 |
| VRF/Crank 异常 | Round 未结算 | Permissionless settle、超时路径、多 Crank |
| Go Anchor 客户端兼容 | 编码或解析错误 | IDL codegen + 真链集成测试 |
| Cocos 钱包兼容 | 移动端无法签名 | MVP 冻结一种主路径 |
| 每局钱包确认 | 体验较差 | 先验证产品，V2 再做受限会话密钥 |
| 三端规则漂移 | 展示或对账错误 | 共享数据、版本哈希和跨语言对拍 |
| 单人上下文切换 | 效率降低 | 按阶段推进，每两周纵向集成 |
| 合约漏洞 | 资产损失 | 小额灰度、暂停、多签及审计 |
| 赌博及代币监管 | 无法上线 | Mainnet 前完成法律评估和地域限制 |

内部计划至少保留 20% 缓冲。资金安全、随机公平性及权限问题不能通过压缩测试时间来追赶排期。

## 10. 第一周行动清单

- [ ] 确认技术实现文档 v1.0 为当前主方案；
- [ ] 确认 MVP 使用原生 SOL，不用 $SPIN 作为筹码；
- [ ] 确认只发布 Web/PWA 和一款游戏；
- [ ] 确认链上非托管 + 外部钱包；
- [ ] 实测 Switchboard 单局成本；
- [ ] 冻结下注档位、最大赔付、RTP 和金库暴露参数；
- [ ] 创建 programs/slot、server、client、shared、webwallet 目录；
- [ ] 锁定 Rust、Solana CLI、Anchor、Go、Node 和 Cocos 版本；
- [ ] 实现 strip 生成器和蒙特卡洛脚本；
- [ ] 建立最小 CI 和双周演示节奏。

## 11. Mainnet Candidate 完成定义

- Rust、Go、Cocos 使用同一版本赔率和 strip；
- 玩家资金不能被 Go 后端或 Crank 密钥直接提取；
- 即使 Go 后端不可用，玩家仍能独立提现；
- 所有 Round 都能复算随机数、符号和赔付；
- 重复提交、断线、重启及 RPC 切换不会重复处理资金；
- 合约、后端、前端及配置均可追踪和回滚；
- 管理权限由多签控制，并完成暂停与恢复演练；
- 监控能识别 Crank 积压、金库风险及异常 Round；
- 审计 Critical/High 问题全部关闭；
- 完成适用地区的法律、年龄、地域及负责任博彩要求确认。
