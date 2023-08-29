
## 源代码结构

涉及核心业务的代码，主要可以分为四个模块

- **InterestRateModel**：利率模型的抽象合约，JumpRateModelV2、JumpRateModel、WhitePaperInterestRateModel 都是具体的利率模型实现。而 LegacyInterestRateModel 的代码和 InterestRateModel 相比，除了注释有些许不同，并没有其他区别。
- **Comptroller**：审计合约，封装了全局使用的数据，以及很多业务检验功能。其入口合约为 Unitroller，也是一个代理合约。
- **PriceOracle**：价格预言机合约，用来获取资产价格的。
- **CToken**：cToken 代币的核心逻辑实现都在该合约中，属于抽象基类合约，没有构造函数，且定义了 3 个抽象函数。CEther 是 cETH 代币的入口合约，直接继承自 CToken。而 ERC20 的 cToken 入口则是 CErc20Delegator，这是一个代理合约，而实际实现合约为 CErc20Delegate 或 CCompLikeDelegate、CDaiDelegate 等。

### InterestRateModel
利率模型

`InterestRateModel.sol`
```

abstract contract InterestRateModel {

/// @notice Indicator that this is an InterestRateModel contract (for inspection)

bool public constant isInterestRateModel = true;

function getBorrowRate(uint cash, uint borrows, uint reserves) virtual external view returns (uint);

function getSupplyRate(uint cash, uint borrows, uint reserves, uint reserveFactorMantissa) virtual external view returns (uint);

}

```

该接口主要包含两个函数，getBorrowRate和getSupplyRate分别表示借贷利息和存款利率。
Compound白皮书的利率模型是在文件`WhitePaperInterestRateModel.sol`，该合约的构造函数如下

```
constructor(uint baseRatePerYear, uint multiplierPerYear) public {

baseRatePerBlock = baseRatePerYear / blocksPerYear;

multiplierPerBlock = multiplierPerYear / blocksPerYear;

emit NewInterestParams(baseRatePerBlock, multiplierPerBlock);

}

```

blocksPeryear：每年产生的区块
baseRatePerYear：每年基本利率
multiplierPerYear：利率年乘数(这个不知道如何称呼，但是这个值在代码中表示的向线性函数的k，而其作用是乘利用率，所以这个变量是将资金利用率的比率权重变大的效果。)

utilizationRate用来计算资金使用率，对应公式为`borrows / (cash + borrows - reserves)`，实现如下

```
function utilizationRate(uint cash, uint borrows, uint reserves) public pure returns (uint) {

// Utilization rate is 0 when there are no borrows

if (borrows == 0) {

return 0;

}

return borrows * BASE / (cash + borrows - reserves);

}
```
cash代表当前资金池里的总资金

getBorrowRate是用来计算当前借贷利率，对应公式为`borrowRate = baseRateYear + utilization * multipierPerBlock`实现如下
```
function getBorrowRate(uint cash, uint borrows, uint reserves) override public view returns (uint) {

uint ur = utilizationRate(cash, borrows, reserves);

return (ur * multiplierPerBlock / BASE) + baseRatePerBlock;

}
```
ur对应资金使用率

getSupplyRate用来计算存款利率，对应公式如下`SupplyRate = borrowRate * ur * (1 - reserveRate)`具体实现如下
```
function getSupplyRate(uint cash, uint borrows, uint reserves, uint reserveFactorMantissa) override public view returns (uint) {

uint oneMinusReserveFactor = BASE - reserveFactorMantissa;

uint borrowRate = getBorrowRate(cash, borrows, reserves);

uint rateToPool = borrowRate * oneMinusReserveFactor / BASE;

return utilizationRate(cash, borrows, reserves) * rateToPool / BASE;

}
```
reserveRate代表保证金率

这种模型如果在几何图形上画出来是直线型的，其对应的方程为 y=k\*x+b

