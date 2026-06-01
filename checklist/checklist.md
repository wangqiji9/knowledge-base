# Personal Audit Checklist

> 来源：每一条都来自我自己盲审中漏掉或差点漏掉的真实漏洞。不是抄来的，是用教训换来的。
> 更新规则：每次盲审复盘后，把漏掉的漏洞转化为新的检查项加到对应分类中。

---

## 通用检查（所有协议适用）

- [ ] 有暂停函数时，其他函数需要检查暂停状态
- [ ] try/catch是否有捕捉到revert但是静默
- [ ] 协议有暂停/停用/关闭机制时，停用后用户是否仍有路径取回自己的资产？特别检查：通过"标志位解包 owner"的函数（`if (vaults[owner]) owner = IVault(owner).ownerOf()`），标志为 false 时用户还能证明所有权吗？若不能，停用操作会永久锁定用户资产（来源：RevertLend M-10）

### 状态变量修改
- [ ] 赋值语句左右两边的变量名是否正确？（来源：PoolTogether H-01，`yieldFeeBalance -= _yieldFeeBalance` 应为 `-= _shares`）
- [ ] 同一个变量在不同函数中的单位是否一致？（来源：PoolTogether fee记账用asset，出账用share）
- [ ] 函数中是否存在应该用参数但误用了缓存变量的情况？

### 外部可控组件（Hook / Callback）
- [ ] 用户可控的hook/callback在执行时，系统处于什么中间状态？
- [ ] hook执行时msg.sender是谁？攻击者能否利用这个身份做有害操作？
- [ ] hook能否调用scope外的合约来抢跑或干扰当前流程？（来源：PoolTogether M-01，winner在beforeHook中直接去PrizePool领奖，偷走claimer的fee）
- [ ] 恶意hook能否通过revert阻断关键流程（清算、批量操作等）？（来源：RevertLend H-06，见 card15 变体B）
- [ ] 合约内部调用 `safeTransferFrom(ERC721/ERC1155)` 时，之后是否还有状态更新未完成？NFT 接收方可在 `onERC721Received` 回调中重入协议函数，导致同一操作被双重记账（来源：RevertLend H-02，见 card15）
- [ ] `safeTransferFrom(ERC721)` 的接收方可在回调中耗尽 gas 或 revert；若此时函数末尾还有关键状态清理（如删除 positionConfig），则：①本次 execute 失败但清理被回滚，状态永久残留；②机器人 / Keeper 下次仍检测到触发条件并重试；③攻击无限循环，协议持续单方面承担 gas 损耗。额外检查：设置触发条件的门槛是否接近零（攻击几乎免费）？（来源：RevertLend M-02，见 card15 变体C）

### ERC20交互
- [ ] 使用的是transfer还是safeTransfer？transfer在余额不足时是返回bool值需要手动检查？（来源：PoolTogether M-06）
- [ ] 使用 ERC20.approve而不是safeApprove,导致在某些 ERC20 上总是回退，比如usdt因为无法从非0状态改到非0状态，所以推荐写法是先写为0再改。
- [ ] 是否在transfer之前已经修改了状态（burn了share等），导致transfer失败时状态已不可逆？比如transfer没有revert只是false
- [ ] approve是否需要先置零？（某些token如USDT要求）

### 权限与访问控制
- [ ] onlyOwner / onlyAdmin等修饰符是否覆盖了所有敏感函数？（合约级权限）
- [ ] 接受「被操作资产 ID」为参数的函数，是否验证 msg.sender 是该资产的 owner 或被授权者？（资产级权限，来源：RevertLend H-04）
- [ ] 协议按 tokenId 存储用户自定义配置（positionConfig、策略参数等）时，NFT ownership 转移后旧配置是否被自动清除？若不清除：新 owner 可能被前任 owner 的配置影响（来源：RevertLend M-13）；检查方法：搜索 `positionConfigs[tokenId] =` 的写入路径，确认 NFT 转移时是否有对应的删除逻辑
- [ ] 多步骤操作（approve → execute 分两笔 tx）中，execute 函数是否在时间窗口内可被他人抢先调用？需检查 execute 是否验证了 msg.sender 对目标资产的所有权（来源：RevertLend H-04）
- [ ] 权限设置函数本身是否被正确保护？
- [ ] 函数向外部合约转发用户传入的 calldata 时，calldata 内的关键 ID（tokenId、poolId 等）是否也经过所有权验证？函数参数的 ID 通过验证 ≠ data 内的 ID 也合法（来源：RevertLend H-03）

操作资产前是否确认了操作者和接收者的权限。
当转移NFT这种资产时，协议自己的用户权限配置是否同步修改。
多步骤时，操作资产前检查sender的权限。
使用calldata时需要检查实际操纵的仓位和sender之间的权限问题。

### 协议约束覆盖度
- [ ] 协议限制（借款上限、抵押率、费率上限等）是否在所有改变相关状态的操作路径上都重新检查？存入时检查 ≠ 提取时也检查；有检查的单步操作 ≠ 「存+借+取」组合原子交易也受限（来源：RevertLend M-01）

