# 限时活动！无奖励挖矿，用显卡给Uniswap v4接生

在11月10日，Uniswap官方发起了一项为[Uniswap v4挖掘地址的活动](https://blog.uniswap.org/uniswap-v4-address-mining-challenge)。
从即日起到12月1日。大家可以计算通过create2方式，计算Uniswap v4的部署地址，这些地址可以通过评价指标得到一个分数，分数最高的地址，将会作为Uniswap V4在主网的地址。简单来说，使用你的设备，进行充分的哈希计算，帮助uniswap 生成 v4的地址。

## 得分规则
这套地址的评分指标有点如下: 

* 每个前缀0得10分.
* 如果前缀0之后的4位是4444，得到40分.
* 如果上述的4444之后不是4，得到20分.
* 如果最后4位是4444，得到20分。
* 地址中每有一个4，就得1分.

举个例子，如果地址是0x00000000044442D64A0BE733A5f2a3187BFA8234，那么

* 9个前缀0得到90分
* 4444得到40分,
* 4444之后没有4，得20分.
* 地址中有6个4，得到6分。

最终可以得到156分。在这套规则下，如果想要高分，地址必须形如0x0000004444XXXXXX...确实是非常有识别度的地址了，堪比seaport的拉风地址。 

另外还有个额外的要求，salt的前20位必须是提交者的地址。

## 如何开始挖矿

Zelos-alpha团队也希望尝试一下。不过我们之前并没有地址挖掘的经验。毕竟，挖矿从来就不是容易的事情，不仅拼脚本，也要拼算力。我们并没有这样的技术储备。好在官方指了一条明路，用create2crunch.

我们了解了一下create2crunch，发现这个小家伙还蛮不错，它是基于rust写的。能够非常充分的利用CPU资源。但我们更看重它支持GPU计算。因为团队新入了一块GTX 4070，可以让它锻炼一下。

但是深入时候后发现create2crunch是一个目的性很强的工具，首先它假定地址都是使用factory合约部署的，因此需要传入factory地址和caller地址，这点和Uniswap的挖矿活动并不符合。另外它实现GPU计算是使用OpenCL框架， 但代码中限制只能使用默认设备。对于我们来说，这个设备是CPU而不是显卡。最严重的问题在挖矿规则上，create2crunch只支持前缀0和全部0的规则，显然这个规则太简单了，和Uniswap的规则相差很大.

在没有更好选择的情况下，我们只能fork项目，自己动手做定制化改进。

首先是地址的问题，我们研究发现，参数中的factory地址实际上是deployer地址，而caller地址实际是salt前缀。因此第一个参数应该是Uniswap提供的deployer地址, 第二个参数是自己地址, 保证salt的前缀是自己: 

```sh
$ cargo run --release 0x48E516B34A1274f49457b9C6182097796D0498Cb [your address] 0x94d114296a5af85c1fd2dc039cdaa32f1ed4b0fe0868f02d888bfc91feb645d9
```

然后我们顺便更改了代码中对应的变量名，避免混淆。

下一个问题是计算平台的问题，为此我们将原来的device id参数扩充了一下，添加了platform id。这样就可以在参数中指定OpenCL的平台和设备了。以我们的工作站的OpenCL设备列表举例

```sh
$ clinfo -l
Platform #0: Intel(R) OpenCL
 `-- Device #0: Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz
Platform #1: NVIDIA CUDA
 `-- Device #0: NVIDIA GeForce RTX 4070
Platform #2: Intel(R) FPGA Emulation Platform for OpenCL(TM)
 `-- Device #0: Intel(R) FPGA Emulation Device
```

显卡的位置在Platform 1，device 0，因此在参数中指定1和0即可.

最终， 程序的入参如下: 

```rust
pub struct Config {
    pub deployer_address: [u8; 20]，// 部署用户的地址
    pub salt_prefix_20: [u8; 20]，// salt的前缀，本次活动中要填写自己的地址
    pub init_code_hash: [u8; 32]，// uniswap合约的init code
    pub platform_id: u8，//OpenCL的platform id
    pub gpu_device: u8，//OpenCL的device id
    pub leading_zeroes_threshold: u8，//前缀0个数，最终可以得到N*2或者N*2+1个前缀0
}
```

现在是最重要的部分，规则和打分。这就需要比较多的修改了。通过分析规则我们发现，如果有N个前缀0,  并保证4444, 形如00004444，可以得到N*10+40, 如果4444之后不是4(事实上很难做到)还能再拿20。而结尾的4444可以不用管，因为如果让结尾是4444，需要保证4位是4才能得到20分(概率是1/16^4)，而如果把算力放到前缀0上，只要保证2位是0就可以得到20分，概率是1/16^2。而规则"4的个数"也不用管，一个4才1分，少几个4对结果影响不大。所以，整体思路是，在保证前缀0之后是4444的情况下，得到尽可能多的前缀0。想来Uniswap制定这样的规则也很合理，毕竟多一个前缀0就能多节约一点gas。日积月累是一笔不小的数字。

现在看一下create2crunch的源代码结构，文件非常的简洁，

```
├── kernels
│   └── keccak256.cl
├── lib.rs
├── main.rs
└── reward.rs
```

其中最主要的文件是lib.rs，主要数据结构和逻辑都在这里了。 而 keccak256.cl是调用OpenCL的源代码，这部分代码并不是rust，幸运的是，这部分代码的语法很像C(应该就是C)，因此即使没接触过OpenCL也应付得过来。

首先要更改的，是keccak256.cl，我们需要将判断前缀0的代码，改成判断前缀0和4444的代码，这样函数只会抛出合格的地址，避免人工再筛选。修改后就成了: 

```c
static inline bool hasLeading(uchar const *d)
{
#pragma unroll
  for (uint i = 0; i < LEADING_ZEROES + 1; ++i) {
    if(i < LEADING_ZEROES){
      if (d[i] != 0) return false;
    }else{
      if (d[i] == 68 && d[i+1]==68) return true;
      if (d[i] == 4 && d[i+1]==68 && ((d[i+2] & 240) == 64 )) return true;
      return false;
    }
  }
  return false;
}
```

在这段代码中，先排除前N位不是0的(注意这里的每一位d[i]的类型是uint8，在地址中占两位，比如N=3实际指000000而不是000)。 如果符合，则判断N+1和N+2是不是44，然后就可以得到形如00 00 00 44 44的地址。然后要注意一个特殊情况，就是N+1当中的第一位也是0， 此时地址形如00 00 00 04 44 4X，这就要通过位运算再判断一下N+1是不是04, N+3的高4位是不是4。 通过这样的改动，GPU就可以通过给定的N，筛选出有N*2或者N*2+1个前缀0的地址。

然后是打分， 这部分代码在lib.rs中,

```rust
            // count total and leading zero bytes
            let mut total = 0;
            let mut leading = 0;
            for (i，&b) in address.iter().enumerate() {
                if b & 240 == 64 {
                    total += 1;
                }
                if b & 15 == 4 {
                    total += 1;
                }
                if b != 0 && leading == 0 {
                    // set leading on finding non-zero byte
                    if b & 240 == 0 {
                        leading = i * 2 + 1;
                    } else {
                        leading = i * 2;
                    }
                }
            }
            let mut tail_is_4444 = 0;
            if address[18] == 68 && address[19] == 68 {
                tail_is_4444 = 1;
            }
            let mut is_not_4444_followed_by_4 = 1;
            if leading % 2 == 0 && (address[leading / 2 + 2] & 240 == 64) {
                is_not_4444_followed_by_4 = 0;
            } else if leading % 2 == 1 && (address[leading / 2 + 2] & 15 == 4) {
                is_not_4444_followed_by_4 = 0;
            }
```

上面的代码做了这样几个事情:

* 在统计4的总个数(total变量)时，b[i]都是地址中的两位，因此要通过位运算分别判断高位和低位是不是4.
* "前缀0"(leading变量)和"4444之后不是4"(is_not_4444_followed_by_4变量)的判断和之前一样，也需要考虑前缀0是奇数个还是偶数个。 
* 结尾的4444(tail_is_4444变量)最好判断。 判断固定位就行.

把上面的汇总就可以得到分数

```rust
let score = leading * 10 + 40 + is_not_4444_followed_by_4 * 20 + tail_is_4444 * 20 + total;
```

程序改好后，使用显卡挖地址的效率非常惊人, 我们的GTX 4070显卡每秒可以尝试18亿次，而18核的CPU只能尝试五千万次。启动后是这个样子, 参数中的1 0 2代表使用Platfom1的Device0, 找有两组前缀0的地址(00 00)或者(00 00 0)

```sh
$ cargo run --release 0x48E516B34A1274f49457b9C6182097796D0498Cb 0x88888414c16cA5793D8D239aBbab2A0e6b358568 0x94d114296a5af85c1fd2dc039cdaa32f1ed4b0fe0868f02d888bfc91feb645d9 1 0 2


Setting up experimental OpenCL miner using device platform 1，device 0...
total runtime: 0:00:7.069246768951416 (190 cycles)                      work size per cycle: 67,108,864
rate: 1800.76 million attempts per second                       total found this run: 5
current search space: 5f4b7ebcxxxxxxxx2f1f5d0e          threshold: 2 leading.
0x88888414c16ca5793d8d239abbab2a0e6b358568493bc5670cdc2802810c8ae0 => 0x000044442a626dC782a8A9038d4c024703DEd20A (4 / 6) 106
0x88888414c16ca5793d8d239abbab2a0e6b358568ba7560035c942e014753d4b6 => 0x00004444bd1648b7792009cD66D33D846AecFB30 (4 / 6) 106
0x88888414c16ca5793d8d239abbab2a0e6b3585689fb0bb1ee3a3320198ea4807 => 0x00004444ED681E1a812b9eF0805e57fd78Fc50A8 (4 / 4) 104
0x88888414c16ca5793d8d239abbab2a0e6b35856865981340dbb8af03a7003c83 => 0x00004444D0852FEcCDa4D0552c0909e92801EE6e (4 / 5) 105
0x88888414c16ca5793d8d239abbab2a0e6b3585686f7109c40f4d2903b8b754aa => 0x00004444bD5DD1a99f749773Bf75128ad79858c8 (4 / 5) 105

```

对于输出的每一行,

> 0x88888414c16ca5793d8d239abbab2a0e6b358568493bc5670cdc2802810c8ae0 => 0x000044442a626dC782a8A9038d4c024703DEd20A (4 / 6) 106

* 0x88888414c16ca5793d8d239abbab2a0e6b358568493bc5670cdc2802810c8ae0是地址的salt
*  0x000044442a626dC782a8A9038d4c024703DEd20A 就是挖出来的地址
* 4/6表示有4个前缀0, 地址中一共有6个4
* 106是这个地址的打分

一切看起来都很完美，直到我们发现，即使用GPU，挖一个有8个前缀0的地址也要很久。理论上说如果要8个前缀0，就需要地址中有12位是确定的(前12位必须是00 00 00 00 44 44)，计算次数是16^12=281474976710656，而我们的算力是1800.76 兆次/秒， 因此找到一个地址理论上需要1.81天，看起来还不错，不是吗?

但是我们的算力显然比不过专业玩家，经过一夜的计算，我们还没算出来地址，而网上已经有人提交9个前缀0的地址了。如果战胜它需要10个前缀0，那就是1.81 \* 16 \* 16天，算了算了不玩了。

最后我们将项目的代码开源，[https://github.com/32ethers/create2crunch](https://github.com/32ethers/create2crunch)，并附上了详细的使用方法。如果你感兴趣, 并且有强大显卡。可以去挑战下。实时结果可以在[活动网站](https://v4-address.uniswap.org/)看到 。

## 挖矿奖励
给uniswap v4 接生，cool！

## 热知识：为什么要搞一个0x0000000 的地址

这实际上要追溯到eth 的黄皮书，0x000会降低gas 消耗.
能省多少的统计可以参考 https://medium.com/@solidity101/unraveling-the-gas-saving-magic-understanding-addresses-with-leading-zeros-in-solidity-smart-7a853573b116
不难推测项目方和链上的高频交易对这样的地址都有需求。比如当年的GasToken的合约地址就很多0x0开头。
此事在wintermute hack中亦有记载：https://www.forbes.com/sites/jeffkauflin/2022/09/20/profanity-may-be-the-cause-of-crypto-trading-firm-wintermutes-160-million-hack/