compound还有另一种模型叫做拐点型，当控制使用率在非安全范围内，则会进入拐点型
其getBorrowRate和直线型的不太一样，实现如下
```
function getBorrowRate(uint cash, uint borrows, uint reserves) override public view returns (uint) {

uint util = utilizationRate(cash, borrows, reserves);

if (util <= kink) {

return (util * multiplierPerBlock / BASE) + baseRatePerBlock;

} else {

uint normalRate = (kink * multiplierPerBlock / BASE) + baseRatePerBlock;

uint excessUtil = util - kink;

return (excessUtil * jumpMultiplierPerBlock/ BASE) + normalRate;

}

}
```
当资金使用率大于kink这个安全范围后，其公式为`whiteVGetBorrow + excessUtil * JumpMultiplierPerBlock` 。
whiteVGetBorrow：正常情况下的最大资金使用率
excessUtil：超过了安全范围的资金使用率部分的资金使用率
jumpMultiplierPerBlock：拐点部分的每一个块借贷利率
其数学公式可以写成如下 `y = k * x + b`这里的b代表whiteVGetBorrow，jumpMultiplierPerBlock为k，x是超出部分。
到这里，利率模型大致讲解完了。现在为们准备进入CToken部分

### CToken
CToken是一个基类合约，里面有些函数由上层合约来实现，我们通过观察代码可以知道其实现了CERC20 Token、CEther Token
老样子，我们首先看看初始化函数。
初始化函数主要对CToken的Comptroller、利率模型、汇率、代币基本信息、当前区块号、重入检查参数进行的初始化，实现如下
```
function initialize(ComptrollerInterface comptroller_,

InterestRateModel interestRateModel_,

uint initialExchangeRateMantissa_,

string memory name_,

string memory symbol_,

uint8 decimals_) public {

require(msg.sender == admin, "only admin may initialize the market");

require(accrualBlockNumber == 0 && borrowIndex == 0, "market may only be initialized once");

// Set initial exchange rate

initialExchangeRateMantissa = initialExchangeRateMantissa_;

require(initialExchangeRateMantissa > 0, "initial exchange rate must be greater than zero.");

// Set the comptroller

uint err = _setComptroller(comptroller_);

require(err == NO_ERROR, "setting comptroller failed");

// Initialize block number and borrow index (block number mocks depend on comptroller being set)

accrualBlockNumber = getBlockNumber();

borrowIndex = mantissaOne;

// Set the interest rate model (depends on block number / borrow index)

err = _setInterestRateModelFresh(interestRateModel_);

require(err == NO_ERROR, "setting interest rate model failed");

name = name_;

symbol = symbol_;

decimals = decimals_;

// The counter starts true to prevent changing it from zero to non-zero (i.e. smaller cost/refund)

_notEntered = true;

}
```
由于函数太多，不会全部都介绍，并且因为篇幅限制，源代码不会贴
transferTokens：sender将src的token发送到dst

balanceOfUnderlying：可以获取某个用户的token能够兑换的ctoken

borrowBalanceStoredInternal：获取用户最新的借贷情况

exchangeRateStoredInternal：返回用户最新的汇率

accrueInterest：更新借贷指数、利息、利率、区块号(这个函数很重要，待会讲解一下)

mintFresh：铸造token时，通过铸造的数量来计算要铸造的ctoken，公式为`mintTokens = actualMintAmount / exchangeRate`更新总代币量totalSupply和铸造者拥有的ctoken余额。

redeemFresh（还需要在看看）：清算函数，如果用ctoken还款，计算公式为`redeemTokens = redeemTokensIn & redeemAmount = redeemTokensIn * exchangeRateCurrent`redeemAmount(token)，否则用低层token进行还款，redeemToken公式为`redeemTokens = redeemAmountIn / exchangeRate & redeemAmount = redeemAmountIn`，然后更新totalSupply和accountTokens，再将token发送给还款者。如下
```
/*

* We write the previously calculated values into storage.

* Note: Avoid token reentrancy attacks by writing reduced supply before external transfer.

*/

totalSupply = totalSupply - redeemTokens;

accountTokens[redeemer] = accountTokens[redeemer] - redeemTokens;

/*

* We invoke doTransferOut for the redeemer and the redeemAmount.

* Note: The cToken must handle variations between ERC-20 and ETH underlying.

* On success, the cToken has redeemAmount less of cash.

* doTransferOut reverts if anything goes wrong, since we can't be sure if side effects occurred.

*/

doTransferOut(redeemer, redeemAmount);
```

