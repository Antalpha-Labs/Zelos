# 探索Uniswap V4: 流动性池

流动性池是Uniswap的核心，也是每次升级的重头戏。如果说从V2到V3，最主要的变化是集中流动性，那么从V3到V4，最主要的变化是流动性池。在V4中，流动性池的变更主要有以下几个方面: 

* V3的每一个流动性池要单独部署一个合约，而在V4中所有流动性池都在PoolManager合约中。
* 手续费不再仅限于几个特定值，可以自由设置手续费。
* V4支持动态手续费，可以根据特定条件调整手续费比例。

下面将逐一详细介绍每个方面的改进，深入探讨它们的实现原理以及影响。

## PoolManager

在 Uniswap V3 中，流动性池是通过 UniswapV3Factory 合约部署的，每个池都是一个单独的智能合约。同时，每条链还有一个Proxy合约(这个合约在大部分链的地址是`0x364484dfb8f2185b90e29fbd10ac96fca8a7e4a7`)。在添加流动性的时候，用户需要对Proxy合约发起交易，Proxy合约再去调用Pool合约。在这个过程中，资金要先转入Proxy合约，再转入Pool变为流动性，最后Proxy合约会给用户发放一个NFT作为流动性权益的证明。可见这个过程非常繁琐，会消耗很多gas。另外，用户在交易代币时，如果使用了多跳交易，也要在不同的Pool合约中转换，同样要额外花费一些Gas。

针对这个问题，Uniswap V4引入单例模式，让所有的Pool通过PoolManager合约管理。在PoolManager有一个变量`_pools`，这个变量保存了所有池的状态。如果要新建一个Pool，只需要为`_pools`变量增加一个键值对就可以了，不需要部署合约。进行交易的时候，资金也用不着在各个合约之间划转，只要在PoolManager中做好记账就可以了。

```solidity
contract PoolManager is IPoolManager，ProtocolFees，NoDelegateCall，ERC6909Claims，Extsload，Exttload {
    // ...
    mapping(PoolId id => Pool.State) internal _pools;
	// ...
}
```

这样做的好处很多，我们可以做一个粗糙的类比。比如我们需要进行BTC到USDC的交易，然后路由计算发现，先把BTC兑换成ETH，再把ETH兑换成USDC是最省钱的方式。因此我们的兑换路径是BTC->ETH->USDC。
对于Uniswap V3来说，它就像一个菜市场，每一个摊位都要进行一次结算，通知各个ERC20进行支付，需要分别结算BTC，ETH，USDC。
而Uniswap V4则像一个大超市，只需要在进门的时候充值，离开的时候提款就可以了，在每个柜台的交易都是挂帐，不会真的结算。只在最后结算BTC和USDC俩个资产。

在这个过程中节省的费用包含两部分：
1. 跨合约调用，在V3菜市场中每次移动摊位（pool address）都需要付费，而V4只支付一次。
2. ERC20结算费用，V4的大超市模式砍掉了全部的交易路径中的清算成本。

