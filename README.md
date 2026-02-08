阶段一：Polymarket架构与链上数据解码
学习目标
了解 Polymarket 的核心数据模型，包括事件 (Event)、市场 (Market)、条件 (Condition)、集合 (Collection)、头寸 (Position 或 TokenId)，并能用自己的话解释它们之间的关系。
理解 Polymarket 链上日志在"市场创建 → 交易 → 结算"全过程中的作用，以及不同日志之间如何串联形成证据链。
掌握链上日志解析的方法，能够实现：
交易解码 (Trade Decoder)：给定交易哈希，解析链上交易日志，还原交易详情（价格、数量、方向等）。
市场解码 (Market Decoder)：给定市场的 conditionId 或创建日志，还原该市场的链上参数（问题描述对应的标识、预言机地址、质押品、Yes/No 头寸 TokenId 等）。
核心概念
事件 (Event) 与市场 (Market)
在 Polymarket 中，事件代表一个预测主题，例如"某次美联储利率决议"。一个事件下可以包含一个或多个市场。每个市场对应该事件下的一个具体预测问题。例如，对于事件"2024年美国大选"可以有多个市场："候选人 A 当选总统？"、"候选人 B 当选总统？"等，每个市场通常是一个二元预测（Yes/No）。

Market（市场）：对应具体的 Yes/No 问题，是交易发生的基本单位。一些事件只有一个市场（例如简单的二元事件），而有些事件包含多个市场形成一个多结果事件。后者通常采用 Polymarket 的"负风险 (NegativeRisk)"机制来提高资金效率（见下文）。

NegativeRisk（负风险）：当一个事件包含多个互斥市场（即"赢家通吃"的多选一事件）时，Polymarket 引入 NegRiskAdapter 合约，将这些市场关联起来，提高流动性利用率。具体来说，在同一事件下，一个市场的 NO 头寸可以转换为该事件中所有其他市场的 YES 头寸。这意味着持有任意一个结果的反向（NO）仓位，相当于持有对其他所有可能结果的正向（YES）头寸。通过这种机制，参与者不需要为每个可能结果都分别提供独立的资金，从而提高资本效率。NegRiskAdapter 合约提供了 convert 功能，实现 NO → YES 的头寸转换。

示例：假设事件是"谁将赢得选举？"，包含 5 个候选人作为 5 个市场（每个市场问"候选人 X 会赢吗？"）。在负风险架构下：

YES 代币表示下注某候选人获胜；NO 代币表示下注该候选人不赢。
如果最终候选人 A 赢了，那么持有 A 市场 YES 代币的人可以兑回 1 USDC；持有其他所有候选人市场 NO 代币的人也可以各兑回 1 USDC（因为那些候选人没赢）。相反，持有 A 的 NO 代币、以及其他候选人的 YES 代币都变得一文不值。
通过 NegRiskAdapter，可以将对某候选人的 NO 头寸随时转换为对其他所有候选人的 YES 头寸持有。这体现了所有候选人"不赢"的头寸和其他候选人"赢"的头寸是等价的，从而联通了各市场的流动性。
条件 (Condition)、问题 (Question)、集合 (Collection) 与头寸 (Position/TokenId)
Polymarket 使用 Gnosis 开发的条件代币框架 (Conditional Token Framework, CTF) 来实现预测市场的头寸代币化。在该框架下：

Condition（条件）
每个市场在链上的"登记身份"。创建市场时，会调用 CTF 合约的 prepareCondition 方法注册一个条件。ConditionId 是通过哈希计算得出的唯一标识：

conditionId = keccak256(oracle, questionId, outcomeSlotCount)
其中：

oracle 是预言机合约地址（Polymarket 目前使用 UMA Optimistic Oracle 作为预言机）。
questionId 是问题的标识符（通常由问题内容等信息哈希得到，或 UMA Oracle 的 ancillary data 哈希）。
outcomeSlotCount 是结果选项数量。对于二元市场，该值为 2。
Condition 就像市场的问题在链上的"出生证明"，绑定了唯一的问题 ID 和预言机。当市场需要结算时，预言机会针对这个 conditionId 发布结果。