borrowFresh：更新用户借贷余额和池子里的总借贷余额，并发送借贷的代币

repayBorrowFresh：代偿还贷款，由其他清算者进行还款，先获取最新的借贷金额，如果repayAmount为-1代表还所有的贷款，否则是偿还repayAmount，再更新总借贷和用户借贷

liquidateBorrowFresh（还需要看看）：清算借款人的抵押品，先检查债务、利息等是否是最新数据，偿还的钱是否为0，借贷者是否不等于清算者，然后执行repayBorrowFresh来进行贷款偿还。接着清算者是需要得到欠贷者相应的抵押品，所以需要计算清算者可以获取多少抵押品，计算后将抵押品发送给清算者。

seizeInternal（还需要看看）：将抵押贷币发送给清算者


在阅读CToken代码时，我们可以看到有许多函数一开始都会进入一个comptroller的一个函数，例如
```
uint allowed = comptroller.seizeAllowed(address(this), seizerToken, liquidator, borrower, seizeTokens);
```
然后检查allowed是否为0，所以你在阅读ctoken代码时，一定会很好奇这个文件里面究竟是什么内容。现在我们来看下一个代码
### Comptroller
该合约不仅包含了审计和校验，还包含了compound自身的comp token治理代币的内容，所以有部分内容准备移交到下一节讲述


### Governance
compound的治理是很经典的，比如可以委托他人投票
Comp：主要是用户治理的token
GovernorAlpha：主要治理函数

#### 变量或结构体
numCheckpoints：记录账户投票权重的更改次数
checkpoints：存储每个账户的历史投票权重更改情况（包含最新的）
delegates：每个账户委托人的关系
结构体Receipt：选民的投票收据
#### Comp
\_delegate：
getCurrentVotes：获取某个用户当前投票数
getPriorVotes：获取账户在某个区块中的权重
```
/**

* @notice Determine the prior number of votes for an account as of a block number

* @dev Block number must be a finalized block or else this function will revert to prevent misinformation.

* @param account The address of the account to check

* @param blockNumber The block number to get the vote balance at

* @return The number of votes the account had as of the given block

*/

function getPriorVotes(address account, uint blockNumber) public view returns (uint96) {

require(blockNumber < block.number, "Comp::getPriorVotes: not yet determined");

// 获取最新的投票权重下标
uint32 nCheckpoints = numCheckpoints[account];

if (nCheckpoints == 0) {

return 0;

}

  

// First check most recent balance
// 查看最新的投票权重所在的区块号是否小于等于当前区块号，否则直接返回投票权
if (checkpoints[account][nCheckpoints - 1].fromBlock <= blockNumber) {

return checkpoints[account][nCheckpoints - 1].votes;

}

  

// Next check implicit zero balance
// 查看最初始的投票权重所在的区块是否大于当前区块，则直接返回0
if (checkpoints[account][0].fromBlock > blockNumber) {

return 0;

}

  
// 使用二分法查找要找的区块的投票
uint32 lower = 0;

uint32 upper = nCheckpoints - 1;

while (upper > lower) {

uint32 center = upper - (upper - lower) / 2; // ceil, avoiding overflow

Checkpoint memory cp = checkpoints[account][center];

if (cp.fromBlock == blockNumber) {

return cp.votes;

} else if (cp.fromBlock < blockNumber) {

lower = center;

} else {

upper = center - 1;

}

}

return checkpoints[account][lower].votes;

}
```