找到所有协议限制（上限、抵押率、健康度），对每一个问：
所有写操作路径都检查了吗？ 存/借/取/清算分别有没有
有没有组合操作入口？（multicall、execute、batch）组合完成后有没有做一次最终状态验证
检查的时机对吗？ 是操作前检查（可能被后续操作破坏）还是操作后检查（更可靠）
最可靠的模式：组合操作末尾统一做一次全局健康检查，而不是依赖每个子操作自己检查。

### Keeper / Operator 模式

- [ ] Keeper/Operator 是否需要先读取链上的用户配置（positionConfig、onlyFees 等）来构造参数（rewardX64 等），再调用 execute()？若是，用户可在 Keeper tx 发出后、执行前抢跑修改配置，使 Keeper 的参数与执行时实际状态不匹配（来源：RevertLend M-23）
- [ ] execute() 是否对 Keeper 传入的参数做了与当前链上状态的一致性验证？若 Keeper 传入 `rewardX64` 但协议不检查它是否与当前 `onlyFees` 匹配，参数失效但 tx 照常执行

Keeper 是链下的自动化机器人，负责监控链上状态，在条件满足时主动发起交易触发合约逻辑。
在 execute() 内部根据当前状态重新计算关键参数，而非完全信任 Keeper 传入的值；或在 execute() 中快照并比较调用前后的关键配置是否一致

### 保护机制与功能触发条件冲突

- [ ] 协议的防操纵/防滑点保护机制（TWAP 偏差检查、价格边界 revert 等）的激活信号，是否与某个核心功能的正当触发场景（止损、清算、紧急提款等）完全重叠？重叠时：极端市场行情下保护机制激活 → 合法操作被拦截 → 用户最需要执行操作时反而无法执行（来源：RevertLend M-18）

保护机制的本意是防止攻击者在自己操纵价格后立刻套利，但是合约无法区分"价格异常是被操纵的"还是"价格异常是真实市场行情"
找到所有价格检查、TWAP 偏差检查、价格边界 require → 追问"真实市场崩盘 / 飙升时，这个检查还会通过吗？" → 若不通过，有没有用户的合法操作需求（止损、清算、紧急提款）被不合理的拦截。评估功能是否在最需要时失效

### 数学与精度
- [ ] 有除法的地方，舍入方向对谁有利？是否符合协议意图？
- [ ] 是否存在可能溢出的累加操作？（来源：PoolTogether M-05/M-07，mint导致totalSupply超出uint96范围）
- [ ] fee/reward的计算在极端输入（0, max, 1 wei）下是否正确？
- [ ] 协议限额（globalLendLimit、借款上限等）是什么单位（assets / shares）？检查比较语句中双方单位是否一致？有 exchangeRate 存在时 shares ≠ assets，需先 `_convertToAssets` 再比较（来源：RevertLend M-12）；快速识别：搜索 `totalSupply()` 与限额的比较，看右边是否是 assets 单位
- [ ] 同一个业务逻辑（如限额检查）在不同函数中有多处实现时（如 `maxDeposit()` 和 `_deposit()`），逐一对比是否完全一致；若一处用了 `convertToAssets` 而另一处没用，必有一处是错的
- [ ] 内部记账变量（totalAssets等）与合约实际token balance是否会因累积舍入误差长期偏离？（#5 Rounding Down Dust Accumulation：share burn多次舍入 → totalAssets内部计数系统性偏低 → 长期insolvency）

舍入方向对谁有利？
Solidity 0.8 的溢出保护只管算术运算，不管类型转换。这一行如果 newBalance > uint96.max，会静默截断，不 revert。使用SafeCast.toUint96(newSupply)来缓解
有没有除零风险。0 输入能否绕过逻。fee 是否向下取整为 0；能否免费重复操作
assert和share的对比是否正确
协议通常有"查询函数"（告知外部能操作多少）和"执行函数"（实际执行），两者都要检查同一个限额，但逻辑不同步
舍入方向都需要对vault有利

---

## ERC4626 Vault

> 参考标准：https://eips.ethereum.org/EIPS/eip-4626
> 核心原则：所有view函数（max*, preview*, convertTo*）必须与对应执行函数的实际行为严格一致。

### 未完全遵循ERC4626标准
始终遵循该标准非常重要，在审计中，任何未完全符合标准实现的函数都可能被视为缺陷，你需要查看标准并核对其中的 MUST 条款

- [ ] `totalAssets`：包括因收益产生的任何复利，必须包含对 Vault 中资产收取的任何费用，不得回退
- [ ] `convertToShares/convertToAssets`：不得包含对金库资产收取的任何费用；不得根据调用者而显示任何差异；在执行实际兑换时不得反映滑点或其他链上条件；不得回退（除输入异常大导致整数溢出）；必须向0方向下取整；应反映”平均用户”而非”每用户”的每股资产价格
- [ ] `maxDeposit`：必须返回不会导致回退的最大可存资产数量（不得依赖 asset.balanceOf）；必须考虑全局和用户特定限制；无限制时返回 2**256-1；不得回退

### 舍入方向（ERC4626明确要求）
- [ ] deposit/mint：舍入是否有利于vault（用户得到更少的shares或需要更多的assets）？
- [ ] withdraw/redeem：舍入是否有利于vault（用户需要burn更多的shares或得到更少的assets）？

