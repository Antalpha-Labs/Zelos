# 探索Uniswap V4: 流动性池

流动性池是Uniswap的核心，也是每次升级的重头戏。如果说从V2到V3，最主要的变化是流动性，那么从V3到V4，最主要的变化是流动性池。在V4中，流动性池的变更主要有以下几个方面: 

* V3中，每一个流动性池到单独部署一个合约，而在V4中所有流动性池都通过PoolManager管理。
* 手续费不再仅限于几个特定值，可以自由设置手续费。
* V4支持动态手续费，可以根据特定条件调整手续费比例。

而流动性的基础模型则继承了下来，在V4中同样可以通过指定流动性仓位的价格范围，实现流动性的集中。

## PoolManager

在 Uniswap V3 中，流动性池是通过 UniswapV3Factory 合约部署的，每个池都是一个单独的智能合约。同时，每条链还有一个Proxy合约(这个合约在大部分链的地址是0x364484dfb8f2185b90e29fbd10ac96fca8a7e4a7)。普通用户在添加流动性的时候，要请求Proxy合约，然后Proxy合约再调用Pool合约。在这个过程中，资金先转入Proxy合约，再转入Pool变为流动性。最后Proxy合约会给用户发放一个NFT作为流动性权益的证明。这个过程很非常繁琐，会消耗很多gas。另外，用户在交易代币时，如果使用了多跳交易，也要在不同的Pool合约中转换，同样要额外花费一些Gas。

针对这个问题，Uniswap V4引入单例模式。所有的Pool都要通过PoolManager合约管理。在PoolManager中，`_pools`变量存放了所有的池。如果要新建一个Pool，只需要为`_pools`变量增加一个键值对。进行交易的时候，资金也不需要在各个合约之间划转，只要在PoolManager中做好记账就可以了。

```solidity
contract PoolManager is IPoolManager，ProtocolFees，NoDelegateCall，ERC6909Claims，Extsload，Exttload {
    // ...
    mapping(PoolId id => Pool.State) internal _pools;
    // ...
}
```

我们可以做一个粗糙的类比，我们需要进行A->B->C 的交易。
V3的菜市场，每一个摊位都要进行一次结算，通知各个ERC20进行支付，需要结算A，B，C。
V4大超市，只会在进门的时候充值，离开的时候提款，每个摊位的交易都是挂帐，不会真的结算。只需结算AC俩个资产。

我们实际上节省的费用就包含两部分:
1. 跨合约调用，在V3 菜市场移动到下一个摊位（pool address）都需要 付费，V4只支付一次。
2. ERC20结算费用，V4大超市砍掉了全部的交易路径中的清算成本。