delegates：
将自己的投票权委托给其他人，具体实现是多个函数实现
```
function _delegate(address delegator, address delegatee) internal {
// 获取当前被委托者
address currentDelegate = delegates[delegator];
// 获取委托人的投票权
uint96 delegatorBalance = balances[delegator];
// 更新新的被委托人
delegates[delegator] = delegatee;  

emit DelegateChanged(delegator, currentDelegate, delegatee);

_moveDelegates(currentDelegate, delegatee, delegatorBalance);

}


function _moveDelegates(address srcRep, address dstRep, uint96 amount) internal {

if (srcRep != dstRep && amount > 0) {

if (srcRep != address(0)) {
// 获取当前被委托人的更新次数
uint32 srcRepNum = numCheckpoints[srcRep];
// 获取当前被委托人的投票权
uint96 srcRepOld = srcRepNum > 0 ? checkpoints[srcRep][srcRepNum - 1].votes : 0;
// 用被委托人的投票权减去委托人的投票权，得到被委托人的投票权
uint96 srcRepNew = sub96(srcRepOld, amount, "Comp::_moveVotes: vote amount underflows");
// 给被委托人写入新的投票权，
_writeCheckpoint(srcRep, srcRepNum, srcRepOld, srcRepNew);

}

// 这一部分将新的委托人的投票权给新的被委托人
if (dstRep != address(0)) {

uint32 dstRepNum = numCheckpoints[dstRep];

uint96 dstRepOld = dstRepNum > 0 ? checkpoints[dstRep][dstRepNum - 1].votes : 0;

uint96 dstRepNew = add96(dstRepOld, amount, "Comp::_moveVotes: vote amount overflows");

_writeCheckpoint(dstRep, dstRepNum, dstRepOld, dstRepNew);

}

}

}
```

#### GovernorAlpha
这个里面就是治理最基本的实现，代码不贴出来了除非比较难理解。
propose：发起一个新的提案，首先会检查发起提案的人的投票权是否大于1%的Comp，只有大于这个投票权才有机会发起提案，
queue：查看当前提案id的状态是否是投票成功状态，计算提案执行时间，再将提案加入到执行队列中
execute：查看当前提案状态是否已经在队列中的状态，遍历执行在执行队列中的提案内容
cancel：查看当前提案不是执行状态的话往后执行，检查调用者是否为comp的监护者或判断当前提案的投票权是否小于总Comp的1%，满足的话则执行取消提案。
castVote：投票函数，实现是将投票人的投票权重(赞成和反对)进行加减，记录在提案的结构体成员中。

### TimeLock
当某个提案通过投票后，如果没有时间锁，意味着它可以立刻执行，这会导致一定风险的。因为可能大部分治理代币持有者并不会积极参与每个提案，所以攻击者操纵代币执行恶意操纵的难度比大多数人想象的要低。增加时间锁机制正是为了尽可能避免这样的风险，这使得提案通过之后，大家还有一个时间窗口可以用来对恶意提案紧急叫停。

executeTranscation：执行提案里的动作，检查消息发送者是否为admin一般指gov，然后检查当前提案是否被加入到了队列中，再判断执行时间是否小于当前块的时间，执行时间+宽限期是否大于区块的时间，否则会被认定为非常成旧的提案，通过这些判断后就执行提案。

### Proxys
compound的代理合约也值得研究

## 总结
通过上面的分析，对于提案来说发起提案是需要金额大于1%Comp的也就是100000 ether是一个非常大的金额，并且提案在发起后，首先会进入Succeed的状态(判断eta为0则进入这种状态)，然后将提案加入队列中，当提案进入队列后就有机会被执行，

execute，但在执行execute时，会先判断当前执行时间是否在规定内，是否有超过宽限期，如果执行时间大于当前区块时间这个提案会被舍弃，如果超过宽限期说明这个提案已经是比较陈旧的提案了，都满足才能执行提案内容。

对于取消来说，判断状态只要不是正在执行中，就有机会被取消，如果是监护人则可以直接取消，如果提案支持少雨1%Comp，也可以直接取消。

参考：https://mirror.xyz/rbtree.eth/OQEHK-Zrmz2RZZ47KE--UjyjLVetE4yNzH1jQ6KWvKM