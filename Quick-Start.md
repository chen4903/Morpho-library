# Quick Start

## 1. 简介

Morpho是一个借贷协议，他是一个无许可的借贷协议，允许任何用户都可以调用合约内部的 `createMarket` 函数使用指定的参数创建交易市场。但是需要注意，Morpho 也并不是完全自由的，用户不可以随意指定一些借贷参数。Market的创建者需要指定：

- a loan token
- a collateral token
- an oracle
- a LLTV (Liquidation Loan-To-Value ratio), which chosen from governance-defined collections(Only can be [0%; 38.5%; 62.5%; 77.0%; 86.0%; 91.5%; 94.5%; 96.5%; 98%] for now)
- IRM(Interest Rate Model), which chosen from governance-defined collections

> For example, one could create the market DAI backed by WETH as collateral with a LLTV of 90%, and using Chainlink as the oracle.

## 2. 治理

- Morpho治理无法停止市场的运营或修改其LLTV、IRM和预言机

- 然而，它有扩大市场创建可用的LLTV和IRM选项范围的能力。

- 在每个市场中，治理可以启用从0%到借款人支付的总利息25%的费率。

## 3. 清算和利率

对于清算，Morpho 也使用了传统的 LTV 方案。我们可以使用以下公式计算某一个仓位的 LTV:
$$
LTV = \frac{Borrow\ Amount * Oracle\ Price}{Collateral\ Amount * Oracle\ Price\ Scale}
$$
述公式内的参数含义如下:

1. `Borrow Amount` 用户借出的资产的数量
2. `Oracle Price` 借出资产的预言机价格
3. `Collateral Amoubnt` 抵押品的数量
4. `Oracle Price Scale` 一个常数，其数值为 `1e36`

> 有读者一定会好奇，这里为什么没有出现抵押品价格的信息。这是因为 Morpho 和大部分现代借贷协议一样采用了相对价格预言机，即预言机返回的价格是一个比价情况。比如用户使用 `DAI` 作为抵押品借出 `USDT` 。那么此处的 `ORACLE PRICE = USDT / DAI`

当用户的资产被清算时，根据 LTV 的情况，用户的抵押品价值实际上仍大于借出资产价值，此时我们是否应该为用户留下部分抵押品？这里为了平衡清算者与借款者之间的利益，Morpho 引入了 **LIF**(Liquidation Incentive Factor):
$$
LIF = \min(maxLIF, \frac{1}{1 - cursor \times (1 - LLTV)})
$$
其中`cursor=0.3`，`maxLIF=1.15`，LIF 因子决定了清算结束后，用户剩余抵押品的数量。

> 举个例子：
>
> 假设用户 A 在 LLTV = 80% 的市场内提供了价值 100 美元的保证金 A。使用上述 LIF 公式计算可得，该市场的 LIF = 1.0638。当用户的借出资产价值大于 80 美元时，用户处于清算状态。此时清算者帮助用户偿还借出资产，可以获得如下数量的保证金:
>
> Seizable Assets = Debt Amount * LIF = 80 * 1.0638 = 85.104
>
> 然后，用户在被清算之后，还剩14.896的抵押品

## 4. 利率模型

在利率模型(IRM)方面，Morpho 要求用户使用的利率模型必须经过治理同意，目前事实上支持一种利率模型。如果读者希望知道 Morpho 启动的利率模型，可以随时使用以下 SQL 代码检索：

```bash
SELECT
  *
FROM morpho_blue_ethereum.MorphoBlue_evt_EnableIrm
ORDER BY
  evt_block_number DESC
```

目前返回结果只有两个 `0x0000000000000000000000000000000000000000` 和 `0x870ac11d48b15db9a138cf899d20f13f79ba00bc`。事实上，当使用零地址作为 IRM 时，等同于放弃借贷利率。