### 通货膨胀攻击

#### 根本前提（所有变体共同依赖）
- `totalShares` 极小（空池/近空池）是所有变体的必要条件——totalShares 极小时，少量资金即可大幅拉高 share_price
- 所有防御方案（Virtual Shares / MIN_INITIAL_SHARES / burn to dead）本质都是：确保 totalShares 永远不从极小值开始
- 受害者损失始终来自 **Rounding.Down**（convertToShares 向下取整吃掉差额）；Rounding.Up 不会导致 0 shares

#### 代码层识别（命中任一则深入验证对应变体）
- [ ] 资产用 `balanceOf` 而非存储变量计算？→ 变体A
- [ ] 有内部记账但无强制初始 totalShares 机制？→ 变体B/C/D
- [ ] `convertToShares` 无 virtual shares offset（OZ `_decimalsOffset`）？→ 所有变体
- [ ] 空池保护仅靠 `if (totalAssets == 0) return assets`？→ 可被绕过（仅防绝对空池，card-21）；同时反向问题：坏账清算等操作可能使 totalAssets 真的归零但 totalShares 不归零，此时该路径被错误触发（假空池），新存款人 1:1 铸 shares 无意中承担历史损失（M-14）
- [ ] 无 `MIN_INITIAL_SHARES` 或首次存款强制 burn 到 dead address？→ 高风险

#### 变体A：直接捐赠（balanceOf 无保护）
- [ ] 空池时存入 1 wei + 直接转账拉高 share 价格 → 后续存款者 Rounding.Down 得 0 shares，全部损失
- 触发条件：`totalAssets = balanceOf(this)`，任何人可直接转账操纵

#### 变体B：存储变量 + 状态操纵制造脱锚（变体A 的绕内部记账版本）
- [ ] 有内部记账（直接转账无效），但仍可通过”收益+状态操纵”制造 `1 share / 2 assets` 起始状态
- [ ] 后续受害者存款因 Rounding.Down 得 0 shares，损失全部资产
- 极端放大：攻击者反复存极小金额（每次 Rounding.Down 得 0 shares，等效捐赠），第 n 次存入 2^n wei，n=64 时需 1.8e19 wei

#### 变体C：存储变量 + Rounding.Down + 循环通胀（Sentiment V2 H-3）
- [ ] 制造 `totalShares=1, totalDeposits=D` 起始状态（通过存款→借款→等利息→还款，或提款至剩 2 wei）
- [ ] 循环：存入 `D+1` assets → Rounding.Down 仅得 1 share（应为~2）→ 立即提走 1 share
- [ ] 每次循环：攻击者净损 1 wei，`totalDeposits+1`，`totalShares` 不变 → share 价格线性上涨
- [ ] 受害者存入 `X` assets（`totalAssets < X < 2×totalAssets`）→ 取整为 1 share → 损失率 = (X-totalAssets)/(2X)，最坏情况（totalAssets=2, X=3）损失 33.3%，大 totalAssets 时上界趋近 25%
- 详见漏洞卡片：`card-21-erc4626-share-inflation-rounding-down`

#### 变体D：隐匿捐赠（存储变量 + Rounding.Down → shares = 0）
- [ ] 项目有内部记账，直接转账无法操纵 totalAssets，但可用”微小存款替代捐赠”
- [ ] 存入极小 assets（少于当前 1 share 的价值）→ Rounding.Down 得 0 shares → assets 留在 vault
- [ ] 效果等同直接捐赠：totalAssets 增加但 totalShares 不变 → share 价格上涨
- 与变体C的关系：同一底层机制（Rounding.Down），变体D是 shares=0 的极端情况，变体C是 shares=1（少于应得 2）的循环累积

### 交易所操纵攻击
- [ ] 如上所述，直接将代币转入金库会操纵兑换率，从而提高份额价格。这些份额可以作为抵押品来借贷（超额抵押贷款）。如果份额代币被用作抵押品且份额价格被人为抬高，攻击者可能借出超过其应有权限的价值。在流动性低的金库中曾发生过一些严重的黑客事件，这类金库最容易被通过直接转账（“捐赠”）来操纵。

### 奖励猎人
- [ ] Vault 需要一种向份额持有者分配奖励的方式。在最简单的 vault 形式中，这是通过直接将奖励代币发送到 vault 合约来完成的（是的，与首个存款者攻击的行为相同）。在其他实现中，会有一个专门的函数用于此目的（比如 harvest()、distributeRewards() 等）。这两种方式都会增加 totalAssets，从而提高份额的价值。如果 totalAssets 增加，_convertToAssets 函数将为相同数量的份额返回更多的资产。
- [ ] 奖励时钟从vault部署时就开始计时时：第一个存款人能否拿走空仓期间积累的所有奖励？奖励分发逻辑是否有 `totalShares > 0` 的前提检查？奖励时钟是否应从第一次实际存款才开始计时？（#2 First User Reward Overallocation）

### 资产小数位漏洞
- [ ] 并非所有代币都有 decimals 函数，而且根据 EIP20 实际上也不是必须的，在这种情况下会获取 decimals 失败并默认使用 18 位小数。