Position（头寸）
头寸指的是用户持有的某市场某结果的份额（又称 Outcome Share）。Polymarket 将每个头寸实现为一个 ERC-1155 标准的可交易代币（又称 PositionId 或 TokenId）。每种结果对应一个不同的 TokenId，用于区分 YES 和 NO 两种头寸。

CollectionId（集合 ID）
在条件代币框架中，中间引入了集合的概念，用于表示特定条件下某个结果集合。计算方法为：

collectionId = keccak256(parentCollectionId, conditionId, indexSet)
其中：

parentCollectionId 对于独立的条件通常为 bytes32(0)（Polymarket 所有市场都是独立条件，没有嵌套条件，因此 parentCollectionId 一律为 0）。
indexSet 是一个二进制位掩码，表示选取哪些结果槽位。对于二元市场，有两个可能的 indexSet：
YES 头寸的 indexSet = 1 (0b01，表示选取第一个结果槽)。
NO 头寸的 indexSet = 2 (0b10，表示选取第二个结果槽)。
TokenId（PositionId）
最后，用抵押品代币地址和集合 ID 一起计算得到 ERC-1155 的 Token ID：

tokenId = keccak256(collateralToken, collectionId)
在 Polymarket 中，对于每个条件(市场)，会产生两个 TokenId —— 一个对应 YES 份额，一个对应 NO 份额。这两个 TokenId 是在该市场上交易的标的资产，代表了对同一预测问题的两种相反结果的头寸。

Collateral（抵押品）
Polymarket 市场的押注资金均以稳定币 USDC (Polygon 上为 USDC.e，地址 0x2791...Aa84174) 作为抵押品。每份 Outcome Token 背后对应 1 USDC 的抵押，当市场结算时兑现。

价格含义：比如价格 0.60 USDC 意味着花 0.60 USDC 可购买该市场 1 份 YES 代币。如果该结果最终发生，持有者可赎回 1 USDC（获得净盈利 0.40 USDC）；如果未发生，则该代币价值归零，损失全部本金 0.60 USDC。因此，二元期权代币价格可以理解为市场对该事件发生概率的定价。

市场的完整生命周期与链上证据链
Polymarket 的市场从创建到结算，关键的链上步骤和日志事件如下：

1. 市场创建 (Creation) – 登记问题
由市场创建者调用 ConditionalTokens.prepareCondition 创建条件。

关键日志：ConditionPreparation 事件，包含 conditionId、oracle、questionId、outcomeSlotCount 等信息。这个事件在链上确认了某预言机地址与问题 ID 的绑定关系，相当于市场的建立。一旦发布，预言机（UMA OptimisticOracle）稍后将根据这个 conditionId 报告结果。

2. 初始流动性提供与拆分 (Split) – 生成初始头寸代币
市场创建后，需要流动性提供者拆分出初始的 YES/NO 代币。通常通过调用 ConditionalTokens.splitPosition 将抵押品 USDC 拆分成等价值的 YES 和 NO 头寸。

关键日志：PositionSplit 事件，包含 conditionId、collateralToken（应为 USDC 地址）、parentCollectionId（一般为 0）、partition（拆分出的 indexSets 列表，如 [1,2]）、以及 amount（拆分抵押品数量）。该事件证明抵押品被锁定，并生成了对应数量的 YES 和 NO 代币。对于二元市场，拆分 1 USDC 通常会得到面值各 1 USDC 的 YES 和 NO 代币各一枚。最初的流动性提供者可能是做市商，他们将 USDC 拆分为两种头寸代币，并可以在订单簿上挂单提供买卖报价。

3. 交易 (Trading) – 撮合买卖订单
交易在 Polymarket 的链上撮合引擎（CLOB 合约）中进行。Polymarket 采用中心限价订单簿模型，订单撮合通过智能合约（对于普通二元市场是 CTF Exchange，多结果市场则通过 NegRisk_CTFExchange）完成。每笔撮合成交在链上记录交易日志：

关键日志：OrderFilled 事件。每当买卖双方的订单在链上部分或全部成交时，都会触发该事件，记录交易的详情，包括：

