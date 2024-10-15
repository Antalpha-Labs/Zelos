# 编写第一个hook

## 引言

在去中心化金融（DeFi）的世界中，Uniswap 占据着不可或缺的地位。随着 Uniswap 的不断演进，V4 版本引入了 Hook 机制，使得开发者能够在流动性池和交易之间插入自定义的逻辑。在[上一篇文章](https://mp.weixin.qq.com/s/0-xCqBNjQmtizH-78ScA-w)中，我们已经详细介绍了 Hook 的核心概念及其作用。为了帮助大家更好地理解和应用 Hook，本篇将手把手教大家如何从头编写一个 Hook，并通过详细的代码示例和测试用例帮助您快速上手。

## 安装开发工具

编写 Uniswap V4 Hook，首先需要搭建一个开发环境。Uniswap 官方从 V3 开始便推荐使用 Foundry 进行合约开发，Foundry 是一个现代化的智能合约开发工具，基于 Rust 开发，提供编译、测试、运行、部署一站式服务。开发环境使用纯 Solidity ，不引入Javascript或者Python。

Foundry 可以通过多种方式安装，官方提供了详细的安装文档: [https://book.getfoundry.sh/getting-started/installation](https://book.getfoundry.sh/getting-started/installation)。

安装完成后，就可以下一步的工作了。

## 初始化项目

安装好开发环境后，接下来我们使用 Foundry 初始化一个新的项目。初始化命令非常简单：

```bash
forge init v4-hook-demo
cd v4-hook-demo
```

初始化后的项目结构非常完善，包含了 Foundry 的必要配置文件，并且自动生成了 GitHub 的 workflow 文件，这意味着如果您使用 GitHub 托管项目，已经完成了部分持续集成的配置工作。如果您希望将项目推送到 GitHub，可以按照以下步骤操作：

如果需要连接到github，可以先创建一个git项目，然后通过下面的命令同步: 

```bash
git remote add origin git@github.com:32ethers/v4-hook-demo.git
git push --set-upstream origin master
```

这个项目中包含一个简单的合约示例 `Counter.sol`，它演示了如何编写一个基础的智能合约。我们可以浏览一下代码，了解它的结构和功能。如果不需要的话可以将其删除：

```bash
rm ./**/Counter*.sol
```

## 安装依赖

接下来，为了使项目支持 Uniswap V4，我们需要安装相应的依赖库。以下命令将帮助我们安装核心库 `v4-core` 和外设库 `v4-periphery`：

```bash
forge install Uniswap/v4-core
forge install Uniswap/v4-periphery
```

为了简化导入路径，避免在每次编写代码时都写冗长的 `import` 语句，我们可以使用 `forge remappings` 生成一个重定向文件：

```bash
forge remappings > remappings.txt
```

生成的 `remappings.txt` 文件内容如下：

```txt
@ensdomains/=lib/v4-core/node_modules/@ensdomains/
@openzeppelin/=lib/v4-core/lib/openzeppelin-contracts/
@openzeppelin/contracts/=lib/v4-core/lib/openzeppelin-contracts/contracts/
@uniswap/v4-core/=lib/v4-periphery/lib/v4-core/
ds-test/=lib/v4-core/lib/forge-std/lib/ds-test/src/
erc4626-tests/=lib/v4-core/lib/openzeppelin-contracts/lib/erc4626-tests/
forge-gas-snapshot/=lib/v4-core/lib/forge-gas-snapshot/src/
forge-std/=lib/forge-std/src/
hardhat/=lib/v4-core/node_modules/hardhat/
openzeppelin-contracts/=lib/v4-core/lib/openzeppelin-contracts/
permit2/=lib/v4-periphery/lib/permit2/
solmate/=lib/v4-core/lib/solmate/
v4-core/=lib/v4-core/src/
v4-periphery/=lib/v4-periphery/

```

这里有一个小问题需要注意：我们需要将 `v4-core/=lib/v4-core/src/` 修改为 `v4-core/=lib/v4-periphery/lib/v4-core/src/`。这样做可以确保我们编写的 Hook 所依赖的 `v4-core` 版本与 `v4-periphery` 引用的版本保持一致，避免因版本冲突导致的编译错误。

## 设置项目环境

由于 Uniswap V4 引入了“临时存储（Transient Storage）”机制，因此我们需要指定运行环境为即将到来的坎昆硬分叉版本，并确保 Solidity 编译器版本大于 `0.8.24`。您可以在项目根目录的 `foundry.toml` 文件中添加以下三行代码来完成配置：

```toml
solc_version = "0.8.26"
evm_version = "cancun"
ffi = true
```

至此，环境搭建部分已经完成，接下来我们就可以开始编写 Hook 代码。

## 创建hook

在 Uniswap V4 中，Hook 的作用是允许开发者在流动性管理和交易过程中插入自定义逻辑。我们接下来将编写一个简单的 Hook，用于记录每次 `swap` 交易时的操作次数。首先，我们需要新建一个合约文件 `FirstHook.sol`， 然后添加引用。

```solidity
import {BaseHook} from "v4-periphery/src/base/hooks/BaseHook.sol";
import {Hooks} from "v4-core/libraries/Hooks.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {PoolId，PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {BeforeSwapDelta，BeforeSwapDeltaLibrary} from "v4-core/types/BeforeSwapDelta.sol";
```

然后声明一个hook类以及类变量，并初始化构造函数。

```solidity
contract CountingHook is BaseHook {
    using PoolIdLibrary for PoolKey;
    mapping(PoolId => uint256 count) public afterSwapCount;
    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}
}
```

需要注意: 

1. 所有的hook都继承自BaseHook。
2. Hook合约可以被多个pool引用，所以我们的类变量afterSwapCount使用了一个mapping类型，可以针对每个pool分别记录。
3. Hook合约的构造函数必须包含IPoolManager参数，IPoolManager是Hook的管理接口，能进行很多操作。 v4的库也提供了onlyByPoolManager修饰符，限制某些函数只能由管理员操作。

接下来是getHookPermissions()函数，这个函数是必须重载的。它的作用很简单，就是定义启动哪些Hook。我们想在swap交易之后，给计数器加1，所以将afterSwap设置为true，其他设置为false。

```solidity
    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }
```

最后，重载afterSwap函数，让计数器加一。 对于这些函数的说明，见[IHooks.sol](https://github.com/Uniswap/v4-core/blob/main/src/interfaces/IHooks.sol)。

```solidity
function afterSwap(address，PoolKey calldata key，IPoolManager.SwapParams calldata，BalanceDelta，bytes calldata)
    external
    override
    returns (bytes4，int128)
{
    afterSwapCount[key.toId()]++;
    return (BaseHook.afterSwap.selector，0);
}
```

最后贴一下完整的例子。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {BaseHook} from "v4-periphery/src/base/hooks/BaseHook.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {PoolId，PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {BalanceDelta} from "v4-core/types/BalanceDelta.sol";
import {BeforeSwapDelta，BeforeSwapDeltaLibrary} from "v4-core/types/BeforeSwapDelta.sol";

contract CountingHook is BaseHook {
    using PoolIdLibrary for PoolKey;

    mapping(PoolId => uint256 count) public afterSwapCount;

    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}

    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            afterAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: false,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    function afterSwap(address，PoolKey calldata key，IPoolManager.SwapParams calldata，BalanceDelta，bytes calldata)
        external
        override
        returns (bytes4，int128)
    {
        afterSwapCount[key.toId()]++;
        return (BaseHook.afterSwap.selector，0);
    }
}

```

## 创建测试用例

测试用例也是hook开发的重要内容。在为hook开发测试用例时，需要设置测试pool并添加流动性。这里演示如何开发测试用例。

首先添加引用。

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.26;

import {Deployers} from "@uniswap/v4-core/test/utils/Deployers.sol";
import {PoolSwapTest} from "v4-core/test/PoolSwapTest.sol";
import {MockERC20} from "solmate/src/test/utils/mocks/MockERC20.sol";

import {PoolManager} from "v4-core/PoolManager.sol";
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";

import {Currency，CurrencyLibrary} from "v4-core/types/Currency.sol";

import {Hooks} from "v4-core/libraries/Hooks.sol";
import {TickMath} from "v4-core/libraries/TickMath.sol";
import {SqrtPriceMath} from "v4-core/libraries/SqrtPriceMath.sol";
import {LiquidityAmounts} from "@uniswap/v4-core/test/utils/LiquidityAmounts.sol";

import "forge-std/Test.sol";
import {CountingHook} from "../src/FirstHook.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {PoolId} from "v4-core/types/PoolId.sol";
```

然后声明测试类，这个类需要继承Test，由于我们还需要部署虚拟类，还需要继承Deployers类，这个类中提供了很多有用的变量。

```solidity
contract CountingHookTest is Test，Deployers {
    using CurrencyLibrary for Currency;

    CountingHook public hook;

    PoolKey pool_key;
    PoolId pool_id;
    
    Currency token0;
    Currency token1;
}
```

同时，还要声明一些类变量，包括

* hook: hook对象会在setup中初始化，然后供各个测试函数调用。
* pool_key，pool_id: 我们将会创建一个虚拟的pool供测试使用。
* token0，token1: 这是pool的交易对所涉及的两个token。

接下来时setup()函数，它会在每个测试函数执行的时候先执行。我会将每句话的作用注释到代码中: 

```solidity
function setUp() public {
    // 部署虚拟的Manager合约和Router合约
    deployFreshManagerAndRouters();

    // 使用内置的函数部署pool的两个token
    (token0，token1) = deployMintAndApprove2Currencies();

    // 计算hook地址
    uint160 flags = uint160(Hooks.AFTER_SWAP_FLAG);
    address hookAddress = address(flags);

    // 把hook部署到指定地址上，获得hook对象
    deployCodeTo("FirstHook.sol"，abi.encode(manager)，hookAddress);
    hook = CountingHook(hookAddress);

    // 将两种token approve给hook
    MockERC20(Currency.unwrap(token0)).approve(address(hook)，type(uint256).max);
    MockERC20(Currency.unwrap(token1)).approve(address(hook)，type(uint256).max);

    // 初始化pool
    (pool_key，pool_id) = initPool(
        token0，// Currency 0
        token1，// Currency 1
        hook，// Hook对象
        3000，// 设置手续费费率
        SQRT_PRICE_1_1，// 设置初始价格，这里相当于将价格设置为1. 
        ZERO_BYTES // 设置Hook的初始化数据为空，这个参数会被传递给Hook的beforeInitialize和afterInitialize函数
    );

    // 添加流动性，范围是-60 ~ 60，并将流动性设置为一个很大的数字.
    modifyLiquidityRouter.modifyLiquidity(
        pool_key,
        IPoolManager.ModifyLiquidityParams({
            tickLower: -60,
            tickUpper: 60,
            liquidityDelta: 10 ether,
            salt: bytes32(0)
        }),
        ZERO_BYTES //传递给Hook的数据同样为空
    );
}
```

然后是测试函数，必须以test_开头。在这个函数中，我们通过swap交易来触发Hook的执行。

```solidity
function test_swap() public {
    // 设置swap参数
    IPoolManager.SwapParams memory params = IPoolManager.SwapParams({
        zeroForOne: true， // zeroForOne代表swap方向是从token0向token1
        amountSpecified: 0.1 ether，// swap的数量，注意这里是token数量. 
        sqrtPriceLimitX96: MIN_PRICE_LIMIT // swap的价格限制，如果到达这个价格，交易会失败
    });
    PoolSwapTest.TestSettings memory testSettings =
        PoolSwapTest.TestSettings({takeClaims: false，settleUsingBurn: false});
    //检查初始值是0
    assertEq(hook.afterSwapCount(pool_id)，0);
    //进行swap交易
    swapRouter.swap(pool_key，params，testSettings，"");
    //swap之后，计数器变为1
    assertEq(hook.afterSwapCount(pool_id)，1);
}


```

完整的例子在[这里](https://github.com/32ethers/v4-hook-demo/blob/master/test/FirstHook.t.sol)

最后，格式化代码并运行测试用例。

```bash
forge fmt
forge test -vv
```

## 总结

通过本文的讲解，我们已经完成了从环境搭建、依赖安装、项目初始化，到编写 Hook 和测试用例的整个流程。借助 Foundry 的便捷功能，开发 Uniswap V4 Hook 变得简单且高效。Uniswap 官方提供了非常详尽的文档和注释，帮助开发者更好地理解和使用 Hook 机制。

相关资源：

- [Uniswap V4 模板项目](https://github.com/uniswapfoundation/v4-template)
- [Uniswap 例子](https://www.v4-by-example.org/)
- [Periphery 合约](https://github.com/Uniswap/v4-periphery)
- [Core 合约](https://github.com/Uniswap/v4-core)