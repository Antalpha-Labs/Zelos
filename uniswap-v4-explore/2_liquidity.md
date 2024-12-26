# 探索Uniswap V4: 流动性管理

AMM的核心是流动性管理。尽管AMM并不是Uniswap第一个提出的，Uniswap也不是第一个链上AMM协议。但Uniswap是第一个将AMM推向主流的项目。在V3引入集中流动性之后，资金利用率的痛点得以解决。这让AMM的模式趋于成熟。

在有了成熟的模型之后，V4将重点放在了对流动性的优化上，这主要表现在以下几个方面：

**手续费再投资**

在V3中，手续费会一直在流动性头寸中单独累计，除非进行收集操作，才会将手续费转入到LP的账户中。因此，如果想将手续费再投资，需要先收集手续费，再添加流动性；或者干脆移除旧的流动性并重新投入。

在V4中，这一点得到了改进，在用户添加和移除流动性时，可以选择手续费收入的处理方式：添加流动性时，可以将手续费收入转换为头寸中的流动性；在移除流动性时，则会自动要求提取未领取的手续费收益。

**流动性标记**

在V3中，从`Pool`的角度看，对于每一个Position，`Pool`都记录了持有人，上下界。如果用户通过`NonfungiblePositionManager`添加流动性，在`Pool`中，会将持有人记录为`NonfungiblePositionManager`的地址，而且`NonfungiblePositionManager`会管理好每个流动性头寸。通常，用户会与`NonfungiblePositionManager`交互，这样没有问题。但是对于高级用户来说，他们通常会直接与`Pool`合约交互，这就引入了一个问题，如果同一个用户投资了多个头寸，而且这些头寸的上下界相同，他们会被流动性池看作同一个流动性头寸。对于个人LP这不是问题，但是对于基金类型的LP就很不方便，因为他们需要像`NonfungiblePositionManager`合约一样，自己记录并分配每个用户的投资和手续费。

V4作出了一些改进，创建流动性时，可以提供一个额外的参数salt，这使得用户在直接通过`Pool`合约添加流动性时，拥有一个额外的维度区分每个流动性。因此基金LP能够通过salt清楚的区分每个用户的头寸，不用计算每个用户到底需要分多少手续费。

**快速记账**

快速记账是V4中另一个强大的功能，可以在不发生实际转账的情况下，完成多跳交易。添加流动性的过程也可以从快速记账中受益。例如，用户希望为`ETH <> DAI `添加流动性，但手头没有`DAI`。用户可以先将一部分`ETH `兑换为`DAI`，以便使用两种代币添加流动性。此外，用户还可以通过多跳交换（multi-hop swap）实现，例如从 `ETH` 兑换为`USDC` 再兑换为`DAI`。如果集成得当，用户只需要一次性转移`ETH` 即可完成整个操作。

在添加了如此多的改进之后，流动性管理的接口也发生了变化。在V3中，流动性管理的接口非常清晰，每一个操作都有对应的操作。比如在`NonfungiblePositionManager`合约中，流动性管理的函数名是：

* mint
* increaseLiquidity
* decreaseLiquidity
* collect
* burn

而在`Pool`合约中，管理流动性的函数名是：

* mint
* collect
* burn

这些函数名看起来清晰明了。但在V4中，为了让流动性操作有足够的灵活性，流动性管理的接口被重新设计。在`PositionManager`中，所有的操作被集成到`modifyLiquidities`函数中。在`PoolManager`中，操作也被集中到了`modifyLiquidity`函数。

> 注意：`PoolManager`也有`mint`和`burn`函数，但是他们是用来管理ERC6909余额的，和V3中`mint`与`burn`的作用不同

以`PositionManager`为例，从`modifyLiquidities`这个名字可以看出，它是支持一次性多个操作的。事实也正是如此。它的设计思路有点像线性脚本，你可以设置多个动作，并设置参数，然后送到这个函数里一股脑执行。这里以创建流动性仓位举例，

1 首先创建一个`PositionManager`实例。

```solidity
import {IPositionManager} from "v4-periphery/src/interfaces/IPositionManager.sol";

IPositionManager posm = IPositionManager(<address>);
```

2 设置动作

要创建一个新的流动性仓位，需要设置两个动作：

1. mint： 用来创建流动性仓位
2. 交易对： 设置添加哪两个token

```solidity
import {Actions} from "v4-periphery/src/libraries/Actions.sol";

bytes memory actions = abi.encodePacked(Actions.MINT_POSITION, Actions.SETTLE_PAIR);
```

3 设置参数，这两个参数对应上面的两个动作

```solidity
bytes[] memory params = new bytes[](2);
```

其中`MINT_POSITION`需要如下参数

| 参数    | 类型      | 描述                                                 |
| ------------ | --------- | ------------------------------------------------------------ |
| `poolKey`    | *PoolKey* | 池的ID                        |
| `tickLower`  | *int24*   | Position的lower tick                      |
| `tickUpper`  | *int24*   | Position的upper tick     |
| `liquidity`  | *uint256* | 流动性的数量                        |
| `amount0Max` | *uint128* | msg.sender可支付token0的最大数量 |
| `amount1Max` | *uint128* | msg.sender可支付token1的最大数量  |
| `recipient`  | *address* | 接收流动性NFT(ERC-721)的地址 |
| `hookData`   | *bytes*   | hook所需参数，这些参数将被直接转发给Hook    |

```solidity
Currency currency0 = Currency.wrap(<tokenAddress1>); // 填写0地址表示使用ETH
Currency currency1 = Currency.wrap(<tokenAddress2>);
PoolKey poolKey = PoolKey(currency0, currency1, 3000, 60, IHooks(hook));

params[0] = abi.encode(poolKey, tickLower, tickUpper, liquidity, amount0Max, amount1Max, recipient, hookData);
```

而`SETTLE_PAIR` 动作需要两个参数：

- `currency0` - msg.sender将支付的token 0的地址
- `currency1` - msg.sender将支付的token 1的地址

```solidity
params[1] = abi.encode(currency0, currency1);
```

4 提交, 这里还可以设置一个过期时间。

```solidity
uint256 deadline = block.timestamp + 60;

uint256 valueToPass = currency0.isAddressZero() ? amount0Max : 0;

posm.modifyLiquidities{value: valueToPass}(
    abi.encode(actions, params),
    deadline
);
```

其它的操作（增加、移除流动性，收集手续费等）也与之类似。

在流动性的操作上，V4的改进并不大。这些改进更像是为了适应其它新功能而进行的升级。但即便如此，我们可以发现在V4的流动性管理上，功能更强，操作更方便。这些细致入微的改进，为流动性管理带来了更强的可操作性，同时为未来更复杂的 DeFi 场景奠定了基础。