### View函数 vs 执行函数一致性
- [ ] maxDeposit() 返回0时，deposit()是否也会revert？反之如果deposit能成功，maxDeposit是否返回正确值？（来源：PoolTogether M-03）
- [ ] maxWithdraw()/maxRedeem() 计算底层 vault 最大可取值时，是否用 `previewRedeem(maxRedeem(this))` 而非 `convertToAssets(maxRedeem(this))`？EIP-4626 的 `convertToAssets` 只是近似值（不含 fee/滑点），可能高估，导致 max* 函数返回过多，违反"MUST NOT return more than actual maximum"（来源：PoolTogether V2 M-02）
- [ ] maxWithdraw() 的计算方式是否和实际 withdraw 使用的底层调用一致？（来源：PoolTogether V5 M-02，一个用 convertToAssets，一个用 maxWithdraw）
- [ ] maxMint() 和 maxRedeem() 同上检查
- [ ] previewDeposit() 返回的shares是否和deposit()实际mint的shares一致？
- [ ] previewWithdraw() 返回的shares是否和withdraw()实际burn的shares一致？
- [ ] convertToAssets() 和 convertToShares() 在所有状态下（正常、亏损、零供应量）是否正确？

### 边界状态
- [ ] totalSupply == 0 时，所有函数是否正常？首次存款的特殊逻辑是否安全？
- [ ] 底层vault亏损时：deposit是否被正确阻止？withdraw是否按正确比例返还？
- [ ] 坏账/大幅亏损场景下：TotalAssets 减少但 TotalShares 不变 → shares/token 比率单向膨胀；多次坏账后，convertToShares 的乘法是否会溢出 uint256 导致协议 DoS？（来源：Sentiment V2 M-9）；主动武器化变体：攻击者**故意**反复触发坏账循环（借款→坏账→再存款），精确控制使 totalShares 接近 uint256.max，之后利息累积触发 feeShares 溢出 revert，使自己在其他池的借款头寸永久不可清算（来源：Sentiment V2 M-23，见 card-26）
- [ ] 坏账清算后，若 totalDepositAssets 被归零但 totalDepositShares 不归零，`totalAssets==0` 的特殊路径会被错误触发（假空池），新存款人按 1:1 铸造 shares，与旧存款人的历史 shares 瓜分资产，无意中承担坏账损失（来源：Sentiment V2 M-14）
- [ ] 坏账摊薄前的提款抢跑（#3 Front-Running Bad Debt）：链上可见的待处理清算 → 用户抢跑撤出 → 剩余用户承担所有损失 → 攻击者以更低share价格重新存入。防御检查：是否有提款延迟/冷却机制？
- [ ] withdraw时asset→share转换若采用Rounding.Up：极端情况下totalShares被减至0而totalAssets仍>0（"死池"），此后convertToShares返回0 → deposit永久失效；可结合前置攻击定向brick目标地址所有池。（来源：Sentiment V2 M-17，见card-24）
- [ ] 底层vault暂停/满额时：上层vault的max*函数是否正确反映？deposit是否会把资金卡住？
- [ ] 底层vault的汇率变化是否会导致上层vault的记账与实际asset不匹配？

### 滑点保护
- [ ] withdraw/redeem是否有滑点保护机制？用户预期取回X asset但实际可能只拿到Y？（来源：PoolTogether M-04）
- [ ] deposit/mint是否有类似保护？

### Fee机制（如果有）
- [ ] fee的记账单位和实际领取时的单位是否一致？（asset vs share）
- [ ] fee的铸造/领取是否会导致totalSupply超出限制？
- [ ] 正常操作流程中（非极端情况），已记账的fee是否一定能被领取？
- [ ] fee领取函数中的状态更新是否正确？（变量名、减去的数量）

### 与底层vault的集成
- [ ] 底层vault的maxDeposit/maxWithdraw限制是否被正确传递到上层？
- [ ] 上层使用的是底层的deposit还是mint？对应的max检查是否匹配？
- [ ] 底层vault如果有fee-on-transfer行为，上层的记账是否正确？
- [ ] 集成的外部vault是否会产生额外的reward token？若有：是否有claim+分发逻辑？reward token列表是否hardcode（无法处理外部vault新增的奖励token）？（#8 Unclaimed Reward Token Accumulation：奖励token永久滞留合约）

### Pool的创建
- [ ] 创建Pool时使用create可以确定一个固定的pool地址，并且创建时如果需要存入asset且销毁一部分时，攻击者可以提前向该地址存入asset从而抬高股份阻止pool的创建（来自sentimentV2）

### 锁定/归属机制（如果有）
- [ ] 锁定计时器是否在每次存款/提款时都被重置？若是：用户可存入1 wei重置计时器，等锁定期过后再存入大额资金，实际规避了锁定限制（#4 Lockup Gaming via Dust Amounts）
- [ ] 每次存款是否创建独立的锁定记录（各自独立计时）？还是全局计时器被任意操作刷新？

---

## ERC20 / Token

### 签名 / Permit 类漏洞

