MasterChef是一个质押合约，其质押方式被很多其他DeFi项目学习。

# 函数
先对所有函数进行一个简单的介绍
add：添加一个池，并给池赋予allocPoint，allocPoint类似权重的意思
set：将poolInfo的pid对应的某个池换成另一个池，并更新其对应的allocPoint
setMigrator：设置转移目标池，用于后面将lp代币转移到另一个池中
migrate：将某个池的lp代币迁移到新的池，并设置新的lptoken合约地址
getMultiplier：计算从from->to的奖励，正常情况下计算公式为time * BONUS_MULTIPLIER
pendingSushi：计算用户待分发的金额，公式为user.amount * accSushiPerShare - user.rewardDebt (rewardDebt：用户已经领取过的奖励；accSushiPerShare：池中每个lp token代表的累积奖励金额)
massUpdatePools：更新所有池
updatePool：更新每个代币的累加奖励，更新最新的块，铸造代币。
deposit：存款，需要制定某个池和用户，首先更新到池的最新状态，如果用户本身的金额不为0，则先领取待分发的金额发送给用户，然后在将用户的lpToken发送到当前合约中，并更新用户的存款金额amount和已领取的奖励。
withdraw：取款，逻辑跟deposit差不多
emergencyWithdraw：紧急传输，用于紧急情况，将用户的token全部转给用户，并且用户的金额和已领取金额全部归0.
safeSushiTransfer：安全传输
dev：查看发送者是否为dev

# 奖励算法讲解
体现该算法的主要变量是accSushiPerShare，该值会根据上一次的accSushiPerShare计算新的，有很多DeFi项目也是用类似这种方式，我们来看看代码是如何实现的
```
// View function to see pending SUSHIs on frontend.

function pendingSushi(uint256 _pid, address _user)

external

view

returns (uint256)

{

PoolInfo storage pool = poolInfo[_pid];

UserInfo storage user = userInfo[_pid][_user];

uint256 accSushiPerShare = pool.accSushiPerShare;

uint256 lpSupply = pool.lpToken.balanceOf(address(this));

if (block.number > pool.lastRewardBlock && lpSupply != 0) {

uint256 multiplier =

getMultiplier(pool.lastRewardBlock, block.number);

uint256 sushiReward =

multiplier.mul(sushiPerBlock).mul(pool.allocPoint).div(

totalAllocPoint

);

accSushiPerShare = accSushiPerShare.add(

sushiReward.mul(1e12).div(lpSupply)

);

}

return user.amount.mul(accSushiPerShare).div(1e12).sub(user.rewardDebt);

}


// Update reward variables of the given pool to be up-to-date.

function updatePool(uint256 _pid) public {

PoolInfo storage pool = poolInfo[_pid];

if (block.number <= pool.lastRewardBlock) {

return;

}

uint256 lpSupply = pool.lpToken.balanceOf(address(this));

if (lpSupply == 0) {

pool.lastRewardBlock = block.number;

return;

}

uint256 multiplier = getMultiplier(pool.lastRewardBlock, block.number);

uint256 sushiReward =

multiplier.mul(sushiPerBlock).mul(pool.allocPoint).div(

totalAllocPoint

);

sushi.mint(devaddr, sushiReward.div(10));

sushi.mint(address(this), sushiReward);

pool.accSushiPerShare = pool.accSushiPerShare.add(

sushiReward.mul(1e12).div(lpSupply)

);

pool.lastRewardBlock = block.number;

}
```

换算为公式：accSushiPerShare(index) = acSushiPerShare(index - 1) + (shushiPerBlock * poolAllocPoint / totalAllocPoint) / lpSupply
而在最终的计算用户可得金额时，便会用accSushiPerShare来进行乘法，所以我们可以知道acSushiPerShare是一直变大的。这里跟我们之前学习的Compound很像。

# 与Compound不同