而 `0x870ac11d48b15db9a138cf899d20f13f79ba00bc` 则是 Morpho 主推的知名的 `AdaptiveCurveIRM` 模型。该模型来源于 [PID](https://en.wikipedia.org/wiki/Proportional–integral–derivative_controller) 的思路。然而这部分主要关于复杂的数学，我们可以将其当作黑盒来处理，想要深入学习的读者可以查看此篇出色的[文章](https://blog.wssh.dev/posts/morpho-bule/)。

## 5. 预言机

- Morpho 为了方便用户构建预言机，直接部署一个预言机的工厂合约，其源代码位于 `src/morpho-chainlink/MorphoChainlinkOracleV2Factory.sol` 内部。
- Morpho 的预言机使用一个非常有趣的链式计算的方案。在构造预言机时，用户最多可以指定 4 个预言机地址，分别称为 `baseFeed1` / `baseFeed2` / `quoteFeed1` / `quoteFeed2`。这四个预言机以此提供以下价格数据:
  - `baseFeed1` 提供 `B1` 资产与 `B2` 资产的汇率 `pB1`，即一单位 B1 可以兑换 `pB1`单位 `B2`
  - `baseFeed2` 提供 `B2` 资产与 `C` 资产的汇率 `pB2`，即一单位 B1 可以兑换 `pB2`单位 `C`
  - `quoteFeed1` 提供 `Q1` 资产与 `Q2` 资产的汇率 `pQ1`，即一单位 Q1 可以兑换 `pQ1` 单位 `Q2`
  - `quoteFeed2` 提供 `Q2` 资产与 `C` 资产的汇率 `pQ2`，即一单位 Q2 可以兑换 `pQ2` 单位 `C`

在最终的价格计算过程中，Morpho 将使用以下方法计算最终的价格:
$$
price = \frac{pB_1 \times pB_2}{pQ_1 \times pQ_2}
$$

> 注意，我们可以通过将预言机的地址置为零地址实现将某一个汇率转化为 1。比如将 `quoteFeed2` 设置为零地址，那么 `pQ2` 就会被置为 0。

### 5.1 例子1

直接将某一个 Chainlink 已经给出的预言机转化为 Morpho 中的预言机。比如 [此交易](https://etherscan.io/tx/0x63d68b6d5d0467049271335a3901f2525c0a7c3f74bf3f6e902961630d09d6a5) 创建的 HETH / ETH 的预言机，该交易给出了以下参数：

```bash
"baseFeed1": "0x027A9CFcBB53cbB1721B495c2dcaF54d4cF33106",
"baseFeed2": "0x0000000000000000000000000000000000000000",
"quoteFeed1": "0x0000000000000000000000000000000000000000",
"quoteFeed2": "0x0000000000000000000000000000000000000000",
```

其中 `baseFeed1` 就是 HETH / ETH 的价格预言机。

### 5.2 例子2

将某一个预言机反转。比如将 `USDC / USD` 反转为 `USD / USDC`。比如 [此交易](https://etherscan.io/tx/0x44bfb2f2eb6e05d3dac726e105af609257bf0667de515fca36b0468d15b65462) 创建了一个特殊的预言机，该交易参数如下:

```bash
"baseVault": "0x07D1718fF05a8C53C8F05aDAEd57C0d672945f9a" // arUSD Token address
"baseFeed1": "0x0000000000000000000000000000000000000000",
"baseFeed2": "0x0000000000000000000000000000000000000000",
"quoteFeed1": "0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6", // Oracle of USDC /  USD
"quoteFeed2": "0x0000000000000000000000000000000000000000",
```

上述预言机被用于 [arUSD / USDC](https://app.morpho.org/market?id=0x6c65bb7104ae6fc1dc2cdc97fcb7df2a4747363e76135b32d9170b2520bb65eb&network=mainnet) 金库。此处的 `baseVault` 就是 `arUSD` 的地址。arUSD 是一种基于 ERC4626 的代币。简单来说，arUSD 是用户存入 rUSD 获得的，但 arUSD 是具有利息的，当用户持有 arUSD 越久，兑换回的 rUSD 越多。对于这种基于 ERC4626 的生息代币往往没有预言机，但是这类代币都提供了 `convertToAssets` 计算代币转化为底层资产的比率。在 Morpho 的预言机合约内，自动实现了此变换，当用户指定 `baseVault` 时，Morpho 预言机在计算价格时，会自动转化 ERC4626 代币为底层代币。比如在上述预言机内，当用户输入 `baseVault` 参数时，预言机会自动计算 `arUSD / rUSD` 的比率，然后在于其他预言机等产生的价格相乘或相除。

由于上述预言机使用了将 `quoteFeed1` 置为 `0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6` ，该地址是 `USDC / USD` 的预言机地址。那么最终 Morpho 预言机产生的价格等于 `(arUSD / rUSD) / (USDC / USD)`。这里预言机的创建者实际上假设了 rUSD 不会脱锚，即 `1 rUSD = 1 USD` 恒成立。由此就可以进行以下推导:
$$
\text{price}_{\text{rUSD/USDC}} 
&= \frac{pB_1 \times pB_2}{pQ_1 \times pQ_2} \\
&= \frac{\frac{\text{arUSD}}{\text{rUSD}} \times 1}{\frac{\text{USDC}}{\text{USD}} \times 1} \\
&= \frac{\frac{\text{arUSD}}{\text{USD}} \times 1}{\frac{\text{USDC}}{\text{USD}} \times 1} \quad (1\ \text{rUSD} = 1\ \text{USD}) \\
&= \frac{\text{arUSD}}{\text{USDC}}
$$
实际上更好的方案是指定 `baseFeed1` 为 `rUSD / USD` 的预言机，由此计算的价格更加准确。但是我们也可以理解上述行为，因为没有任何 rUSD 的利益相关方认为 rUSD 会脱锚

### 5.3 例子3

通过连续指定 `baseFeed1` 和 `baseFeed2` 来进行链式预言机价格计算。当然，同时指定 `quoteFeed1` 和 `quoteFeed2` 也可以实现链式的价格计算。这里以 [此交易](https://etherscan.io/tx/0x4d859ffa35d8635b5a448d31fcf568951997efaa3322a78b50548a96db5a9fa7) 为例。该交易构建了 `SolvBTC / USD` 的预言机。该预言机使用了 `SolvBTC / BTC` 和 `BTC / USD` 两个预言机拼接产生，使用了如下参数:

```bash
"baseFeed1": "0x936b31c428c29713343e05d631e69304f5cf5f49", // Oracle of solvBTC / BTC
"baseFeed2": "0xf4030086522a5beea4988f8ca5b36dbc97bee88c", // Oracle of BTC / USD
"quoteFeed1": "0x0000000000000000000000000000000000000000",
"quoteFeed2": "0x0000000000000000000000000000000000000000",
```

推导：
$$
\begin{align*}
\text{price} &= \frac{pB_1 \times pB_2}{pQ_1 \times pQ_2} \\
&= \frac{\left(\frac{\text{solvBTC}}{\text{BTC}}\right) \times \left(\frac{\text{BTC}}{\text{USD}}\right)}{1 \times 1} \\
&= \frac{\text{solvBTC}}{\text{BTC}} \times \frac{\text{BTC}}{\text{USD}} \\
&= \frac{\text{solvBTC}}{\text{USD}}
\end{align*}
$$

### 5.4 例子4

我们可以混合上述多个 Feed 实现更加复杂的预言机构造，比如此交易构建的 wstETH / tBTC 预言机。该交易使用了一下参数:

```bash
"baseFeed1": "0x4F67e4d9BD67eFa28236013288737D39AeF48e79", // Oracle of wstETH / ETH
"baseFeed2": "0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419", // Oracle of ETH / USD
"quoteFeed1": "0x8350b7De6a6a2C1368E7D4Bd968190e13E354297", // Oracle of tBTC / USD
"quoteFeed2": "0x0000000000000000000000000000000000000000",
```

推导：
$$
\begin{align*}
\text{price} &= \frac{pB_1 \times pB_2}{pQ_1 \times pQ_2} \\
&= \frac{wstETH/ETH \times ETH/USD}{tBTC/USD} \\
&= \frac{wstETH/USD}{tBTC/USD} \\
&= wstETH/tBTC
\end{align*}
$$

## 6. 交互

### 6.1 创建市场

需要提供的信息：

```solidity
struct MarketParams {
    address loanToken;
    address collateralToken;
    address oracle;
    address irm;
    uint256 lltv;
}
```

这里需要注意的是选取的IRM利率模型，如果不是0地址，则会调用一下`borrowRate()`做一些初始化、检查工作

```solidity
function createMarket(MarketParams memory marketParams) external {
    Id id = marketParams.id();
    require(isIrmEnabled[marketParams.irm], ErrorsLib.IRM_NOT_ENABLED);
    require(isLltvEnabled[marketParams.lltv], ErrorsLib.LLTV_NOT_ENABLED);
    require(market[id].lastUpdate == 0, ErrorsLib.MARKET_ALREADY_CREATED);

    // Safe "unchecked" cast.
    market[id].lastUpdate = uint128(block.timestamp);
    idToMarketParams[id] = marketParams;

    emit EventsLib.CreateMarket(id, marketParams);

    // Call to initialize the IRM in case it is stateful.
    if (marketParams.irm != address(0)) IIrm(marketParams.irm).borrowRate(marketParams, market[id]);
}

uint256 internal constant MARKET_PARAMS_BYTES_LENGTH = 5 * 32;
function id(MarketParams memory marketParams) internal pure returns (Id marketParamsId) {
    assembly ("memory-safe") {
        marketParamsId := keccak256(marketParams, MARKET_PARAMS_BYTES_LENGTH)
    }
}
```

但是market的fee只能由Morpho的管理员设置，因为默认情况下是没有fee的：

```solidity
/// @inheritdoc IMorphoBase
function setFee(MarketParams memory marketParams, uint256 newFee) external onlyOwner {
    Id id = marketParams.id();
    require(market[id].lastUpdate != 0, ErrorsLib.MARKET_NOT_CREATED);
    require(newFee != market[id].fee, ErrorsLib.ALREADY_SET);
    require(newFee <= MAX_FEE, ErrorsLib.MAX_FEE_EXCEEDED);

    // Accrue interest using the previous fee set before changing it.
    _accrueInterest(marketParams, id);

    // Safe "unchecked" cast.
    market[id].fee = uint128(newFee);

    emit EventsLib.SetFee(id, newFee);
}

/// @inheritdoc IMorphoBase
function setFeeRecipient(address newFeeRecipient) external onlyOwner {
    require(newFeeRecipient != feeRecipient, ErrorsLib.ALREADY_SET);

    feeRecipient = newFeeRecipient;

    emit EventsLib.SetFeeRecipient(newFeeRecipient);
}
```

### 6.2 资产的存款、取款

> 四舍五入的误差给用户，因为要保护协议

用户存钱进来-->累计利息-->取款。下面是利息的计算，值得注意的是，一般的手续费处理方法是直接把手续费记录到一个变量里，直接累加到该存储变量，这种常见方案最大的问题在于将手续费的逻辑完全独立。Morpho采用了另外一种方法：将feeRecipient 视为存款人，而手续费只是为 `feeRecipient` 增加的存款，以此我们可以统一手续费逻辑和存款逻辑。同时`feeRecipient`也相当于享受到了连续复利的好处。

```solidity
/// @dev Accrues interest for the given market `marketParams`.
/// @dev Assumes that the inputs `marketParams` and `id` match.
function _accrueInterest(MarketParams memory marketParams, Id id) internal {
    uint256 elapsed = block.timestamp - market[id].lastUpdate;
    if (elapsed == 0) return;

    if (marketParams.irm != address(0)) {
    		// 当需要累积利息时，我们首先调用 IRM 的 borrowRate 函数获得当前的借款利息，
    		// 然后使用利息计算公式，计算本次利息计算与上一次的利息计算的时间间隔内的利息收益。
    		// 此处是wTaylorCompounded()是连续复利，(e^x)-1
        uint256 borrowRate = IIrm(marketParams.irm).borrowRate(marketParams, market[id]);
        uint256 interest = market[id].totalBorrowAssets.wMulDown(borrowRate.wTaylorCompounded(elapsed));
        market[id].totalBorrowAssets += interest.toUint128();
        market[id].totalSupplyAssets += interest.toUint128();

        uint256 feeShares;
        if (market[id].fee != 0) {
            uint256 feeAmount = interest.wMulDown(market[id].fee);
            // The fee amount is subtracted from the total supply in this calculation to compensate for the fact
            // that total supply is already increased by the full interest (including the fee amount).
            feeShares =
                feeAmount.toSharesDown(market[id].totalSupplyAssets - feeAmount, market[id].totalSupplyShares);
            position[id][feeRecipient].supplyShares += feeShares;
            market[id].totalSupplyShares += feeShares.toUint128();
        }

        emit EventsLib.AccrueInterest(id, borrowRate, interest, feeShares);
    }

    // Safe "unchecked" cast.
    market[id].lastUpdate = uint128(block.timestamp);
}
```

来看看存款函数。Morpho为了降低读取市场设置的 gas ，要求用户调用函数时必须携带 `MarketParams` 结构体。用户可以通过assets和shares来表示自己的存款方式，并且两者必须一个为0，另外一个不为0。计算好之后，进行转移即可。

```solidity
function supply(
    MarketParams memory marketParams,
    uint256 assets,
    uint256 shares,
    address onBehalf,
    bytes calldata data
) external returns (uint256, uint256) {
    Id id = marketParams.id();
    require(market[id].lastUpdate != 0, ErrorsLib.MARKET_NOT_CREATED);
    require(UtilsLib.exactlyOneZero(assets, shares), ErrorsLib.INCONSISTENT_INPUT);
    require(onBehalf != address(0), ErrorsLib.ZERO_ADDRESS);

    _accrueInterest(marketParams, id);

    if (assets > 0) shares = assets.toSharesDown(market[id].totalSupplyAssets, market[id].totalSupplyShares);
    else assets = shares.toAssetsUp(market[id].totalSupplyAssets, market[id].totalSupplyShares);

    position[id][onBehalf].supplyShares += shares;
    market[id].totalSupplyShares += shares.toUint128();
    market[id].totalSupplyAssets += assets.toUint128();

    emit EventsLib.Supply(id, msg.sender, onBehalf, assets, shares);

    if (data.length > 0) IMorphoSupplyCallback(msg.sender).onMorphoSupply(assets, data);

    IERC20(marketParams.loanToken).safeTransferFrom(msg.sender, address(this), assets);

    return (assets, shares);
}
```

取款函数跟存款差不多，反向操作。不过多了一个授权取款的功能，可以看看`setAuthorization()`和`setAuthorizationWithSig()`

```solidity
function _isSenderAuthorized(address onBehalf) internal view returns (bool) {
    return msg.sender == onBehalf || isAuthorized[onBehalf][msg.sender];
}
```

### 6.3 增加、减少抵押品

> 与 AAVE 等借贷协议不同，用户不能直接使用自己存入的资产直接在Morph进行借款，而是需要单独为借款存入担保品(`supplyCollateral`)，且担保品不具有利息收入。而提取担保品需要使用 `withdrawCollateral` 方法。

由于担保品不并记录利息，所以此处也没有使用 share 方案为担保品进行计息操作，而是直接在内部使用 `Position` 中的 `collateral` 字段记录了用户已存入的担保品数量。

```solidity
struct Position {
    uint256 supplyShares;
    uint128 borrowShares;
    uint128 collateral;
}
```

存入担保品的方法非常简单，就检查、转移、记录：

```solidity
function supplyCollateral(MarketParams memory marketParams, uint256 assets, address onBehalf, bytes calldata data)
    external
{
    Id id = marketParams.id();
    require(market[id].lastUpdate != 0, ErrorsLib.MARKET_NOT_CREATED);
    require(assets != 0, ErrorsLib.ZERO_ASSETS);
    require(onBehalf != address(0), ErrorsLib.ZERO_ADDRESS);

    // Don't accrue interest because it's not required and it saves gas.

    position[id][onBehalf].collateral += assets.toUint128();

    emit EventsLib.SupplyCollateral(id, msg.sender, onBehalf, assets);

    if (data.length > 0) IMorphoSupplyCollateralCallback(msg.sender).onMorphoSupplyCollateral(assets, data);

    IERC20(marketParams.collateralToken).safeTransferFrom(msg.sender, address(this), assets);
}
```

提取抵押品类似，镜像操作，而不过增加了健康度检查，对着健康度的公式来实现就可以了。

```solidity
function _isHealthy(MarketParams memory marketParams, Id id, address borrower) internal view returns (bool) {
    if (position[id][borrower].borrowShares == 0) return true;

    uint256 collateralPrice = IOracle(marketParams.oracle).price();

    return _isHealthy(marketParams, id, borrower, collateralPrice);
}

function _isHealthy(MarketParams memory marketParams, Id id, address borrower, uint256 collateralPrice)
    internal
    view
    returns (bool)
{
    uint256 borrowed = uint256(position[id][borrower].borrowShares).toAssetsUp(
        market[id].totalBorrowAssets, market[id].totalBorrowShares
    );
    uint256 maxBorrow = uint256(position[id][borrower].collateral).mulDivDown(collateralPrice, ORACLE_PRICE_SCALE)
        .wMulDown(marketParams.lltv);

    return maxBorrow >= borrowed;
}
```

### 6.4 借款、偿还借款

> 借出的资产需要支付利息，而偿还时也需要支付利息和借款

借款其实很像减少抵押品的操作，需要注意的是健康度检查、授权检查、利息计算

```solidity
function borrow(
    MarketParams memory marketParams,
    uint256 assets,
    uint256 shares,
    address onBehalf,
    address receiver
) external returns (uint256, uint256) {
    Id id = marketParams.id();
    require(market[id].lastUpdate != 0, ErrorsLib.MARKET_NOT_CREATED);
    require(UtilsLib.exactlyOneZero(assets, shares), ErrorsLib.INCONSISTENT_INPUT);
    require(receiver != address(0), ErrorsLib.ZERO_ADDRESS);
    // No need to verify that onBehalf != address(0) thanks to the following authorization check.
    require(_isSenderAuthorized(onBehalf), ErrorsLib.UNAUTHORIZED);

    _accrueInterest(marketParams, id);

    if (assets > 0) shares = assets.toSharesUp(market[id].totalBorrowAssets, market[id].totalBorrowShares);
    else assets = shares.toAssetsDown(market[id].totalBorrowAssets, market[id].totalBorrowShares);

    position[id][onBehalf].borrowShares += shares.toUint128();
    market[id].totalBorrowShares += shares.toUint128();
    market[id].totalBorrowAssets += assets.toUint128();

    require(_isHealthy(marketParams, id, onBehalf), ErrorsLib.INSUFFICIENT_COLLATERAL);
    require(market[id].totalBorrowAssets <= market[id].totalSupplyAssets, ErrorsLib.INSUFFICIENT_LIQUIDITY);

    emit EventsLib.Borrow(id, msg.sender, onBehalf, receiver, assets, shares);

    IERC20(marketParams.loanToken).safeTransfer(receiver, assets);

    return (assets, shares);
}
```

还款其实也差不多，不过多了一个`onMorphoRepay()`回调`msg.sender`

### 6.5 清算

`seizedAssets` 的含义为清算者希望在此处清算中获得用户担保品的数量，而 `repaidShares` 的含义为清算者希望偿还的用户负债 share 的数量。

清算其实也挺简单，分几个部分而已。

计算健康度、清算奖励

```solidity
{
    uint256 collateralPrice = IOracle(marketParams.oracle).price();

    require(!_isHealthy(marketParams, id, borrower, collateralPrice), ErrorsLib.HEALTHY_POSITION);

    // The liquidation incentive factor is min(maxLiquidationIncentiveFactor, 1/(1 - cursor*(1 - lltv))).
    uint256 liquidationIncentiveFactor = UtilsLib.min(
        MAX_LIQUIDATION_INCENTIVE_FACTOR,
        WAD.wDivDown(WAD - LIQUIDATION_CURSOR.wMulDown(WAD - marketParams.lltv))
    );

    if (seizedAssets > 0) {
        uint256 seizedAssetsQuoted = seizedAssets.mulDivUp(collateralPrice, ORACLE_PRICE_SCALE);

        repaidShares = seizedAssetsQuoted.wDivUp(liquidationIncentiveFactor)
            .toSharesUp(market[id].totalBorrowAssets, market[id].totalBorrowShares);
    } else {
        seizedAssets = repaidShares.toAssetsDown(market[id].totalBorrowAssets, market[id].totalBorrowShares)
            .wMulDown(liquidationIncentiveFactor).mulDivDown(ORACLE_PRICE_SCALE, collateralPrice);
    }
}
```

计算好之后，记录一下信息：

```solidity
uint256 repaidAssets = repaidShares.toAssetsUp(market[id].totalBorrowAssets, market[id].totalBorrowShares);

position[id][borrower].borrowShares -= repaidShares.toUint128();
market[id].totalBorrowShares -= repaidShares.toUint128();
market[id].totalBorrowAssets = UtilsLib.zeroFloorSub(market[id].totalBorrowAssets, repaidAssets).toUint128();

position[id][borrower].collateral -= seizedAssets.toUint128();
```

有坏账的话，处理一下：

```solidity
uint256 badDebtShares;
uint256 badDebtAssets;
if (position[id][borrower].collateral == 0) {
    badDebtShares = position[id][borrower].borrowShares;
    badDebtAssets = UtilsLib.min(
        market[id].totalBorrowAssets,
        badDebtShares.toAssetsUp(market[id].totalBorrowAssets, market[id].totalBorrowShares)
    );

    market[id].totalBorrowAssets -= badDebtAssets.toUint128();
    market[id].totalSupplyAssets -= badDebtAssets.toUint128();
    market[id].totalBorrowShares -= badDebtShares.toUint128();
    position[id][borrower].borrowShares = 0;
}
```

最后资产转移和回调函数：

```solidity
IERC20(marketParams.collateralToken).safeTransfer(msg.sender, seizedAssets);

if (data.length > 0) IMorphoLiquidateCallback(msg.sender).onMorphoLiquidate(repaidAssets, data);

IERC20(marketParams.loanToken).safeTransferFrom(msg.sender, address(this), repaidAssets);
```

### 6.6 闪电贷

没什么好说的，简单、优雅、高效

```solidity
function flashLoan(address token, uint256 assets, bytes calldata data) external {
    require(assets != 0, ErrorsLib.ZERO_ASSETS);

    emit EventsLib.FlashLoan(msg.sender, token, assets);

    IERC20(token).safeTransfer(msg.sender, assets);

    IMorphoFlashLoanCallback(msg.sender).onMorphoFlashLoan(assets, data);

    IERC20(token).safeTransferFrom(msg.sender, address(this), assets);
}
```







