> 使用场景区分：
> **A. 调用外部 token.permit()** — 协议调用用户 ERC20 的 `permit()`，然后 `transferFrom`
> **B. 自实现 permit 函数** — 合约自己写了 `permit()` 供外部调用
> **C. 集成 Uniswap Permit2** — 使用 `permit2.permitTransferFrom` 接受签名授权转账

#### 通用 ECDSA 签名验证（B/C 自实现时必查）
- [ ] 使用 `ECDSA.recover` / `SignatureChecker.isValidSignatureNow`，有s值的检查。而非 raw `ecrecover`，两个s值都可以（防 malleability）【OZ 自动避免】
- [ ] 强制 `s < secp256k1n/2`（low-s）+ `v ∈ {27,28}`【OZ v4.7+/v5.x 已内置】
- [ ] 安全处理 EIP-2098 compact signature（64 bytes）【OZ 自动避免】
- [ ] 合约签名（owner 为合约）必须走 ERC-1271 `isValidSignature`【OZ `SignatureChecker` 部分支持】
- [ ] 签名哈希是否包含 `msg.sender` / 本合约地址（防跨合约滥用）【自定义检查，OZ 不自动】

EIP-2098 紧凑签名v和s
ERC-1271用于可以让合约，多签钱包也能正常验证

#### EIP-712 结构化数据（B/C 适用）
- [ ] `DOMAIN_SEPARATOR` 必须包含 `name`、`version`、`chainId`、`verifyingContract`（防跨链/跨合约重放）【OZ `EIP712` 抽象合约自动计算】
- [ ] 不要硬编码 `DOMAIN_SEPARATOR`（用 `_domainSeparatorV4()` 或缓存+链ID检查）【OZ 自动避免】
- [ ] `_hashTypedDataV4(keccak256(structHash))` 使用正确【OZ 内置】
- [ ] `TYPEHASH` 与 signed struct 完全一致（字段顺序、类型）【手动校验】
- [ ] 跨链部署时未更新 `chainId`（高危）

#### A. 调用外部 token.permit()
- [ ] 是否支持所有 permit 变体？DAI 使用 `nonce` 而非 `deadline`，签名格式不同（来源：PoolTogether M-08）
- [ ] permit + 操作原子性（e.g. permit + swap）：前跑者可抢先消耗签名，使后续操作 revert（来源：Uniswap V4 multicall 场景）
- [ ] 非标准 token 调用 permit 时 fallback 崩溃（参考 Multichain 95万刀事件）

#### B. 自实现 ERC20Permit
- [ ] **强烈推荐**：直接继承 `OpenZeppelin ERC20Permit` 而非手写（99% 的自定义 permit 漏洞出在这里）
- [ ] `deadline` 检查：`block.timestamp <= deadline`（防过期签名重用）【OZ 自动避免】
- [ ] `nonces(owner)` 原子递增（`_useNonce`）（防重放）【OZ 自动处理】
- [ ] `signer == owner` 且 `owner != address(0)`【OZ 自动避免】
- [ ] 成功后必须 `_approve` 并 `emit Approval`【OZ 自动避免】

#### C. 集成 Uniswap Permit2
- [ ] 使用官方固定地址的 Permit2（不可升级、无 owner）
- [ ] 调用 `permitTransferFrom` 前，验证 `permit.permitted.token == expectedAsset`；Permit2 只验证签名合法性，不验证 token 地址，攻击者可用垃圾 token 签名套走真实资产（来源：RevertLend H-01）
- [ ] `permitWitnessTransferFrom`：`witnessTypeString` 必须精确匹配 `encodeType`，否则类型哈希错【手动严格检查】
- [ ] 支持 EIP-1271 合约签名【OZ `SignatureChecker` 部分支持】
- [ ] `invalidateUnorderedNonces` / `lockdown` 是否正确暴露给用户（用户无法撤销时为中危）
- [ ] Permit2 + multicall 组合：防前跑 DoS（攻击者抢先 consume signature，导致用户 tx revert 但 approval 已生效）
- [ ] 使用 `Permit2Lib.transferFrom2` / permit2 fallback 时，nonce 传递正确

### 特殊Token行为
- [ ] fee-on-transfer token是否被考虑？
- [ ] rebasing token是否被考虑？
- [ ] 返回false而非revert的token是否被正确处理？（需要safeTransfer）
- [ ] blacklist token（USDC/USDT）：在清算、批量转账等关键路径上，如果单个 token 的 transfer 因黑名单而 revert，是否会阻断整个操作？批量循环中是否缺少 try/catch 保护？（来源：Sentiment V2 M-12）
- [ ] 批处理中某个操作 revert → 整个批次回退
        ↓
失败后，谁受损？
  ├─ 只有发起者自己 → 通常不是漏洞（原子性保护）
  └─ 协议/其他用户 → 追问：
           ├─ 攻击者能否故意触发这个失败？ → 是 → DoS/漏洞
           └─ 只是意外失败，可以重试 → 可能只是设计缺陷

---

## ERC6909

- [ ] 每个ERC-6909 兼容合约除了必须实现以下接口外，还必须实现 ERC-165 接口


---

## 借贷协议

### 仓位生命周期