maker 和 taker 地址：做市（挂单）方和吃单方地址。
makerAssetId 和 takerAssetId：成交时双方各自支付的资产 ID（Polymarket 将资产用一个 ID 表示：0 表示 USDC，非零则表示特定市场的头寸 TokenId）。
makerAmountFilled 和 takerAmountFilled：各方成交的数量（对应各自资产的数量，整数形式）。
fee：maker 方支付的手续费数量。
因 Polymarket 交易总是发生在 USDC 和某个 Outcome Token 之间，所以一个 OrderFilled 事件里必然有一个资产是 USDC（资产 ID 为 0），另一个资产是某市场的 TokenId。例如，若 makerAssetId = 0 且 takerAssetId 为某 TokenId，则表示挂单方卖出 USDC、买入了该 TokenId（即挂单方下的是买入该头寸的订单）。相应地，成交价可以通过 makerAmountFilled 和 takerAmountFilled 计算得出（需结合资产的精度，详见下文"交易日志解码"部分）。在订单完全撮合时，通常还会伴随一个 OrdersMatched 事件，它将多个 OrderFilled 事件归组表示一次撮合完成，但初学分析时可以主要关注 OrderFilled 日志。

重要细节（避免重复计数）：同一笔撮合在链上可能产生多条 OrderFilled。通常会有“每个 maker 一条”的 OrderFilled，以及一条“taker 汇总”的 OrderFilled，其中 taker 字段会显示为 Exchange 合约地址本身。若直接按 OrderFilled 条数统计成交笔数或成交量，会出现双计。实践中可选择以下方式之一避免重复：

过滤掉 taker == exchange_address 的 OrderFilled（保留 maker 侧填单）。
或改用 OrdersMatched 作为“一次撮合”的唯一汇总记录。
值得注意的是，在 Polymarket 内部，真正的代币铸造和销毁与交易匹配是紧密相关的。当两个相反方向的订单成交且共同投入的 USDC 满足 1:1 配比时，会触发抵押品锁定和头寸代币铸造的过程。例如：

如果一名买家愿意以 0.70 USDC 价格买入 YES，另一名卖家（或另一买单的对手）愿意以 0.30 USDC 价格买入 NO，两人的意向可匹配为一笔交易：总共 1.00 USDC 被锁定，铸造出 1 个 YES 和 1 个 NO 代币，分别分配给出价 0.70 的买家和出价 0.30 的买家。这对应链上 PositionSplit 事件记录了 1 USDC 拆分出一对头寸，以及后续的 OrderFilled 记录了双方各得到代币和支出 USDC 的情况。
类似地，如果一方想卖出 YES 代币，另一方想卖出相同市场的 NO 代币，两笔 sell 单可以匹配成一个"合并"操作：这两枚 YES 和 NO 代币被同时销毁并赎回总计 1 USDC 给卖出方（各得相应报价的 USDC）。这种情况下会出现 PositionsMerge 和 OrderFilled 等日志，表示头寸被合并赎回。这是 Polymarket 允许无需等待事件结算就能退出仓位的一种机制。
4. 结算 (Resolution) – 确定结果并清算
当事件结果揭晓且到达市场设定的关闭时间后，预言机合约（例如 UMA OptimisticOracle）将把结果提交回 CTF 合约，调用 reportPayouts(conditionId, payouts[]) 来公布各结果的兑付率。

结果日志：调用 reportPayouts 本身通常不会有特殊事件（或有 ConditionResolution 事件），但其效果是将相应 conditionId 下的头寸标记为可赎回：胜出的头寸代币每份价值 1 USDC，失败的头寸代币价值 0。用户随后可以调用 ConditionalTokens.redeemPositions 来赎回胜出代币的抵押品。Polymarket 当前使用 UMA 的乐观预言机机制，这意味着通常在预言机确认结果后，通过 Polymarket 前端或合约即可触发结算。结算完成后，对应的 YES/NO 代币可以兑换回 USDC，市场生命周期结束。

以上链上事件共同构成了市场的证据链：从 ConditionPreparation 证明市场的存在和参数、PositionSplit 证明资金注入和代币铸造、OrderFilled 记录交易交换细节、直到 reportPayouts 确认结果以供赎回。这些事件串联起来，可以让我们基于链上数据重建出市场发生的一切。