鉴于PoolManager合约的重要性，Uniswap还搞了个[轰轰烈烈的地址挖矿活动](https://v4-address.uniswap.org/)， 为PoolManager选个吉祥地址,也能节省用户的gas。可见官方对这个合约的重视程度。

单利模式的实现，依赖于瞬态存储技术。瞬态存储是以太坊中新的数据存储方式，在EIP-1153中提出，并在2024年通过坎昆升级上线。它允许在单个交易执行期间临时存储数据，并在交易结束后自动清除。瞬态存储填补了现有存储机制的空缺，尤其适合那些数据只需在单个交易内有效且需要频繁读写的场景。下面是瞬态存储（Transient Storage）、存储（Storage）、内存（Memory）和调用数据（Calldata）的比较（引用自[瞬态存储：Solidity 中的高效临时数据解决方案](https://learnblockchain.cn/article/9847)）

| 特性         | 瞬态存储                | 存储                            | 内存               | 调用数据           |
| :----------- | :---------------------- | :------------------------------ | :----------------- | :----------------- |
| **存储位置** | 临时存储                | 永久存储                        | 临时存储           | 只读存储           |
| **Gas 成本** | 较低（约 100 gas/操作） | 高昂（首次写入至少 20,000 gas） | 较低               | 较低               |
| **作用域**   | 跨函数调用可用          | 跨交易可用                      | 仅限于单个函数调用 | 仅限于函数参数     |
| **状态保持** | 可以保持状态            | 永久保持状态                    | 不可保持状态       | 不可保持状态       |
| **清理成本** | 无需清理                | 需要额外支付 gas                | 无需清理           | 无需清理           |
| **大小限制** | 无明显限制              | 受链上状态限制                  | 有限的内存空间     | 有限的输入参数大小 |
| **适用场景** | 临时数据和计算          | 永久性数据存储                  | 临时数据处理       | 函数参数传递       |

正是通过引入瞬态存储（Transient Storage）机制，Uniswap V4能够在单个交易中以极低的成本存储交易状态，而不需要将中间变量永久性地写入区块链存储。这种设计不仅降低了每次交易的gas消耗，也提升了V4的交易效率。

## 手续费

除了单例模式外，另一个巨大的变化是手续费。与V3相比，V4的流动性池可以自由设置手续费率，不再局限于几个特定的值。要直观的了解这一点，可以看一下流动性池的创建参数。在V3中，创建池的参数是这样的:

> * address token0: 池的第一个token
> * address token1: 池的第二个token，通常第一个和第二个根据地址排序
> * uint24 fee: 池的手续费率，只能在1%，0.3%，0.05%，0.01%中选择

而在V4中，创建参数被封装成`PoolKey`对象。这个对象的字段包括: 

> * Currency currency0: token 0。(Currency类型是address类型的别名)
> *  Currency currency1: token 1。
> *  uint24 fee: 池的手续费率，可以设置为不超过100%的值，如果设置为0x800000，表示池使用动态手续费。
> * int24 tickSpacing: 流动性的tickspacing。可以自由指定，从2到32766都可以。
> *  IHooks hooks: Pool的hook，想要详细了解可以查看[之前的文章](https://github.com/antalpha-labs/zelos)。

从这些参数可见，流动性池的初始化参数主要包含两部分: 代币和手续费。

V4的手续费设置有两个参数，`fee`和`tickSpacing`，而V3只有`fee`。对于手续费，V3只能设置四种值(1%，0.3%，0.05%，0.01%)，相比于V2只能设置0.3%，已经有了一些进步。但是在V4中，手续费率可以在小于100%的数字中任意挑选，这让池的创建由了更高的自由度，可以根据代币的特点设置更精准的手续费值。

`tickSpacing`是流动性的粒度。较小的值可提高价格精度；然而，较小的值将导致swap交易更频繁地跨越tick，从而产生更高的gas成本。在V3中，tick spacing是由手续费率决定的，对应关系如下：

| Fee   | Fee Value | Tick Spacing |
| ----- | --------- | ------------ |
| 0.01% | 100       | 1            |
| 0.05% | 500       | 10           |
| 0.30% | 3000      | 60           |
| 1.00% | 10_000    | 200          |

而V4取消了这种绑定关系，因此fee和tick spacing变成了两个参数分开设置，在V4中，tick spacing可以取从2～32766的任意整数。至于这个值取大还是取小，就要权衡价格精度以及交易频率了。比如对于稳定币来说，tick spacing要尽量小，而对于BTC这种价格接近十万的代币来说，tick spacing可以大一些以节约gas。将费率和tick spacing解绑，相当于解除了代币价格和精度的绑定关系，这会让高价格的币种收益。

这种任意设置带来了一个问题。对于相同的交易对，每个手续费，每种tick spacing都是一个新的池，因此池的数量会大大增加。比如在V3中，USDC-ETH就存在0.05%和0.3%以及1%三个池，而且三个池都很活跃，流动性排名都在前50。到了V4，由于费率有1000000种可能，V4中USDC-ETH池很容易出现上百个。要是考虑到不同的Hook也算新的池，未来的池就更多了。这必然会分散每个池的流动性。

## 动态手续费

动态手续费是V4中除Hook以外最大的变化了，意义不亚于V3的集中流动性。所谓“动态手续费”，是指流动性池的手续费不再固定为某个具体值，而是可以根据市场的具体情况随时进行调整。这种灵活性使得动态手续费能够衍生出多种使用场景，为Uniswap带来更多的创新应用和策略，从而提高整个协议的效率和适应性。以下是一些可能的动态手续费调整场景：

1. **根据当前市场价格调整**
   在这种模式下，手续费的调整与市场价格的波动挂钩。例如，当某种资产的价格上涨时，手续费也相应增加，反之亦然。这种调整方式可以帮助Uniswap更好地适应市场环境的变化，尤其是在资产价格剧烈波动时，增加手续费可以有效地补偿流动性提供者的风险。
2. **根据交易量调整**
   在交易量较大时，手续费可以进行折扣或者优惠。这种模式适用于大宗交易，既能吸引高频交易者来降低其交易成本，也能够鼓励流动性提供者提供更多的流动性，特别是在交易量大、价格波动小的市场中。通过优化手续费结构，可以实现市场需求与流动性提供者收益之间的平衡。
3. **根据网络gas费调整**
   这一机制使得手续费与区块链网络的拥挤程度进行联动。在网络gas费较高时，流动性池可以自动提高手续费，控制流动性的变化速度；而当网络较为空闲时，手续费则可以适当降低。这种调整方式有助于流动性池的运营效率，也有助于用户在低网络费用时享受更低的交易成本。
4. **根据外部价格（如预言机）调整**
   通过与外部预言机或市场数据源的结合，Uniswap能够根据准确的外部价格来动态调整手续费。这种调整机制可以确保流动性池的手续费始终与市场的实际情况相匹配，从而提供更加公平和精确的定价机制。例如，当某种资产的市场波动性增大时，手续费可以增加，以反映其风险。
5. **在特定时间调整**
   这种策略允许在某些特定时间点（如每年、季度或按周）调整手续费率。这种方式适用于需要定期审视市场环境或根据季节性需求进行优化的场景。例如，每年根据上一年度的市场表现和流动性需求进行调整，以保持系统的长期健康发展。

通过上述例子可以看出，动态手续费的引入不仅为流动性池提供了更多的灵活性，也为流动性提供者和用户带来了不同的收益与节省机会。对于流动性提供者来说，动态手续费能够在市场波动较大的时候提供更高的收益，从而激励他们在高风险时期提供更多的流动性；而对于普通用户来说，手续费的动态调整能够根据交易时段、市场情况或网络拥堵程度进行优化，减少交易成本，从而提高交易体验和平台的整体吸引力。

总之，动态手续费为Uniswap引入了一种更为灵活和智能的费用管理方式，不仅增强了市场适应性，还为流动性提供者和用户提供了更多的选择和机会，推动去中心化交易协议在日益复杂的市场环境中不断向前发展。

下面的话引用了Uniswap官方的说法，描述了他们眼中动态手续费的场景：

> * **改进波动性的定价**：根据市场波动性调整费用，类似于传统交易所调整买卖差价。
> * **根据订单定价**：更加准确地定价不同类型的交易（例如，套利交易与非信息性交易）。
> * **提升市场效率与稳定性**：费用可以根据实时市场状况进行调整，优化流动性提供者和交易者的利益。动态费用有助于通过实时调整激励措施来抑制极端市场波动。
> * **提升资本效率与流动性提供者回报**：通过优化费用，池子能够吸引更多流动性并促进更高效的交易。更精准的费用定价可能带来更好的回报，进而吸引更多资本进入池子。
> * **更好的风险管理**：在高波动时期，费用可以提高，以保护流动性提供者免受暂时性损失。
> * **可定制的策略**：为特定代币对或市场细分提供复杂的费用策略。

动态手续费的实现依赖于流动性池的Hook。Hook是V4的新概念，它是一个单独的合约，可以为流动性池增加额外的逻辑。通过Hook，流动性池可以在交易前后插入一些自定义函数，在这些函数中都可以更改手续费率。因此，更改手续费的方式是非常灵活的，比如在添加移除流动性的时候，或者swap的前后都可以计算新的手续费，并实时更改池的手续费率

如果要允许流动性池使用动态手续费，需要在创建池的时候，将费率设置为`0x80000`， 也就是`LPFeeLibrary.DYNAMIC_FEE_FLAG`

```solidity
import {PoolKey} from "v4-core/src/types/PoolKey.sol";

PoolKey memory pool = PoolKey({
    currency0: currency0,
    currency1: currency1,
    fee: LPFeeLibrary.DYNAMIC_FEE_FLAG,
    tickSpacing: tickSpacing,
    hooks: hookContract
});
IPoolManager(manager).initialize(pool, startingPrice);
```

而改变动态手续费率有两种办法。一是在Hook中调用`IPoolManager.updateDynamicLPFee(key，newValue)`，永久改变池的手续费率。这个函数可以在Hook的任意挂入点调用。 比如下面这个例子:

```solidity
contract HookExample {
    IPoolManager public immutable manager;

    constructor(IPoolManager _manager) {
        manager = _manager;
    }

    /// 在初始化池的时候设置初始的手续费
    function afterInitialize(
        address sender,
        PoolKey calldata key,
        uint160 sqrtPriceX96,
        int24 tick,
        bytes calldata hookData
    ) external returns (bytes4) {
        uint24 INITIAL_FEE = 3000; // 0.30%
        manager.updateDynamicLPFee(key，INITIAL_FEE);
        
        return this.afterInitialize.selector;
    }

    /// 在添加流动性之前更新手续费
    function beforeAddLiquidity(
        address,
        PoolKey calldata key,
        IPoolManager.ModifyLiquidityParams calldata,
        bytes calldata
    ) external override returns (bytes4) {
        // 计算得到新手续费
        uint24 newFee = XXX;
        manager.updateDynamicLPFee(key，newFee);
    }
}
```

> 注意: 在创建池后，默认费率是0，所以要在Hook的`afterInitialize`为手续费设置一个初始值

另一种方式是临时更改费率。在Hook的`beforeSwap`中，可以返回一个费率，然后此次swap交易就会使用这个新的费率。而池的手续费率不会更改，其他的交易也不会受影响。

```solidity
import {BeforeSwapDelta，BeforeSwapDeltaLibrary} from "v4-core/src/types/BeforeSwapDelta.sol";
import {LPFeeLibrary} from "v4-core/src/libraries/LPFeeLibrary.sol";

contract HookExample {

    // ...

    function beforeSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        bytes calldata hookData
    ) external returns (bytes4，BeforeSwapDelta，uint24) {
        // 通过设置uint24的第二高位，设置临时手续费
        uint24 fee = 3000 | LPFeeLibrary.OVERRIDE_FEE_FLAG;
        return (this.beforeSwap.selector，BeforeSwapDeltaLibrary.ZERO_DELTA，fee);
    }
}
```

不过对于动态手续费，有两个概念容易混淆。在Hook中也可以额外手续一部分手续费，它和池的手续费不冲突，两个可以一起收。事实上，流动性池的手续费一般会叫做LP fee，而Hook收的手续费会叫Hook fee。一般提到动态手续费是指流动性池的手续费。

## 总结

作为领先的AMM协议，Uniswap V4的流动性池更新再次彰显了其在去中心化交易协议领域的创新与领先地位。这一系列的变革不仅增强了协议的灵活性和可扩展性，还显著提升了其市场竞争力，使得Uniswap V4在许多方面将对手甩开，继续保持在DeFi生态中的主导地位。