- [ ] 新创建实体的初始状态（debtShares=0、debt=0 等计数器归零）是否与"完全完成/销毁"的终止条件相同？若是，任何人可用零值操作（amount=0）提前触发清理逻辑，驱逐刚创建的实体；识别方法：找到触发"完成"的判断（如 `if (currentShares == shares)`），追问：amount=0 时等号是否成立？（来源：RevertLend M-15）
- [ ] 修改状态的函数（repay、burn、close 等）是否过滤了零值输入（`require(amount > 0)`）？遗漏时，零值操作在无债务实体上可能触发"完全完成"路径并执行清理

### 精确金额校验 + 第三方可改变状态 = 前置 DoS

- [ ] repay/liquidate 等函数是否对"amount 超过当前债务"采用严格 revert（而非截断至 min）？若是，任何可提前调用 repay 的第三方都能以极低成本（1 share）改变目标状态，使受害者的精确金额 tx 持续 revert，形成无限循环 DoS（来源：RevertLend M-16）。两个条件缺一不可：① 精确金额校验（`if (amount > current) revert`）；② 第三方可触发目标状态的微小变化（如任何人均可为任意 tokenId 还款）；同时成立时必须将校验改为截断：`amount = min(amount, current)`。受影响路径：用户自救（接近清算阈值时偿还债务）+ 清算人（liquidate 指定精确金额）；两者都会被抢跑的单个 share 还款击败（不是常见的漏洞）

### 资产/白名单配置变更

#### 模式A：下架操作破坏清算公式（RevertLend M-20）
- [ ] 控制"是否允许某资产作为抵押品"的参数（如 `collateralFactor`）是否同时也参与清算计算？若是，将该参数设为 0 以下架资产时，会同时破坏已有仓位的清算逻辑（抵押值归零、除零等）
- [ ] 协议是否有两阶段资产下架机制：①先禁止新增（collateralFactor=0），等现有仓位被清算消化；②所有仓位清零后才完全移除？若只有一步操作，下架会立即影响现有仓位清算

#### 模式B：下架操作截断清算路径（Sentiment V2 M-11）
- [ ] owner 移除某个 token/资产的白名单资格时，已持有该资产的仓位（positionAssets）是否也被清除？若不清除：①资产仍参与抵押品计算；②若同时移除该资产的 oracle（设为 address(0)），清算时读取 oracle revert → 仓位永久不可清算 → 坏账
- [ ] 检查方法：搜索该配置标志的读取路径（添加时读）vs 清算/健康因子计算路径（直接遍历 positionAssets，不再读标志），对比两者的状态覆盖范围

共同识别思路：admin 对资产配置的任何变更（下架、禁用、移除 oracle）→ 追问"已有仓位中持有该资产的清算路径是否仍然完整可用？"

### 抵押品计算

- [ ] 仓位资产数量是否用 `balanceOf(position)` 计算？若是，任何人可向仓位地址直接转账，操纵抵押品数量。协议使用加权平均 LTV 时：向仓位捐赠低 LTV 资产 → 拉高其价值权重 → minRequired 飙升 → 强制仓位进入不健康状态并被清算（来源：Sentiment V2 M-13，见 card-22）。防御验证：抵押品数量是否用内部记账（虚拟余额）追踪，而非 `balanceOf`？
- [ ] 以 Uniswap V3 LP 仓位为抵押品时：`价值 = price × amounts`，**price 和 amounts 必须都防操纵**。amounts（amount0/amount1 的分布）由 `pool.slot0()`（spot price）决定，可被闪贷操纵；仅用 Chainlink/TWAP 保护 price 不够，攻击者操纵 spot 改变 amounts 比例，以 oracle price 估值时总价值仍被高估（来源：RevertLend M-19）；正确做法：用 TWAP 而非 spot 计算 amounts，或对 amounts 独立做价格合理性校验

### 清算

- [ ] 计算清算时的协议手续费应该从最后的利润中计算而不是从清算者获得的抵押物直接计算，会导致清算者无利可图
- [ ] 清算奖励发送给 msg.sender 而非收款人
- [ ] LTV（最大可借比例）与清算阈值是否相同？若相同则缓冲为零：用户借到上限时，价格任意微小下跌就触发清算，且攻击者可短暂操纵价格制造强制清算；正常设计应是 LTV < 清算阈值（至少 5% 缓冲）（来源：RevertLend M-11）
- [ ] LTV如果被设置为常量时是否过高，如98%，如果预言机有偏差可以导致>100%LTV
- [ ] 将 minDebt 和 minBorrow 设置为较低值可能导致协议产生坏账
- [ ] 健康检查（`isPositionHealthy`）是否遍历攻击者可控的 pool？若是，攻击者可通过故意污染某个 pool（使其 shares 溢出）来使遍历路径 revert，从而让自己在**其他**池的借款头寸永久不可清算（来源：Sentiment V2 M-23，见 card-26）；防御验证：坏账清算后该池是否被冻结/禁止再存款？

### 多池/聚合

