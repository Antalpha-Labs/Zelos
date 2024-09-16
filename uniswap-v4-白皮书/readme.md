# Uniswap v4 白皮书

[译者](https://github.com/32ethers)注: Uniswap V4正在做发布前的准备. 白皮书也从一年前的草稿变成了正式版, 和草稿相比, 正式版对很多细节做了修改. 因此我重新翻译了一下白皮书. 把最新的版本同步给大家. 

原文: [https://github.com/Uniswap/v4-core/blob/main/docs/whitepaper/whitepaper-v4.pdf](https://github.com/Uniswap/v4-core/blob/main/docs/whitepaper/whitepaper-v4.pdf)

**2024年8月**

作者: 

* [Hayden Adams](hayden@uniswap.org)
* [Moody Salem](moody.salem@gmail.com)
* [Noah Zinsmeister](noah@uniswap.org)
* [Sara Reynolds](sara@uniswap.org)
* [Austin Adams](me@aada.ms)
* [Will Pote](willpote@gmail.com)
* [Mark Toda](mark@uniswap.org)
* [Alice Henshaw](alice@uniswap.org)
* [Emily Williams](emily@uniswap.org)
* [Dan Robinson](dan@paradigm.xyz)

Uniswap v4 is a non-custodial automated market maker implemented for the Ethereum Virtual Machine. Uniswap v4 offers customizability via arbitrary code hooks, allowing developers to augment the concentrated liquidity model introduced in Uniswap v3 with new functionality. In Uniswap v4, anyone can create a new pool with a specified hook, which can run before or after predetermined pool actions. Hooks can be used to implement features that were previously built into the protocol, like oracles, as well as new features that previously would have required independent implementations of the protocol. Uniswap v4 also offers improved gas efficiency and developer experience through a singleton implementation, flash accounting, and support for native ETH.

Uniswap v4 是为以太坊虚拟机实现的非托管自动化做市商。Uniswap v4 通过任意代码挂钩（hooks）提供了定制化功能，允许开发者在 Uniswap v3 引入的集中流动性模型上扩展新功能。在 Uniswap v4 中，任何人都可以通过指定的挂钩创建新的资金池，该挂钩可以在预定的资金池操作之前或之后运行。挂钩可以用于实现之前内置于协议中的功能，比如预言机（oracles），以及那些以前需要独立实现协议的新功能。Uniswap v4 还通过单例实现、闪电记账（flash accounting）以及对原生 ETH 的支持，提供了改进的 gas 效率和开发者体验。


## 1 INTRODUCTION(介绍)

Uniswap v4 is an automated market maker (AMM) facilitating efficient exchange of value on the Ethereum Virtual Machine (EVM). As with previous versions of the Uniswap Protocol, it is noncustodial, non-upgradable, and permissionless. The focus of Uniswap v4 is on additional customization for developers and architectural changes for gas efficiency improvements, building on the AMM model built by Uniswap v1 and v2 and the concentrated liquidity model introduced in Uniswap v3.

Uniswap v4 是一种自动化做市商（AMM），用于在以太坊虚拟机（EVM）上促进高效的价值交换。与之前的 Uniswap 协议版本一样，它是非托管的、不可升级的，并且无需许可。Uniswap v4 的重点是为开发者提供更多的定制化选项，并通过架构上的改进提高 gas 效率，基于 Uniswap v1 和 v2 建立的 AMM 模型以及 Uniswap v3 引入的集中流动性模型进一步发展。

Uniswap v1 [2] and v2 [3] were the first two iterations of the Uniswap Protocol, facilitating ERC-20 <> ETH and ERC-20 <> ERC-20 swaps, respectively, both using a constant product market maker (CPMM) model. Uniswap v3 [4] introduced concentrated liquidity, enabling more capital efficient liquidity through positions that provide liquidity within a limited price range, and multiple fee tiers.

Uniswap v1 [2] 和 v2 [3] 是 Uniswap 协议的前两次迭代，分别促进了 ERC-20 对 ETH 和 ERC-20 对 ERC-20 的兑换，两者都使用了恒定乘积做市商（CPMM）模型。Uniswap v3 引入了集中流动性，通过在有限价格范围内提供流动性的头寸，使得资本效率更高，并且支持多个手续费等级。

While concentrated liquidity and fee tiers increased flexibility for liquidity providers and allowed for new liquidity provision strategies, Uniswap v3 lacks flexibility to support new functionalities invented as AMMs and DeFi have evolved.

尽管集中流动性和手续费等级为流动性提供者增加了灵活性，并允许采用新的流动性提供策略，Uniswap v3 在支持随着 AMM 和 DeFi 发展的新功能方面仍然缺乏足够的灵活性。

Some features, like the price oracle originally introduced in Uniswap v2 and included in Uniswap v3, allow integrators to utilize decentralized onchain pricing data, at the expense of increased gas costs for swappers and without customizability for integrators. Other possible enhancements, such as time-weighted average price orders (TWAP) through a time-weighted average market maker (TWAMM) [8], volatility oracles, limit orders, or dynamic fees, require reimplementations of the core protocol, and can not be added to Uniswap v3 by third-party developers.

一些功能，如最初在 Uniswap v2 中引入并包含在 Uniswap v3 中的价格预言机，允许集成者利用去中心化的链上定价数据，但代价是增加了兑换者的 gas 成本，且对集成者缺乏定制化支持。其他可能的改进，例如通过时间加权平均做市商（TWAMM）实现的时间加权平均价格订单（TWAP）、波动率预言机、限价单或动态手续费，均需要重新实现核心协议，第三方开发者无法在 Uniswap v3 中添加这些功能。

Additionally, in previous versions of Uniswap, deployment of new pools involves deploying a new contract—where cost scales with the size of the bytecode—and trades with multiple Uniswap pools involve transfers and redundant state updates across multiple contracts. Additionally since Uniswap v2, Uniswap has required ETH to be wrapped into an ERC-20, rather than supporting native ETH. These design choices came with increased gas costs for end users.

此外，在之前版本的 Uniswap 中，部署新资金池需要部署一个新合约——其成本随着字节码大小的增加而增加——并且与多个 Uniswap 资金池进行交易时，会涉及多次转账和多个合约的冗余状态更新。此外，自 Uniswap v2 起，Uniswap 要求将 ETH 包装为 ERC-20，而不支持原生 ETH。这些设计选择给终端用户带来了更高的 gas 成本。

In Uniswap v4, we improve on these inefficiencies through a few notable features:

在 Uniswap v4 中，我们通过以下几个显著特性改进了这些低效问题：

* Hooks: Uniswap v4 allows anyone to deploy new concentrated liquidity pools with custom functionality. For each pool, the creator can define a “hook contract” that implements logic executed at specific points in a call’s lifecycle. These hooks can also manage the swap fee of the pool dynamically, implement custom curves, and adjust fees charged to liquidity providers and swappers though Custom Accounting.
* Singleton: Uniswap v4 moves away from the factory model used in previous versions, instead implementing a single contract that holds all pools. The singleton model reduces the cost of pool creation and multi-hop trades.
* Flash accounting: The singleton uses “flash accounting,” which allows a caller to lock the pool and access any of its tokens, as long as no tokens are owed to or from the caller by the end of the lock. This functionality is made efficient by the transient storage opcodes described in EIP-1153 [5]. Flash accounting further reduces the gas cost of trades that cross multiple pools and supports more complex integrations with Uniswap v4.
* Native ETH: Uniswap v4 brings back support for native ETH, with support for pairs with native tokens inside v4 pools. ETH swappers and liquidity providers benefit from gas cost reductions from cheaper transfers and removal of additional wrapping costs.
* Custom Accounting: The singleton supports both augmenting and bypassing the native concentrated liquidity pools through hook-returned deltas, utilizing the singleton as an immutable settlement layer for connected pools. This feature can support use-cases like hook withdrawal fees, wrapping assets, or constant product market maker curves like Uniswap v2.

* 挂钩（Hooks）: Uniswap v4 允许任何人部署具有自定义功能的新集中流动性池。对于每个池，创建者可以定义一个“挂钩合约”，该合约在调用生命周期的特定点执行逻辑。这些挂钩还可以动态管理池的兑换手续费，实现自定义曲线，并通过自定义记账（Custom Accounting）调整向流动性提供者和兑换者收取的费用。
* 单例模式（Singleton）: Uniswap v4 摒弃了之前版本中使用的工厂模型，改为实施一个包含所有资金池的单一合约。单例模式降低了池创建成本和多跳交易的费用。
* 闪电记账（Flash Accounting）: 单例合约使用“闪电记账”，允许调用者锁定池并访问其任意代币，只要在锁定结束时没有代币欠款给或从调用者处借款。此功能通过 EIP-1153 [5] 中描述的瞬态存储操作码得以高效实现。闪电记账进一步降低了跨多个资金池交易的 gas 成本，并支持与 Uniswap v4 更复杂的集成。
* 原生 ETH: Uniswap v4 恢复了对原生 ETH 的支持，允许在 v4 资金池中使用包含原生代币的交易对。ETH 兑换者和流动性提供者可以从更便宜的转账和去除额外的包装成本中受益，降低 gas 成本。
* 自定义记账（Custom Accounting）: 单例合约支持通过挂钩返回的增量来增强和绕过原生集中流动性池，利用单例作为连接池的不可变结算层。此功能可以支持诸如挂钩提取费用、资产包装或像 Uniswap v2 那样的恒定乘积做市商曲线等用例。

The following sections provide in-depth explanations of these changes and the architectural changes that help make them possible

以下章节将深入解释这些变化及使这些变化成为可能的架构调整。

## 2.HOOKS(挂钩)

Hooks are externally deployed contracts that execute some developer-defined logic at a specified point in a pool’s execution. These hooks allow integrators to create a concentrated liquidity pool with flexible and customizable execution. Optionally, hooks can also return custom deltas that allow the hook to change the behavior of the swap — described in detail in the Custom Accounting section (5).

挂钩（Hooks）是外部部署的合约，在资金池执行的特定点执行一些由开发者定义的逻辑。这些挂钩允许集成者创建具有灵活和可定制执行的集中流动性池。可选地，挂钩还可以返回自定义增量，使挂钩能够改变兑换行为——在自定义记账（Custom Accounting）部分（5）中有详细描述。

Hooks can modify pool parameters, or add new features and functionality. Example functionalities that could be implemented with hooks include:

挂钩可以修改池参数，或添加新的功能和特性。通过挂钩可以实现的功能示例如下：

• Executing large orders over time through TWAMM [8]
• Onchain limit orders that fill at tick prices
• Volatility-shifting dynamic fees
• Mechanisms to internalize MEV for liquidity providers [1]
• Median, truncated, or other custom oracle implementations
• Constant Product Market Makers (Uniswap v2 functionality)

* 通过 TWAMM [8] 执行长期大额订单
* 在价格刻度处填充的链上限价单
* 波动率变化的动态费用
* 为流动性提供者内化MEV的机制[1]
* 中位数、截断或其他自定义预言机实现
* 恒定产品市场做市商(Uniswap V2的功能)


### 2.1 Action Hooks(操作的挂钩)

When someone creates a pool on Uniswap v4, they can specify a hook contract. This hook contract implements custom logic that the pool will call out to during its execution. Uniswap v4 currently supports ten such hook callbacks:

当有人在 Uniswap v4 上创建一个资金池时，他们可以指定一个挂钩合约。这个挂钩合约实现了自定义逻辑，在资金池执行过程中会被调用。Uniswap v4 目前支持以下十种挂钩回调：

• beforeInitialize/afterInitialize
• beforeAddLiquidity/afterAddLiquidity
• beforeRemoveLiquidity/afterRemoveLiquidity
• beforeSwap/afterSwap
• beforeDonate/afterDonate

The address of the hook contract determines which of these hook callbacks are executed. This creates a gas efficient and expressive methodology for determining the desired callbacks to execute, and ensures that even upgradeable hooks obey certain invariants. There are minimal requirements for creating a working hook. In Figure 1, we describe how the beforeSwap and afterSwap hooks work as part of swap execution flow.

挂钩合约的地址决定了哪些挂钩回调被执行。这种方式为确定所需的回调提供了一种高效且灵活的解决方案，同时确保即使是可升级的挂钩也遵守某些不变量。创建一个可运行的挂钩的要求非常低。在图 1 中，我们描述了 `beforeSwap` 和 `afterSwap` 挂钩如何作为兑换执行流程的一部分工作。

![钩子图](/imgs/uniswap-v4_1.png)

### 2.2 Hook-managed fees(通过挂钩管理手续费)

Uniswap v4 allows fees to be taken on swapping by the hook. Swap fees can be either static, or dynamically managed by a hook contract. The hook contract can also choose to allocate a percentage of the swap fees to itself. Fees that accrue to hook contracts can be allocated arbitrarily by the hook’s code, including to liquidity providers, swappers, hook creators, or any other party.

Uniswap v4 允许通过挂钩收取兑换手续费。兑换手续费可以是静态的，也可以由挂钩合约动态管理。挂钩合约还可以选择将一部分兑换手续费分配给自己。挂钩合约累积的费用可以由挂钩的代码任意分配，包括分配给流动性提供者、兑换者、挂钩创建者或任何其他方。

The capabilities of the hook are limited by immutable flags chosen when the pool is created. For example, a pool creator can choose whether a pool has a static fee (and what that fee is) or dynamic fees.

挂钩的功能受限于在创建池时选择的不可变标志。例如，池创建者可以选择池是否有静态费用（以及费用的具体数额）或动态费用。

Governance also can take a capped percentage of swap fees, as discussed below in the Governance section (6.2).

治理还可以收取一个封顶的兑换手续费百分比，具体内容在治理部分（6.2）中讨论。

## 3 SINGLETON AND FLASH ACCOUNTING(单利和闪电记账)

Previous versions of the Uniswap Protocol use the factory/pool pattern, where the factory creates separate contracts for new token pairs. Uniswap v4 uses a singleton design pattern where all pools are managed by a single contract, making pool deployment 99% cheaper.

以前版本的 Uniswap 协议使用工厂/池模式，工厂为新的代币对创建单独的合约。Uniswap v4 使用单例设计模式，其中所有资金池由一个合约管理，这使得池的部署成本降低了 99%。

The singleton design complements another architectural change in v4: flash accounting. In previous versions of the Uniswap Protocol, most operations (such as swapping or adding liquidity to a pool) ended by transferring tokens. In v4, each operation updates an internal net balance, known as a delta, only making external transfers at the end of the lock. The new take() and settle() functions can be used to borrow or deposit funds to the pool, respectively. By requiring that no tokens are owed to the pool manager or to the caller by the end of the call, the pool’s solvency is enforced. 

单例设计与 v4 中另一个架构变化——闪电记账（flash accounting）互补。在之前版本的 Uniswap 协议中，大多数操作（如兑换或向池中添加流动性）都以转账代币的方式结束。而在 v4 中，每个操作只更新一个内部净余额，称为增量（delta），仅在锁定结束时进行外部转账。新的 take() 和 settle() 函数可以分别用于借入或存入资金到池中。通过要求在调用结束时池管理者或调用者不欠任何代币，从而强制执行池的偿付能力。

Flash accounting simplifies complex pool operations, such as atomic swapping and adding. When combined with the singleton model, it also simplifies multi-hop trades or compound operations like swapping before adding liquidity.

闪电记账简化了复杂的池操作，如原子化的兑换和添加(流动性)。当与单例模型结合使用时，它还简化了多跳交易或复合操作（如在添加流动性之前兑换）。

Before the Cancun hard fork, the flash accounting architecture was expensive because it required storage updates at every balance change. Even though the contract guaranteed that internal accounting data is never actually serialized to storage, users would still pay those same costs once the storage refund cap was exceeded [6]. But, because balances must be 0 by the end of the transaction, accounting for these balances can be implemented with transient storage, as specified by EIP-1153 [5].

在 Cancun 硬分叉之前，闪电记账架构成本较高，因为它需要在每次余额变更时更新存储。尽管合约保证内部记账数据不会实际序列化到存储中，但一旦超出存储退款上限，用户仍需支付相同的成本 [6]。然而，由于余额必须在交易结束时为 0，因此这些余额的记账可以通过瞬态存储来实现，如 EIP-1153 [5] 所规定的。

Together, singleton and flash accounting enable more efficient routing across multiple v4 pools, reducing the cost of liquidity fragmentation. This is especially useful given the introduction of hooks, which will greatly increase the number of pools.

单例模式和闪电记账共同使得跨多个 v4 池的路由更加高效，减少了流动性碎片化的成本。鉴于挂钩的引入，这尤其有用，因为它将大大增加池的数量。


## 4 NATIVE ETH(原生ETH)

Uniswap v4 is bringing back native ETH in trading pairs. While Uniswap v1 was strictly ETH paired against ERC-20 tokens, native ETH pairs were removed in Uniswap v2 due to implementation complexity and concerns of liquidity fragmentation across WETH and ETH pairs. Singleton and flash accounting mitigate these problems, so Uniswap v4 allows for both WETH and ETH pairs. Native ETH transfers are about half the gas cost of ERC-20 transfers (21k gas for ETH and around 40k gas for ERC-20s). Currently Uniswap v2 and v3 require the vast majority of users to wrap (unwrap) their ETH to (from) WETH before (after) trading on the Uniswap Protocol, requiring extra gas. According to transaction data, the majority of users start or end their transactions in ETH, adding this additional unneeded complexity.

Uniswap v4 恢复了在交易对中使用原生 ETH。虽然 Uniswap v1 中严格使用 ETH 与 ERC-20 代币配对，但由于实现复杂性和 WETH 与 ETH 配对之间的流动性碎片化问题，Uniswap v2 移除了原生 ETH 配对。单例模式和闪电记账缓解了这些问题，因此 Uniswap v4 允许同时支持 WETH 和 ETH 配对。原生 ETH 转账的 gas 成本大约是 ERC-20 转账的一半（ETH 的 gas 成本为 21k，而 ERC-20 为约 40k）。目前，Uniswap v2 和 v3 要求绝大多数用户在 Uniswap 协议上交易前（后）将 ETH 包装（解包装）为 WETH，这增加了额外的 gas 成本。根据交易数据，大多数用户开始或结束他们的交易时使用 ETH，这增加了不必要的复杂性。

## 5 CUSTOM ACCOUNTING(自定义记账)

Newly introduced in Uniswap v4 is custom accounting which allows hook developers to alter end user actions utilizing hook-returned deltas, token amounts that are debited/credited to the user and credited/debited to the hook, respectively. This allows hook developers to potentially add withdrawal fees on LP positions, customized LP fee models, or match against some flow, all while ultimately utilizing the internal concentrated liquidity native to Uniswap v4.

在 Uniswap v4 中引入的新功能是自定义记账（custom accounting），允许挂钩开发者通过挂钩返回的增量（hook-returned deltas）来调整终端用户的操作，这些增量分别表示从用户账户借记/贷记的代币数量和贷记/借记到挂钩的代币数量。这使得挂钩开发者能够为流动性提供者（LP）位置添加提款费用、定制 LP 收费模型，或与某些流动性匹配，同时最终利用 Uniswap v4 原生的集中流动性。

Importantly, hook developers can also forgo the concentrated liquidity model entirely, creating custom curves from the v4 swap parameters. This creates interface composability for integrators allowing the hook to map the swap parameters to their internal logic.

重要的是，挂钩开发者还可以完全绕过集中流动性模型，从 v4 兑换参数创建自定义曲线。这为集成者提供了接口组合性，允许挂钩将兑换参数映射到其内部逻辑。

In Uniswap v3, users were required to utilize the concentrated liquidity AMM introduced in the same version. Since their introduction, concentrated liquidity AMMs have become widely used as the base liquidity provision strategy in the decentralized finance markets. While concentrated liquidity is able to support most arbitrary liquidity provision strategies, it may require increased gas overhead to implement specific strategies.

在 Uniswap v3 中，用户被要求利用该版本引入的集中流动性 AMM。自其引入以来，集中流动性 AMM 已成为去中心化金融市场中的基础流动性提供策略。尽管集中流动性可以支持大多数任意的流动性提供策略，但实施特定策略可能需要增加 gas 开销。

One possible example is a Uniswap v2 on Uniswap v4 hook, which bypasses the internal concentrated liquidity model entirely utilizing a constant product market maker fully inside of the hook. Using custom accounting is cheaper than creating a similar strategy in the concentrated liquidity math.

一个可能的例子是在 Uniswap v4 中使用的 Uniswap v2 挂钩，它完全绕过了内部集中流动性模型，利用挂钩内部的恒定乘积做市商。使用自定义记账比在集中流动性数学中创建类似策略更便宜。

The benefit of custom accounting for developers(compared to rolling a custom AMM) is the singleton, flash accounting, and ERC-6909. These features support cheaper multi-hop swaps, security benefits, and easier integration for flow. Developers should also benefit from a well-audited code-base for the basis of their AMM.

对于开发者来说，自定义记账的好处（与创建自定义 AMM 相比）包括单例模式、闪电记账和 ERC-6909。这些特性支持更便宜的多跳兑换、安全性益处和更容易的流动性集成。开发者还可以从经过良好审计的代码库中受益，作为他们 AMM 的基础。

Custom accounting will also support experimentation in liquidity provision strategies, which historically requires the creation of an entirely new AMM. Creating a custom AMM requires significant technical resources and investment, which may not be economically viable for many.

自定义记账还将支持流动性提供策略的实验，这在历史上需要创建一个全新的 AMM。创建自定义 AMM 需要大量的技术资源和投资，对于许多开发者来说，经济上可能不可行。

## 6 OTHER NOTABLE FEATURES(其他功能)

### 6.1 ERC-6909 Accounting(ERC-6909记账)

Uniswap v4 supports the minting/burning of singleton-implemented ERC-6909 tokens for additional token accounting, described in the ERC-6909 specification [7]. Users can now keep tokens within the singleton and avoid ERC-20 transfers to and from the contract. This will be especially valuable for users and hooks who continually use the same tokens over multiple blocks or transactions, like frequent swappers, liquidity providers, or custom accounting hooks.

Uniswap v4 支持铸造/销毁单例实现的 ERC-6909 代币，以进行额外的代币记账，具体描述见 ERC-6909 规范 [7]。用户现在可以将代币保留在单例合约中，避免了 ERC-20 转账进出合约。这对那些在多个区块或交易中持续使用相同代币的用户和挂钩尤为重要，例如频繁的兑换者、流动性提供者或自定义记账挂钩。

### 6.2 Governance updates(治理机制升级)

Similar to Uniswap v3, Uniswap v4 allows governance the ability to take up to a capped percentage of the swap fee on a particular pool, which are additive to LP fees. Unlike in Uniswap v3, governance does not control the permissible fee tiers or tick spacings.

与 Uniswap v3 类似，Uniswap v4 允许治理机制从特定资金池的交换手续费中收取一定的封顶百分比，这些费用是对流动性提供者费用的额外补充。但不同于 Uniswap v3，治理机制不再控制允许的手续费等级或价格刻度间隔。

### 6.3 Gas reductions(减少gas使用)

As discussed above, Uniswap v4 introduces meaningful gas optimizations through flash accounting, the singleton model, and support for native ETH. Additionally, the introduction of hooks makes the protocol-enshrined price oracle that was included in Uniswap v2 and Uniswap v3 unnecessary, which also means base pools forgo the oracle altogether and save around 15k gas on the first swap on a pool in each block.

如前所述，Uniswap v4 通过闪电记账、单例模型和对原生 ETH 的支持引入了重要的 gas 优化。此外，挂钩的引入使得 Uniswap v2 和 Uniswap v3 中包含的协议内置价格预言机变得不再必要，这也意味着基础资金池完全舍弃了预言机，每个区块内池的第一次兑换可节省约 15k gas。

### 6.4 donate() (捐赠函数)
donate() allows users, integrators, and hooks to directly pay in-range liquidity providers in either or both of the tokens of the pool. This functionality relies on the fee accounting system to facilitate efficient payments. The fee payment system can only support either of the tokens in the token pair for the pool. Potential use-cases could be tipping in-range liquidity providers on TWAMM orders or new types of fee systems.

donate()允许用户、集成者和挂钩直接向处于范围内的流动性提供者支付池中的一个或两个代币。这一功能依赖于费用记账系统以实现高效的支付。费用支付系统只能支持池中代币对的其中一个代币。潜在的使用场景包括对 TWAMM 订单中的范围内流动性提供者进行小费，或新型费用系统的实现。

## 7 SUMMARY(总结)

In summary, Uniswap v4 is a non-custodial, non-upgradeable, and permissionless AMM protocol. It builds upon the concentrated liquidity model introduced in Uniswap v3 with customizable pools through hooks. Complementary to hooks are other architectural changes like the singleton contract which holds all pool state in one contract, and flash accounting which enforces pool solvency across each pool efficiently. Additionally, hook developers can elect to bypass the concentrated liquidity entirely, utilizing the v4 singleton as an arbitrary delta resolver. Some other improvements are native ETH support, ERC-6909 balance accounting, new fee mechanisms, and the ability to donate to in-range liquidity providers.

总而言之，Uniswap v4 是一个非托管、不可升级且无需许可的 AMM 协议。它在 Uniswap v3 引入的集中流动性模型基础上，通过挂钩实现了可定制的资金池。与挂钩互补的还有其他架构变化，如单例合约，它将所有资金池状态集中在一个合约中，以及闪电记账，它高效地强制执行每个池的偿付能力。此外，挂钩开发者可以选择完全绕过集中流动性，利用 v4 单例作为任意增量解析器。其他改进包括对原生 ETH 的支持、ERC-6909 余额记账、新的费用机制，以及向范围内流动性提供者捐赠的能力。

## REFERENCES(引用)
[1] Austin Adams, Ciamac Moallemi, Sara Reynolds, and Dan Robinson. 2024. am-AMM: An Auction-Managed Automated Market Maker. arXiv preprint arXiv:2403.03367 (2024).
[2] Hayden Adams. 2018. Uniswap v1 Core. Retrieved Jun 12, 2023 from https://hackmd.io/@HaydenAdams/HJ9jLsfTz
[3] Hayden Adams, Noah Zinsmeister, and Dan Robinson. 2020. Uniswap v2 Core. Retrieved Jun 12, 2023 from https://uniswap.org/whitepaper.pdf
[4] Hayden Adams, Noah Zinsmeister, Moody Salem, River Keefer, and Dan Robinson. 2021. Uniswap v3 Core. Retrieved Jun 12, 2023 from https://uniswap.org/whitepaper-v3.pdf
[5] Alexey Akhunov and Moody Salem. 2018. EIP-1153: Transient storage opcodes. Retrieved Jun 12, 2023 from https://eips.ethereum.org/EIPS/eip-1153
[6] Vitalik Buterin and Martin Swende. 2021. EIP-3529: Reduction in refunds. Retrieved Jun 12, 2023 from https://eips.ethereum.org/EIPS/eip-3529
[7] JT Riley, Dillon, Sara, Vectorized, and Neodaoist. 2023. ERC-6909: Minimal MultiToken Interface. Retrieved Aug 26, 2024 from https://eips.ethereum.org/EIPS/eip-6909
[8] Dave White, Dan Robinson, and Hayden Adams. 2021. TWAMM. Retrieved Jun 12, 2023 from https://www.paradigm.xyz/2021/07/twamm


## DISCLAIMER(免责声明)
This paper is for general information purposes only. It does not constitute investment advice or a recommendation or solicitation to buy or sell any investment and should not be used in the evaluation of the merits of making any investment decision. It should not be relied upon for accounting, legal or tax advice or investment recommendations. This paper reflects current opinions of the authors and is not made on behalf of Uniswap Labs, Paradigm, or their affiliates and does not necessarily reflect the opinions of Uniswap Labs, Paradigm, their affiliates or individuals associated with them. The opinions reflected herein are subject to change without being updated.

本文仅供一般信息参考之用，不构成投资建议，也不构成购买或出售任何投资的推荐或引导，且不应作为评估任何投资决策优劣的依据。本文不应被用作会计、法律或税务建议或投资推荐的依据。本文反映了作者的当前观点，并非代表 Uniswap Labs、Paradigm 或其关联公司发表，且不一定反映 Uniswap Labs、Paradigm、其关联公司或与其相关人员的观点。本文所述观点可能会发生变化，且不会进行更新。