鉴于PoolManager合约的重要性，Uniswap还搞了个[轰轰烈烈的地址挖矿活动](https://v4-address.uniswap.org/)， 为PoolManager选个吉祥地址,也能节省用户的gas。可见官方对这个合约的重视程度。

## 手续费

除了单例模式外，还有一个很大的变化是手续费。在V4中，可以自由设置手续费率，也可以动态调整手续费费率。要直观的了解这一点，可以看一下流动性池的创建参数。在V3中，创建池的参数是这样的:

> * address token0: 池的第一个token
> * address token1: 池的第二个token，通常第一个和第二个根据地址排序
> * uint24 fee: 池的手续费率，只能在1%，0.3%，0.05%，0.01%中选择

而在V4中，创建参数变成了`PoolKey`对象。这个对象的字段包括: 

> * Currency currency0: token 0。(Currency类型是address类型的别名)
> *  Currency currency1: token 1。
> *  uint24 fee: 池的手续费率，可以设置为不超过100%的值，如果设置为0x800000，表示池使用动态手续费。
> * int24 tickSpacing: 流动性的tickspacing。可以自由指定，从2到32766都可以。
> *  IHooks hooks: Pool的hook，想要详细了解可以查看[之前的文章](https://github.com/antalpha-labs/zelos)。

初始化参数主要包含两部分: 代币和手续费。V4的手续费设置有两个参数，`fee`和`tickSpacing`，而V3只有`fee`，这是由于在V3中， tick spacing是由手续费率决定的，约等于费率乘以200(比如0.05%*200=10，那么对于0.05%费率的池，tick spacing是10)。而V4取消了这种绑定关系，fee和tick spacing不再挂钩，因此要分开设置。而对于手续费本身来说，V3只能设置四种值(1%，0.3%，0.05%，0.01%)，相比于V2只能设置0.3%，已经有了一些进步。但是在V4中，手续费率可以在小于100%的数字中任意挑选，这让池的创建由了更高的自由度，可以根据代币的特点设置更精准的手续费值。

但这带来了一个问题，由于变一下手续费就会创建一个新的池，池的数量会大大增加。比如在V3中，USDC-ETH就存在0.05%和0.3%以及1%三个池，而且三个池的流动性都不少。到了V4，由于费率有1000000种可能，V4中USDC-ETH池可能会有上百个。要是考虑到不同的Hook也算新的池，未来的池就更多了。这必然会分散每个池的流动性。

## 动态手续费

除了Hook，V4中另一个大变化就是动态手续费了。所谓动态，就是允许流动性池在特定条件下改变手续费，这是通过Hook实现的，由于Hook可以在交易的诸多流程(比如流动性交易前后，token兑换交易前后)插入一定的业务逻辑，动态手续费也可以在Pool的交易中动态调整。这让动态手续费能够衍生出很多使用场景: 

* 根据当前市场价格调整: 价格越高，手续费越高
* 根据交易量调整: 对于大笔的交易，提供折扣.
* 根据gas fee调整: 让手续费与网络拥挤程度联动
* 根据语言机的价格调整: 通过外部的价格来为资产进行准确的定价
* 在特定时间调整，比如每年调整一次

总之，动态手续费让流动性池能够衍生出各种各样的产品。对流动性提供者来说，他们可以在波动率更高的时候获得更高的收益，从而有动力提供更多的流动性。而对普通用户来说，可以减少手续费的支出。

从Uniswap官方画的饼来看，他们对动态手续费的期待非常高: 

> * **改进波动性的定价**：根据市场波动性调整费用，类似于传统交易所调整买卖差价。
> * **根据订单定价**：更加准确地定价不同类型的交易（例如，套利交易与非信息性交易）。
> * **提升市场效率与稳定性**：费用可以根据实时市场状况进行调整，优化流动性提供者和交易者的利益。动态费用有助于通过实时调整激励措施来抑制极端市场波动。
> * **提升资本效率与流动性提供者回报**：通过优化费用，池子能够吸引更多流动性并促进更高效的交易。更精准的费用定价可能带来更好的回报，进而吸引更多资本进入池子。
> * **更好的风险管理**：在高波动时期，费用可以提高，以保护流动性提供者免受暂时性损失。
> * **可定制的策略**：为特定代币对或市场细分提供复杂的费用策略。

不过要注意的是，在Hook中也可以额外手续一部分手续费。这个手续费也可以是动态的，而且更加灵活。它和池的手续费不冲突，两个可以一起收。所以提到动态手续费，一般是指流动性池的手续费，这通常会叫做LP fee，而Hook收的手续一般会被称为Hook fee。

为流动性池设置动态手续费，需要在创建池的时候，将费率设置为`0x80000(LPFeeLibrary.DYNAMIC_FEE_FLAG)`.

在Hook中设置动态手续有两种办法。一是在Hook中调用`IPoolManager.updateDynamicLPFee(key，newValue)`改变手续费。这个函数可以在Hook的任意挂入点调用。 比如下面这个例子:

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

    /// 在特定流程中手工更新手续费
    function forceUpdateLPFee(PoolKey calldata key) external {
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