- [ ] 多池/聚合存款架构：向第三方协议存款时，try/catch 捕获失败后是直接跳过（损失该池的部分剩余容量），还是用更小的量重试？内部计算的 supplyAmt 是否同时受到"内部上限"和"外部协议实际剩余"两个约束？
- [ ] 多池架构的池间隔离性：仓位可同时向多个池借款时：健康因子计算是否遍历所有debtPools全局聚合LTV（双重循环）？若是：Pool A设置低LTV会拉高全局minReqAssetValue，使"Pool B自身标准下健康"的仓位被触发清算，清算者可选择性偿还Pool B债务 → 违反"池风险隔离"设计（来源：Sentiment V2 M-16，见card-23）。清算者是否可以自由选择偿还哪个池的债务？若是，检查这个自由度是否会让"无辜池"受到清算（即使该池的LTV认为仓位健康）。对手方攻击路径：恶意Pool owner可故意降低LTV，触发其他池的清算，拿奖励+操纵其他池利用率；timelock不能完全缓解（天然配置差异即可触发，无需改LTV）
- [ ] depositQueue 与 withdrawQueue 是否顺序不同？若不同，任意用户可通过"存款→立即提款"将自己在 SuperPool 内的风险暴露从安全池切换到即将被坏账清算的池，自己零损失而其他用户承担更多坏账损失，该漏洞和在处理坏账前的普通跑路是同一种，但是这里是由于一个聚合的池子，通过存加取转嫁了被坏账社会化的损失（M-20型队列不对称攻击，见 card-25）。链上可观察的待处理坏账清算 + 不对称队列 = 上述攻击的充要条件；防御验证：是否有存取冷却期，或强制两队列顺序一致？

### 每日限额（dailyLimit）

- [ ] 协议有"每日限额"机制时，找到限额的**消耗路径**（deposit/borrow）和**释放路径**（withdraw/repay），检查两者是否对称调用同一 reset 逻辑
- [ ] reset 函数是完全覆盖（`=`）还是相对调整（`+=/-=`）？若是完全覆盖，任何在 reset 触发前通过释放路径累积的增加值都会在下次 reset 时被清除（来源：RevertLend M-08）
- [ ] 跨天时间轴验证：第 N 天末的释放操作（+X）→ 过午夜 → 第 N+1 天的消耗操作触发 reset → +X 是否被覆盖？若是，"释放为当天腾出空间"的设计意图完全失效
- [ ] 额度的**释放路径是否覆盖所有等效操作**？若 `repay` 触发 `dailyDebtIncreaseLimitLeft += assets`，则等效的"债务减少"操作（`liquidate`、强制平仓、自动偿还等）也必须触发相同更新；识别方法：找到所有"债务/存款减少"的路径，逐一对比是否有等效的额度释放（来源：RevertLend M-22）

### 利息计算时序

- [ ] 利息计算是否依赖 `block.timestamp` 差值（timeDelta）？同一区块内 timeDelta = 0，借款可支付零利息。攻击者能否在单区块内：借入最大可借量（占满 dailyLimit）→ 免息使用 → 还款，使当日限额被耗尽但协议零收入，且每区块重复导致持续 DoS？（来源：RevertLend M-04）。L2 部署时：是否多个区块可共享相同 `block.timestamp`？若是，免息窗口从 1 个区块扩展到多个区块，DoS 成本大幅降低。防御验证：是否有最小借款持续时间（cooldown）或同区块借还的最低利息保障（至少按 1 秒计算）？
- [ ] admin 调用改变利率计算参数的函数（setReserveFactor、setInterestRateModel 等）时，是否先调用 accrueInterest / _updateGlobalInterest？若不先 accrue，新参数会被追溯应用到过去那段 timeDelta，人为影响用户已应得的利息（来源：RevertLend M-05）；检查方法：搜索所有写入利率相关参数的 setter 函数，逐一检查函数体第一行是否有 accrue 调用

### 精度损失（低小数位资产）

- [ ] 利息/费用计算中是否有 `interest.mulDiv(rate, 1e18)` 类结构？代入6位小数资产（USDT/USDC）+ 短时间窗口（1-60秒）验算：结果是否归零？若归零，lastUpdated/时间戳是否仍被更新？若是，精度损失期间的时间被永久消耗，不可追回。（来源：Sentiment V2 M-18）
- [ ] 精度损失影响双方：①feeAssets归零 → 协议收入损失，Lenders可批量小额操作规避费用；②interestAccrued本身归零 → Borrowers少付利息，Lenders利息收入受损
- [ ] 是否有 minBorrow 防止微额借款触发此攻击？存款路径是否有类似保护（minDeposit）？

---

## AMM / DEX

> 从Uniswap V2/V3学习和后续盲审中积累

### Uniswap V3 NPM 集成

#### NPM `owed` 机制的关键特性（必须熟记）
- `decreaseLiquidity()` 不立即返还资产，只将流动性资产追加到该 tokenId 的 `owed`（tokensOwed0/1）
- `collect(type(uint128).max)` 一次性清空**全部历史积累的 owed**，包括所有之前 `decreaseLiquidity` 调用留下的量
- 用户可在协议外自行调用 `decreaseLiquidity`，将本金资产预存入 `owed`，而无需立即 collect

