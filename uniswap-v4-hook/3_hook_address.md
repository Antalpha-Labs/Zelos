# Hook的地址

在上一篇文章中, 我们在测试用例中部署了一个hook, 这个hook被部署到了特定的地址"0x0000000000000000000000000000000000000040", 正是通过这个特殊的地址, PoolManager可以在Hook中调用after_swap函数.  

```solidity
        uint160 flags = uint160(Hooks.AFTER_SWAP_FLAG);
        address hookAddress = address(flags);
        deployCodeTo("FirstHook.sol", abi.encode(manager), hookAddress);
```

具体来说, EVM链的地址实际上是uint160, "0x0000000000000000000000000000000000000040"用二进制可以表示为:  

```
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 
0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 
0000 0000 0000 0000 0000 0000 0100 0000
```

其中第七位是1,  PoolManager在执行swap操作时, 会查看hook地址的第七位, 如果是1, 就会在每笔swap交易之后执行hook的after_swap操作.  与此类似, 当PoolManager执行其他流程的时候, 也会查看地址对应的标志位, 并决定执行哪个函数. 

Hook.sol中记录了每个标志位对应的函数, 如AFTER_SWAP_FLAG=1<<6, 相当于二进制的```0100 0000```, 相当于第7位是1: 

```solidity
    uint160 internal constant ALL_HOOK_MASK = uint160((1 << 14) - 1);

    uint160 internal constant BEFORE_INITIALIZE_FLAG = 1 << 13;
    uint160 internal constant AFTER_INITIALIZE_FLAG = 1 << 12;

    uint160 internal constant BEFORE_ADD_LIQUIDITY_FLAG = 1 << 11;
    uint160 internal constant AFTER_ADD_LIQUIDITY_FLAG = 1 << 10;

    uint160 internal constant BEFORE_REMOVE_LIQUIDITY_FLAG = 1 << 9;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_FLAG = 1 << 8;

    uint160 internal constant BEFORE_SWAP_FLAG = 1 << 7;
    uint160 internal constant AFTER_SWAP_FLAG = 1 << 6;

    uint160 internal constant BEFORE_DONATE_FLAG = 1 << 5;
    uint160 internal constant AFTER_DONATE_FLAG = 1 << 4;

    uint160 internal constant BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3;
    uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2;
    uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1;
    uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0;
```

如果要激活多个hook函数, 可以在生成地址的时候, 对多个地址标志位使用并集. 比如```uint160(AFTER_ADD_LIQUIDITY_FLAG|BEFORE_SWAP_FLAG|AFTER_SWAP_FLAG)```, 这会让地址形如```... 0100 1100 0000```,  也就是0x00000000000000000000000000000000000004c0. 

最终部署的时候, 需要先create2计算出合适的地址, 然后部署到这个地址上. 

还有一个额外的问题, 既然PoolManager能够通过地址确定调用Hook的哪个函数, 为什么还要在Hook中写重载一个getHookPermissions函数? 实际上这个函数有两个用途, 首先是便捷且明确的告知这个Hook插入了哪些函数, 其次是起到校验的作用. 在部署Hook时, 会校验Hook的地址是否与期待的权限相符, 确保Hook地址包含对应的标志位. 在在Hook.sol的注释说明了这一点: 

```solidity
    /// @notice Returns a struct of permissions to signal which hook functions are to be implemented
    /// @dev Used at deployment to validate the address correctly represents the expected permissions
    function getHookPermissions() public pure virtual returns (Hooks.Permissions memory);
```

这样做的目的可能是为了节约gas. PoolManager调用Hook是通过合约地址+函数签名. 如果在每个Hook的插入点都查询一下Hook是否有对应的函数, 会使用很多gas, 这对于频繁的交易是不可接受的. 相反, 检测一个int的标记位则很简单, 只要一个交运算就可以了.  虽然这样会让部署合约有些麻烦, 但这并不是一个大问题. 