- [ ] 协议是否用 `collect总量 - 本次decreaseLiquidity量` 差值来推算"手续费"或"纯收益"？若是，用户可提前在协议外调用 `decreaseLiquidity`（不 collect），将本金堆入 owed，使差值虚胖；依赖此差值的手续费保护（如 `onlyFees` 模式）完全失效，运营商可获得超额奖励（来源：RevertLend M-17）。检查方法：搜索 `collect(... type(uint128).max ...)` 的调用，追问：此 `collect` 收取的 owed 是否只包含本次操作产生的？还是也包含历史 owed？若包含历史 owed，差值计算是否仍然成立？

### Uniswap V3 NPM deadline / 滑点

- [ ] 所有调用 Uniswap NPM 的地方（`increaseLiquidity`、`decreaseLiquidity`、`mint` 等），deadline 参数是否为 `block.timestamp`？若是，Uniswap 内部检查 `require(block.timestamp <= deadline)` 恒为 true，等价于无 deadline；tx 在 mempool 挂起后可被 MEV 机器人在任意时间提交执行（来源：RevertLend M-21）；快速检查：全局搜索 `block.timestamp` 在 NPM 调用参数中的出现
- [ ] deadline 与 minAmount（滑点保护）通常联动检查：两者同时为零/无效时，延迟执行的价格损失完全不受约束
- [ ] Automator/Keeper 等代理合约（不由用户直接调用）尤其要查 deadline：用户无法自己传入合理截止时间，协议必须自行设置或允许用户配置

---

## 跨协议交互（待补充）

> 闪电贷、预言机操纵、三明治攻击等

---

## Fork类的协议

- [ ] 需要考虑是否发生由于编译器版本的变化而导致相同的代码发生了不同的情况

---

## 预言机类漏洞

### Chainlink Oracle

- [ ] `latestRoundData()` 返回的 `price` 是否与 aggregator 的 `minAnswer`/`maxAnswer` 边界比较？若 `price == minAnswer`，说明实际价格已跌破边界，feed 返回截断值而非真实价格，协议会以虚高价格为缩水资产估值 → 借款人可用高估抵押品恶意借款制造坏账（来源：Sentiment V2 M-21）。检查方式：读取 `aggregator.minAnswer()` / `aggregator.maxAnswer()`，若 `price <= minAnswer || price >= maxAnswer` 则 revert；注意不同链/资产的边界值不同，需逐一核实
- [ ] L2 部署时（Arbitrum/Optimism 等），是否在调用 `latestRoundData()` 前检查 sequencer uptime feed？Sequencer 宕机期间价格看起来新鲜但实际已过期，可被利用；sequencer 恢复后还需等待 grace period（通常1小时）才可信任价格（来源：RevertLend M-27）

### Red Stone Oracle

- [ ] 在客户端或合约层面记录最后一次更新的具体观测值和/或哈希，并拒绝在有效期内相同或不同但已有的观测，否则在不断更新函数时可以使用之前的价格，或者由 oracle 协议本身返回带有时间戳的每个值，供消费者验证不被回退
- [ ] RedstoneCoreOracle在设置STALE_PRICE_THRESHOLD值时不能设置为一个常量，比如3600，因为不同的代币的STALE_PRICE_THRESHOLD值不同，比如TRX为10分钟，BNB为1分钟

### Uniswap V3 TWAP 集成
- [ ] `tickCumulativesDelta / twapSeconds` 计算 TWAP tick 时，是否处理负 delta 的向下取整？Solidity 整数除法对负数向零取整，但 Uniswap tick 语义要求向负无穷取整；缺少 `if (delta < 0 && delta % period != 0) tick--` 会使价格系统性偏高（来源：RevertLend H-05，见 card16）
- [ ] 与 Uniswap OracleLibrary 对比：`secondsAgos` 数组顺序是否一致？delta 计算方向（`[0]-[1]` vs `[1]-[0]`）是否和 OracleLibrary 对称？

### 价格偏差（Spread）计算

- [ ] 协议将 `|price - verified_price|` 的比值（spread）用作熔断/验证阈值时，分母是否统一？典型错误：`price > verified_price` 时除以 price，`price < verified_price` 时除以 verified_price；结果：同等绝对偏差（如均为10）在上涨方向产生更小的 spread（9.09% vs 10%），价格被向上操纵时比向下更难触发保护（来源：RevertLend M-25）；修复：统一除以 `verified_price`（参考基准）

---

## 跨链漏洞

- [ ]

---

## 机器人漏洞

- [ ]

---

# 更新日志

| 日期 | 来源 | 新增条目 |
|------|------|----------|
| 2026-02-23 | PoolTogether V5 PrizeVault 盲审 | ERC4626全部条目、通用检查中的变量赋值/hook/ERC20交互 |
| 2026-03-02 | Sentiment V2 M-20 复盘 | 借贷协议新增"SuperPool/聚合存款架构"节：队列不对称坏账转嫁 |
| 2026-03-08 | Sentiment V2 M-21 复盘 | 预言机新增 Chainlink minAnswer/maxAnswer 边界检查 |
| 2026-03-13 | Sentiment V2 M-23 复盘 | ERC4626边界状态M-9条目补充主动武器化变体；借贷协议清算节新增健康检查DoS条目 |
| 2026-03-22 | RevertLend M-02 复盘 | 外部可控组件节新增 ERC721 回调 Gas Grief + 状态残留检查项（card15 变体C） |