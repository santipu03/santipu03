# Audit Report - dHEDGE Integration GMX

<img src="https://2347102836-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Ff03kK69OTEEthfwi6VoC%2Fuploads%2FDxBkbtjuJRGbqlegKwPU%2FDHEDGE_word_mark_blue.svg?alt=media&token=bf0bd0da-a32d-43be-b912-ecb0679e085d" width="250" height="150" align="left"/>

|                |                                                                           |
| -------------- | ------------------------------------------------------------------------- |
| **Audit Date** | 21/01/2025 - 31/01/2025                                                   |
| **Auditor**    | Santipu ([@santipu_](https://x.com/santipu_)) |
| **Version 1**  | 31/01/2025 Initial Report                                                 |
| **Version 2**  | 19/02/2025 Mitigation Review                                              |

<br clear="both" />

# Disclaimer

_This audit report is based on a time-limited review of the provided code and information. While efforts were made to identify potential issues, it does not guarantee the discovery of all vulnerabilities or the security of the smart contract. The findings are for informational purposes only and should not be considered as investment, legal, or financial advice._

_The audit does not constitute an endorsement or certification of the project. I disclaim any liability for losses or damages arising from the use or reliance on this report, including issues related to smart contract exploitation or changes made after the audit. The project team is responsible for any actions taken based on this report and should seek additional professional advice if needed. By using this report, the team acknowledges and agrees to these terms._


# About Santipu

Santipu is a highly ranked Smart Contract Auditor, primarily conducting audits through [Sherlock](https://www.sherlock.xyz/), while also contributing to platforms like [Code4rena](https://code4rena.com/) and [Cantina](https://cantina.xyz/). With multiple podium finishes in audit contests, including some top-1 placements, Santipu has established a reputation for excellence in the field. Beyond competitions, Santipu has collaborated with leading audit firms, such as [Pashov Audit Group](https://www.pashov.net/) and [Bailsec](https://bailsec.io/).

For inquiries, reach out via [@santipu_](https://x.com/santipu_) on X (formerly Twitter).

# Scope

This review focuses on the new integration between [dHEDGE](https://dhedge.org/) and [GMX](https://gmx.io/#/), specifically examining its unique design and operational constraints.

A defining feature of this integration is that it is restricted to whitelisted Vaults. This limitation arises due to GMX’s withdrawal mechanism, which requires a two-step process, preventing users from withdrawing their full balance in a single transaction.

To mitigate this limitation, dHEDGE has introduced a mechanism where Vault managers maintain a reserve of liquid funds in an alternative token. When a withdrawal request is made, the manager provides the equivalent value of the GMX pool in this other token, ensuring a smoother and more immediate withdrawal experience.

Since this approach requires managers to guarantee sufficient liquidity at all times, the integration is restricted to whitelisted Vaults operated by trusted managers. This safeguard ensures that the system remains functional and does not create liquidity risks for Vault users.

By adopting this model, whitelisted Vaults can leverage GMX to develop yield-generating strategies for their users, while non-whitelisted Vaults remain unaffected by the integration. This selective implementation balances innovation with risk management, preserving the integrity of dHEDGE protocol.

The scope includes the following files:

- `contracts/guards/assetGuards/gmx/GmxPerpMarketAssetGuard.sol`
- `contracts/guards/contractGuards/gmx/GmxExchangeRouterContractGuard.sol`
- `contracts/priceAggregators/ChainlinkPythPriceAggregator.sol`
- `contracts/utils/gmx/GmxPriceLib.sol`

Due to time constraints, the withdrawal flow on `GmxPerpMarketAssetGuard` has been only overviewed and assumed to work correctly in favor of ensuring that the GMX accounting works correctly under all edge cases. 

The PR to review for the audit is [_1005_](https://github.com/dhedge/dhedge-v2/pull/1005).

The mitigation review has been conducted in the following PR: [_1086_](https://github.com/dhedge/dhedge-v2/pull/1086)

**Due to the extensive refactoring and introduction of new code, dHEDGE is advised that a simple Fix Review might not be sufficient to ensure the code’s safety. Therefore, it is recommended that another audit be conducted on the new version of the code.**

# Severity Classification
| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | High      | High       | Medium   |
| **Likelihood: Medium** | High      | Medium      | Low      |
| **Likelihood: Low**    | Medium    | Low         | Low      |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability is discovered and exploited

# Summary

| Severity       | Total | Fixed  |Partially Mitigated | Acknowledged | Disputed |
| -------------- | ----- | ----- | ------|------ | -------- |
| High       | 3     | 3     |  0            | 0        |0
| Medium     | 8     | 7     |  1            | 0        |0
| Low        | 4     | 1     |  1            | 2        |0

<br>

| # | Title | Severity | Status |
| --- | ----- | -------- | ------ |
| [H-01](#H-01)   | DoS on Vault Withdrawals Due to Non-Whitelisted Managers Using GMX      | High | Fixed |
| [H-02](#H-02)   | GMX Balance Does Not Account for Claimable Collateral      | High | Fixed |
| [H-03](#H-03)   | GMX Balance Does Not Account for Fees Incurred by Open Positions | High | Fixed |
| [M-01](#M-01)   | PYTH Oracle Confidence Used Even When Considered Too Low      | Medium | Fixed |
| [M-02](#M-02)   | Missing Support for `cancelDeposit` and `cancelWithdrawal` in Contract Guard  | Medium | Fixed |
| [M-03](#M-03)   | `getMarketLpTokenPrice` Assumes `marketPrice` Will Always Be Positive      | Medium | Fixed |
| [M-04](#M-04)   | Incorrect GMX Token Prices When Underlying Price Feeds Have More Than 8 Decimals      | Medium | Fixed |
| [M-05](#M-05)   | GMX LP Token Prices on dHEDGE Are Hardcoded to `1e18`, Causing Issues on Deposits      | Medium | Fixed |
| [M-06](#M-06)   | Missing Checks for Supported Assets in `claimFundingFees`, `claimCollateral`, and `cancelOrder`      | Medium | Fixed |
| [M-07](#M-07)   | Slippage Accumulation Is Not Reversed When a `MarketSwap` Order Is Canceled      | Medium | Partially Mitigated |
| [M-08](#M-08)   | Inaccurate Leverage Check When There Are Pending Orders      | Medium | Fixed |
| [L-01](#L-01)   | GMX Deposits and Withdrawals Can Be Griefed by Frontrunning Due to Slippage Checks | Low | Acknowledged |
| [L-02](#L-02)   | Double-Counting of Execution Fees on `MarketSwap` Orders        | Low | Fixed |
| [L-03](#L-03)   | Systemic Risks of Untrusted Managers       | Low | Acknowledged |
| [L-04](#L-04)   | GMX Balance Is Always Underestimated  | Low | Partially Mitigated |

# Findings

## High Risk Findings

<a name="H-01"></a>
### [H-01] DoS on Vault Withdrawals Due to Non-Whitelisted Managers Using GMX

**Description:**<br>

Non-whitelisted Vault managers can still add a GMX pool as a supported asset, potentially causing a permanent denial of service (DoS) on withdrawals for all users of the Vault.

Currently, the restriction ensuring only whitelisted Vaults can interact with GMX is enforced within the contract guard (`GmxExchangeRouterContractGuard`):

```solidity
  function _validateTxGuardParams(
    address _poolManagerLogic,
    bytes memory _data
  ) internal view returns (bytes4 method, bytes memory params, address poolLogic, PoolSetting memory poolSetting) {
    poolLogic = IPoolManagerLogic(_poolManagerLogic).poolLogic();
    require(msg.sender == poolLogic, "not pool logic");
    poolSetting = dHedgePoolsWhitelist[poolLogic];
>>  require(poolSetting.poolLogic == poolLogic, "not gmx whitelisted");
    //...
  }
```

However, this check does not prevent a non-whitelisted manager from adding a GMX pool by calling `PoolManagerLogic::changeAssets` directly. When this occurs, users attempting to withdraw will experience a DoS due to a revert in the GMX asset guard (`GmxPerpMarketAssetGuard`):

```solidity
// If withdrawal asset configured for current pool is not enabled, then withdraw should revert
require(IHasSupportedAsset(poolManagerLogic).isSupportedAsset(withdrawAsset), "withdrawal asset not enabled");
```

This restriction is bypassed when the Vault does not initially hold any GMX assets. However, a malicious manager could transfer GMX LP tokens to the Vault, causing all withdrawals to fail due to the enforced revert.

**Recommended Mitigation:**<br>

To prevent this issue, implement additional validation in `PoolManagerLogic::changeAssets` to restrict non-whitelisted Vaults from adding permissioned assets such as GMX.

**Mitigation Review:**<br>

The protocol has fixed the issue by only allowing whitelisted vaults to add permissioned assets such as GMX.

<a name="H-02"></a>
### [H-02] GMX Balance Does Not Account for Claimable Collateral

**Description:**<br>

The `getBalance` function within the GMX asset guard currently calculates the USD value of assets deposited in GMX, including the following holdings:

1. Collateral in GMX positions, factoring in price impact and profit/loss
2. Collateral in pending orders, plus execution fees
3. Claimable funding fees
4. Collateral in a deposit vault
5. LP token balance
6. Market tokens in a withdrawal vault

However, **claimable collateral** is not included in this calculation, even though it should be.

When a position is decreased and the negative price impact is capped, the excess collateral is held in the **claimable collateral pool**. This amount must be manually claimed using the `claimCollateral` function. This feature is described in the [GMX docs](https://github.com/gmx-io/gmx-synthetics/tree/main?tab=readme-ov-file#integration-notes).

Although the `claimCollateral` function is supported within the GMX contract guard, the claimable collateral is not factored into the total GMX balance within the asset guard.

Since claimable collateral is excluded from the balance calculation, the Vault’s total value is underestimated. This discrepancy can lead to:

- Minting more shares than expected for new depositors, diluting existing stakeholders.
- Redeeming fewer assets than expected for users withdrawing from the Vault, resulting in losses.

**Recommended Mitigation:**<br>

The `getBalance` function should track claimable collateral when a position is decreased. The Enzyme Finance protocol addresses this issue using the following approach:

1. For all GMX orders, a fixed callback contract is set to receive a call whenever an order is executed.
2. Upon execution, the callback (`afterOrderExecution`) records a time key and stores it if claimable collateral is nonzero (which occurs mainly on `MarketDecrease` orders).
3. When computing the GMX balance, the system iterates over these time keys to determine the claimable collateral at any given moment.

Implementing a similar mechanism would ensure the Vault’s total value accurately reflects all available assets, preventing share dilution and withdrawal discrepancies.

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the accounting of claimable collateral on `getBalance`.

<a name="H-03"></a>
### [H-03] GMX Balance Does Not Account for Fees Incurred by Open Positions

**Description:**<br>

The `getBalanc`e function tracks assets deposited in GMX and converts them to USD. A key aspect of this function is its tracking of open positions, factoring in profit and loss (PnL) as well as price impact:

```solidity
  // Collateral in the GMX positions, taking into account price impact and profit/loss
  for (uint256 i; i < positionInfos.length; i++) {
    IGmxPosition.PositionInfo memory positionInfo = positionInfos[i];

    uint256 collateralAmount = positionInfo.position.numbers.collateralAmount;
>>  // We use priceImpactDiffUsd + pnlAfterPriceImpactUsd to get the total value of the position
>>  // priceImpactDiffUsd reflects the price of the collateral that will be able to be claimed after withdrawal
>>  // pnlAfterPriceImpactUsd reflects the value of the position after the price impact
>>  // combining those two values we get the most accurate value of the position after withdrawal
    // https://docs.gmx.io/docs/trading/v2#price-impact-rebates
    
    // ...
  }
}
```

However, the function does not account for **fees incurred by open positions**, leading to an overestimation of the remaining collateral.

When GMX calculates the [remaining collateral](https://github.com/gmx-io/gmx-synthetics/blob/main/contracts/position/PositionUtils.sol#L396C1-L400C44
) of a position, it takes into account the PnL, the price impact, and the **fees**. 

```solidity
>>  uint256 collateralCostUsd = fees.totalCostAmount * cache.collateralTokenPrice.min;

    // the position's pnl is counted as collateral for the liquidation check
    // as a position in profit should not be liquidated if the pnl is sufficient
    // to cover the position's fees
    info.remainingCollateralUsd =
        cache.collateralUsd.toInt256()
        + cache.positionPnlUsd
        + cache.priceImpactUsd
>>      - collateralCostUsd.toInt256();
```

Since fees are not deducted in dHEDGE’s calculations, open positions appear more valuable than they actually are. This inflates the value of GMX positions within dHEDGE, resulting in:

- Minting fewer shares than expected for new depositors.
- Redeeming more assets than expected for users withdrawing from the Vault, resulting in losses for the remaining users.
- Drops in Vault share price when GMX positions are closed

**Recommended Mitigation:**<br>

To accurately track open positions, the `getBalance` function should deduct fees from the collateral amount:

```diff
  // Collateral in the GMX positions, taking into account price impact and profit/loss
  for (uint256 i; i < positionInfos.length; i++) {
    // ...
    
    
+   // Subtract the fees incurred by the position from the total collateral
+   if (positionInfo.fees.totalCostAmount < collateralAmount) {
+       collateralAmount -= positionInfo.fees.totalCostAmount;
+   } else {
+       collateralAmount = 0;
+   }
    
    if (collateralAmount > 0) {
      balance = balance.add(
        _assetValue(poolManagerLogic, positionInfo.position.addresses.collateralToken, collateralAmount)
      );
    }
  }
```

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the recommended mitigation. 

## Medium Risk Findings

<a name="M-01"></a>
### [M-01] PYTH Oracle Confidence Used Even When Considered Too Low

**Description:**<br>

The `ChainlinkPythPriceAggregator` contract aggregates price feeds from **Chainlink** and **PYTH** to provide reliable pricing for GMX. It fetches the latest prices, applies validation checks, and returns the freshest price.

As part of this validation, the contract ensures that **PYTH’s confidence value meets a minimum threshold** before using its price. If the confidence level is too low, the PYTH price is marked as **invalid**:

```solidity
// Check that Pyth price confidence meets minimum
if (priceData.price.div(int64(priceData.conf)) < int32(oracleData.offchainOracle.minConfidenceRatio)) {
  invalid = true; // price confidence is too low
}

confidence = uint256(priceData.conf).mul(10 ** (uint256(18 + priceData.expo)));
```

However, **even if the PYTH price is marked as invalid, the confidence value is still used upstream** in `_getPrice`:

```solidity
  function _getPrice() internal view returns (uint256 price, uint256 timestamp, uint256 confidence) {
    (uint256 onchainPrice, uint256 onchainTime) = _getOnchainPrice(); // will revert if invalid
    (
      uint256 offchainPrice,
      uint256 offchainTime,
      bool offchainInvalid,
      uint256 offchainConfidence
    ) = _getOffchainPrice();
>>  confidence = offchainConfidence;
    bool offchain;

    // ...
  }
```

This results in the function `getTokenMinMaxPrice` wrongly incorporating the confidence value in its calculations, even when it has been deemed too low to be reliable:

```solidity
function getTokenMinMaxPrice() external view override returns (IGmxPrice.Price memory priceMinMax) {
    (uint256 price, , uint256 confidence) = _getPrice();

    require(confidence != 0, "Offchain price invalid");

    priceMinMax = IGmxPrice.Price({min: price.sub(confidence).div(1e10), max: price.add(confidence).div(1e10)});
}
```

This flaw causes incorrect price calculations in `getBalance` because it applies an invalid confidence value when determining minimum and maximum prices. As a result:

- The minimum and maximum price ranges are skewed, making GMX pricing unreliable.
- The Vault balance may be incorrectly calculated, affecting deposit and withdrawal valuations.

**Recommended Mitigation:**<br>

To ensure that the confidence value is only used when valid, modify the logic so that it is set only if the confidence threshold is met:

```diff
    // Check that Pyth price confidence meets minimum
    if (priceData.price.div(int64(priceData.conf)) < int32(oracleData.offchainOracle.minConfidenceRatio)) {
      invalid = true; // price confidence is too low
-   }
+   } else {
+     confidence = uint256(priceData.conf).mul(10 ** (uint256(18 + priceData.expo)));
+   }

-   confidence = uint256(priceData.conf).mul(10 ** (uint256(18 + priceData.expo)));
```

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the recommended mitigation.

<a name="M-02"></a>
### [M-02] Missing Support for `cancelDeposit` and `cancelWithdrawal` in Contract Guard

**Description:**<br>

The GMX contract guard (`GmxExchangeRouterContractGuard`) defines a set of **whitelisted functions** that Vault managers can call to interact with GMX. Currently, the following functions are supported:

- `sendTokens`
- `createOrder`
- `claimFundingFees`
- `claimCollateral`
- `cancelOrder`
- `createDeposit`
- `createWithdrawal`

However, two critical functions in the GMX router are missing from this whitelist:

- `cancelDeposit`
- `cancelWithdrawal`

If these functions are not supported, **funds could become locked in GMX** under certain conditions.

When a deposit or withdrawal is initiated in GMX, external factors may prevent execution by keepers. If these actions fail to complete, they must be **canceled manually to recover the locked funds**.

Example Scenario

1. A deposit is created with an overly strict slippage requirement (`minMarketTokens`).
2. Since the keepers cannot execute the deposit due to slippage constraints, the funds remain stuck in the GMX deposit vault.
3. Without the ability to call `cancelDeposit`, the Vault cannot recover its funds.

This issue applies similarly to withdrawals, where execution failures could leave funds stranded in GMX, such as the execution fees.

**Recommended Mitigation:**<br>

To prevent funds from being locked, support should be added for function `cancelDeposit` and `cancelWithdrawal`. This will allow Vault managers to manually cancel stuck deposits and withdrawals, ensuring funds can always be recovered.

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the recommended mitigation.

<a name="M-03"></a>
### [M-03] `getMarketLpTokenPrice` Assumes `marketPrice` Will Always Be Positive

**Description:**<br>

The function `getMarketLpTokenPrice` in the `GmxPriceLib` library calls GMX’s `getMarketTokenPrice` to retrieve the USD price of an LP token. However, it incorrectly **assumes that the price will always be positive**, and blindly casts `marketPrice` from `int256` to `uint256`:

```solidity
  function getMarketLpTokenPrice(
    GmxPriceDependecies memory deps,
    IGmxMarket.Props memory market,
    bool maximize
  ) internal view returns (uint256 lpTokenPriceD18) {
    // marketPrice is in 30 decimals
    (int256 marketPrice, ) = deps.reader.getMarketTokenPrice({
      _dataStore: deps.dataStore,
      _market: market,
      _indexTokenPrice: GmxPriceLib.getTokenMinMaxPrice(deps, market.indexToken),
      _longTokenPrice: GmxPriceLib.getTokenMinMaxPrice(deps, market.longToken),
      _shortTokenPrice: GmxPriceLib.getTokenMinMaxPrice(deps, market.shortToken),
      _pnlFactorType: MAX_PNL_FACTOR_FOR_WITHDRAWALS,
      _maximize: maximize
    });
>>  return uint256(marketPrice).div(1e12); //  convert to 18 decimals
  }
```

However, according to the [GMX documentation](https://github.com/gmx-io/gmx-synthetics?tab=readme-ov-file#market-token-price-1), LP token prices can become negative under certain conditions:

> It is rare but possible for a pool's value to become negative, this can happen since the `impactPoolAmount` and pending PnL is subtracted from the worth of the tokens in the pool

If `marketPrice` is negative, casting it to `uint256` will cause a silent underflow, **converting the value into an extremely large number**. This would lead to:

- Severely inflated Vault balances in `getBalance`.
- New depositors receiving fewer shares than expected, unfairly diluting their stake.
- Withdrawing users receiving excess assets, draining the Vault’s funds.

**Recommended Mitigation:**<br>

Before casting marketPrice to `uint256`, ensure it is non-negative:

```diff
  function getMarketLpTokenPrice(
    GmxPriceDependecies memory deps,
    IGmxMarket.Props memory market,
    bool maximize
  ) internal view returns (uint256 lpTokenPriceD18) {
    // marketPrice is in 30 decimals
    (int256 marketPrice, ) = deps.reader.getMarketTokenPrice({
      _dataStore: deps.dataStore,
      _market: market,
      _indexTokenPrice: GmxPriceLib.getTokenMinMaxPrice(deps, market.indexToken),
      _longTokenPrice: GmxPriceLib.getTokenMinMaxPrice(deps, market.longToken),
      _shortTokenPrice: GmxPriceLib.getTokenMinMaxPrice(deps, market.shortToken),
      _pnlFactorType: MAX_PNL_FACTOR_FOR_WITHDRAWALS,
      _maximize: maximize
    });
-   return uint256(marketPrice).div(1e12); //  convert to 18 decimals
+   return marketPrice <= 0 ? 0 : uint256(marketPrice).div(1e12);
  }
```

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the recommended mitigation. 

<a name="M-04"></a>
### [M-04] Incorrect GMX Token Prices When Underlying Price Feeds Have More Than 8 Decimals

**Description:**<br>

In the `GmxPriceLib` library, when `getTokenMinMaxPrice` is called, the returned prices are expected to be in `30 - tokenDecimals` decimals. This is required because GMX expects all prices to be in this precision, ensuring that when multiplied by token amounts, **the result remains in 30 decimals**.

The function `getTokenMinMaxPrice` retrieves prices using `ChainlinkPythPriceAggregator`, which always returns prices in 8 decimals, and then adjusts them using `adjustPrice`:

```solidity
 function adjustPrice(IGmxDataStore dataStore, uint256 price, address token) internal view returns (uint256) {
    // adjust the prices that are in 8 decimals, to be in (30 - tokenDecimals) decimals
    // https://github.com/gmx-io/gmx-synthetics/blob/main/contracts/oracle/ChainlinkPriceFeedUtils.sol
    uint256 multiplier = dataStore.getPriceFeedMultiplier(token);
    return FullMath.mulDiv(price, multiplier, FLOAT_PRECISION);
 }
```

To convert the 8-decimal price to `30 - tokenDecimals` decimals, it calls `getPriceFeedMultiplier`, which ensures the correct scale.

For example, with USDC (6 decimals):
- `ChainlinkPythPriceAggregator` returns 8-decimal price.
- The multiplier is applied, converting the price to 24 decimals (`30 - 6`), as expected.
    
However, this logic **only works when the underlying price feed also uses 8 decimals**.

Since `ChainlinkPythPriceAggregator` always returns prices in 8 decimals without considering the actual decimals of the underlying feed, the adjusted price will be incorrect if the original price feed has more than 8 decimals.

As a result:
- `getTokenMinMaxPrice` produces incorrectly scaled prices.
- GMX functions return incorrect values due to these flawed price inputs.
- GMX accounting is miscalculated, leading to losses for users

**Recommended Mitigation:**<br>

Modify `ChainlinkPythPriceAggregator` so that it returns prices in the same decimals as the underlying feed, rather than always forcing 8 decimals.

This ensures that when `adjustPrice` is called, the multiplier correctly scales the price to `30 - tokenDecimals`, maintaining accuracy across all tokens.

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the recommended mitigation. 


<a name="M-05"></a>
### [M-05] GMX LP Token Prices on dHEDGE Are Hardcoded to `1e18`, Causing Issues on Deposits

**Description:**<br>

When a Vault manager executes an action within GMX, the contract guard performs slippage checks to ensure that excessive value is not lost due to price fluctuations:

```solidity
    _checkAndUpdateSlippage(
      _poolManagerLogic,
      _to,
      SlippageAccumulator.SwapData({
        srcAsset: latestDeposit.addresses.market, // using USDPriceAggregator
        dstAsset: latestDeposit.addresses.market, // using USDPriceAggregator
        srcAmount: inputTokensValueD18,
        dstAmount: outputTokensValueD18
      })
    );
```

During these checks, the LP token addresses used in the GMX transaction are passed to the `SlippageAccumulator` contract, which retrieves prices from `AssetHandler`. As noted in the comments, GMX LP tokens use `USDPriceAggregator`, which hardcodes their price to `1e18`.

This works fine in the above scenario because the `srcAmount` and `dstAmount` values are already in USD terms. However, this hardcoded price creates issues when LP tokens are used as deposit assets within the dHEDGE Vault.

If a Vault manager enables GMX LP tokens as deposit assets, users will be able to deposit LP tokens into the Vault and receive shares.

Since the LP token price is hardcoded to `1e18`, the **share calculations will be incorrect**, leading to the Vault minting incorrect amount of shares to depositors, causing losses to them or to existing users.

**Recommended Mitigation:**<br>

There are two potential solutions:

1. Correct the LP Token Pricing Mechanism:
Avoid hardcoding the price of GMX LP tokens to `1e18`. Instead, pass a unique and arbitrary address (e.g., `address(123)`) to `SlippageAccumulator` to indicate a USD-pegged asset, ensuring `USDPriceAggregator` is still used when appropriate.

2. Restrict GMX LP Tokens as Deposit Assets
Disallow managers from setting arbitrary tokens as deposit assets. Add validation to ensure only appropriate assets can be used for deposits. Specifically, prevent GMX LP tokens from being used as deposit assets, avoiding incorrect share minting.

**Mitigation Review:**<br>

The protocol has fixed the issue by preventing Vaults from setting GMX asset as a deposit type.

<a name="M-06"></a>
### [M-06] Missing Checks for Supported Assets in `claimFundingFees`, `claimCollateral`, and `cancelOrder`

**Description:**<br>

The functions `claimFundingFees`, `claimCollateral`, and `cancelOrder` in the GMX contract guard lack checks to ensure that the tokens being claimed are still supported in the Vault. As a result, managers can **claim tokens that are no longer included in the Vault's total value**.

Currently, the GMX contract guard does not verify whether the claimed tokens remain supported before executing these functions:

```solidity
  function txGuard(
    address _poolManagerLogic,
    address _to,
    bytes memory _data
  ) external view override returns (uint16 txType, bool) {
    // ...
    if (method == IGmxExchangeRouter.multicall.selector) {
      // ...
>>  } else if (method == IGmxExchangeRouter.claimFundingFees.selector) {
      (, , address receiver) = abi.decode(params, (address[], address[], address));
      require(receiver == poolLogic, "receiver not pool logic");
      txType = uint16(TransactionType.GmxClaimFundingFees);
>>  } else if (method == IGmxExchangeRouter.claimCollateral.selector) {
      (, , , address receiver) = abi.decode(params, (address[], address[], uint256[], address));
      require(receiver == poolLogic, "receiver not pool logic");
      txType = uint16(TransactionType.GmxClaimCollateral);
>>  } else if (method == IGmxExchangeRouter.cancelOrder.selector) {
      txType = uint16(TransactionType.GmxCancelOrder);
    }
    // IGmxExchangeRouter.updateOrder is for limit orders, not supported yet
    return (txType, false);
  }
```

This issue allows malicious Vault managers to exploit the system by **claiming unsupported tokens**, which will not be reflected in the Vault’s total value.

Even if the integration is restricted to whitelisted Vaults, trusted managers could **accidentally trigger this issue**.

Example Scenario:
1. The Vault manager opens a position in GMX using specific tokens.
2. After some time, they close the position and open a new one with different tokens, forgetting to claim the collateral and fees from the previous position.
3. Since the old position is closed, the manager removes the old tokens from the Vault’s supported assets.
4. When the manager claims fees and collateral from the old position, these tokens are sent to the Vault but are not accounted for, since they are no longer supported.
5. The Vault share price drops, as these tokens are excluded from the total Vault value, leading to losses for depositors.

**Recommended Mitigation:**<br>

To mitigate this issue, is recommended to always check that the tokens sent to the Vault are still supported and will be accounted for. 

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing the recommended mitigation.

<a name="M-07"></a>
### [M-07] Slippage Accumulation Is Not Reversed When a `MarketSwap` Order Is Canceled 

**Description:**<br>

When a Vault manager interacts with GMX to create a `MarketSwap` order, the slippage incurred in the swap is tracked and accumulated in the `SlippageAccumulator` contract.

```solidity
    function _checkAndUpdateSlippage(
    address _poolManagerLogic,
    address _to,
    SlippageAccumulator.SwapData memory _swapData
  ) internal {
    // using USDPriceAggregator
    require(_swapData.dstAmount >= _swapData.srcAmount.mul(10_000 - MAX_SLIPPAGE).div(10_000), "high slippage");
>>  slippageAccumulator.updateSlippageImpact(_poolManagerLogic, _to, _swapData);
  }
```

The `SlippageAccumulator` stores the slippage impact and ensures that the total accumulated slippage **does not exceed the maximum allowable threshold**.

However, when a `MarketSwap` order is canceled, the contract guard **does not reverse the recorded slippage**, even though the swap was **never executed**.

**Example Scenario:**
1. A Vault manager submits a `MarketSwap` order with 0.6% slippage.
2. The order fails to execute due to market conditions, so the manager cancels it.
3. The manager then tries to create a new `MarketSwap` order with 1% slippage.
4. The `SlippageAccumulator` reverts the transaction because the accumulated slippage is now 1.6% (0.6% + 1%), exceeding the 1.5% maximum—even though the first swap never executed.

This issue also affects deposits and withdrawals, as they also incur slippage. However, since deposits and withdrawals cannot currently be canceled, they are not immediately impacted. (See `M-02` for details on missing cancellation support for deposits and withdrawals.)

**Recommended Mitigation:**<br>

To mitigate this issue, is recommended to reverse the accumulated slippage incurred in a `MarketSwap` order that has been canceled. 

**Mitigation Review:**<br>

The protocol has partially mitigated the issue by implementing the following mechanism:
1. When the order is submitted, there's a slippage check that takes into account the accumulated slippage and the new incurred slippage. However, the new slippage is not saved. 
2. Only 1 pending order is allowed within the Vault.
3. When the order is executed, the callback is called and the real incurred slippage is saved and accumulated.

However, if the callback is reverted due to having more slippage than allowed, the order on GMX will still be executed and the incurred slippage won't be saved. This can only happen when the prices have changed between submitting the order and executing it, but it's a real possibility that will allow managers to incur more slippage than they're allowed. 

<a name="M-08"></a>
### [M-08] Inaccurate Leverage Check When There Are Pending Orders 

**Description:**<br>

When a Vault manager interacts with GMX to increase or decrease a position, the contract guard includes a check to ensure that the new leverage does not exceed the maximum allowed limit:

```solidity
  function _maxLeverageCheck(Order.Props memory _latestOrder, address _assetHandler) internal view {
    // ...
    // only allow OrderType: MarketDecrease and MarketIncrease, for now
    if (_latestOrder.numbers.orderType == Order.OrderType.MarketDecrease) {
      // ...
      newLeverage = position.numbers.sizeInUsd.sub(_latestOrder.numbers.sizeDeltaUsd).mul(1e18).div(
        position.numbers.collateralAmount.sub(_latestOrder.numbers.initialCollateralDeltaAmount).mul(collateralPrice)
      );
    } else if (_latestOrder.numbers.orderType == Order.OrderType.MarketIncrease) {
      if (position.numbers.collateralAmount.add(_latestOrder.numbers.initialCollateralDeltaAmount) > 0) {
        newLeverage = position.numbers.sizeInUsd.add(_latestOrder.numbers.sizeDeltaUsd).mul(1e18).div(
          position.numbers.collateralAmount.add(_latestOrder.numbers.initialCollateralDeltaAmount).mul(collateralPrice)
        );
      }
    }
    if (newLeverage > currentLeverage) {
>>    require(newLeverage <= MAX_LEVERAGE, "max leverage exceeded");
    }
  }
```

The issue is that this leverage check only considers the latest order and the current position, but **does not account for pending orders** that have not yet been executed.

Example Scenario:
1. A Vault manager has an open position with 3x leverage.
2. The manager opens a market order to increase leverage by 2x. The check passes, as the max leverage is 5x.
3. Before the first order executes, the manager opens another market order to increase leverage by 2x again. The check passes again because the previous order has not yet executed.
4. Once both orders execute, the final leverage becomes 7x, even though the maximum allowed is 5x.
    
This issue allows managers to **bypass the leverage check** and expose users' funds to higher risk. Even trusted managers could accidentally trigger this issue if they submit multiple orders in quick succession.

**Recommended Mitigation:**<br>

To mitigate this issue, is recommended that the leverage check includes the leverage of pending orders that are still not executed. 

**Mitigation Review:**<br>

The protocol has mitigated the issue by implementing an extra safeguard that only allows managers to have one pending order on GMX. 

## Low Risk Findings

<a name="L-01"></a>
### [L-01] GMX Deposits and Withdrawals Can Be Griefed by Frontrunning Due to Slippage Checks

**Description:**<br>

When a manager attempts to deposit or withdraw from GMX, a slippage check is performed to ensure that excessive value is not lost in the transaction. However, an external attacker can exploit this slippage check to prevent the manager from successfully executing a deposit or withdrawal.

Consider the following scenario:

1. The manager intends to withdraw assets by redeeming GMX LP tokens.
2. Before the manager submits the withdrawal request, an attacker frontruns the transaction and transfers LP tokens directly into the withdrawal vault.
3. When the manager submits the withdrawal request, the system calculates the LP tokens to redeem based on the total unrecorded tokens in the withdrawal vault. Since the attacker’s tokens have increased the vault balance, the expected `amountOut` falls below the 1.5% slippage threshold of `amountIn`, causing the transaction to fail.
4. Afterward, the attacker can backrun the transaction to reclaim their own tokens from the withdrawal vault.

This issue arises because the exact amount of LP tokens to withdraw is not explicitly passed as a parameter. Instead, the system relies on the amount of unrecorded tokens in the withdrawal vault, making it vulnerable to this type of manipulation.

```solitity
>>  uint256 marketTokenAmount = withdrawalVault.recordTransferIn(params.market);
```

The exploitability of this issue depends on whether the specific blockchain supports MEV. If MEV is present, an attacker could use this method to prevent a Vault manager from making deposits or withdrawals on GMX.

**Recommended Mitigation:**<br>

Currently, since dHEDGE is only deployed on chains without MEV, no immediate mitigation is necessary. However, this remains a systemic risk that could become exploitable in the future.

**Mitigation Review:**<br>

The issue is acknowledged by the protocol.

<a name="L-02"></a>
### [L-02] Double-Counting of Execution Fees on `MarketSwap` Orders 

**Description:**<br>

When a `MarketSwap` order specifies different supported market addresses in the `market` and `swapPath[0]` fields, the execution fee is counted for both markets, leading to double-counting.

```solidity
  if (
    (order.numbers.executionFee > 0 && order.addresses.market == _asset) ||
    (order.numbers.executionFee > 0 && order.addresses.swapPath[0] == _asset) // swap order
  ) {
    balance = balance.add(_assetValue(poolManagerLogic, guardData.dataStore.wnt(), order.numbers.executionFee));
  }
```

If a `MarketSwap` order has one supported GMX pool in the `market` field and another in `swapPath[0]`, the execution fee is applied to both markets, resulting in double-counting.

This leads to asset inflation in GMX, which negatively impacts new Vault depositors while unfairly benefiting users withdrawing from the Vault.

However, a `MarketSwap` order should ideally have an empty address in the `market` field. Therefore, only a malicious manager could intentionally exploit this issue. Given the assumption that only trusted managers will use this GMX integration, the severity of this issue is considered low.

**Recommended Mitigation:**<br>

To prevent double-counting on `MarketSwap` orders, the execution fee should only be applied when `swapPath[0]` matches the asset:

```diff
    if (
-     (order.numbers.executionFee > 0 && order.addresses.market == _asset) ||
-     (order.numbers.executionFee > 0 && order.addresses.swapPath[0] == _asset) // swap order
+     (order.numbers.executionFee > 0 && !Order.OrderType.MarketSwap && order.addresses.market == _asset) ||
+     (order.numbers.executionFee > 0 && Order.OrderType.MarketSwap && order.addresses.swapPath[0] == _asset) // swap order
    ) {
      balance = balance.add(_assetValue(poolManagerLogic, guardData.dataStore.wnt(), order.numbers.executionFee));
    }
```

Alternatively, a check could be placed at the contract guard that ensures the manager can only set `address(0)` on the `market` field on `MarketSwap` orders. 

**Mitigation Review:**<br>

The protocol has fixed the issue by implementing a check on the contract guard that ensures managers set `address(0)` on the `market` field on `MarketSwap` orders. 

<a name="L-03"></a>
### [L-03] Systemic Risks of Untrusted Managers

**Description:**<br>

As stated at the beginning of this report, the GMX integration is restricted to whitelisted Vaults with trusted managers. This restriction is crucial because if an untrusted manager were able to use the integration, they could inflict significant harm on users, including various Denial-of-Service (DoS) attacks.

For example, a manager could:
- Avoid maintaining a liquidity buffer for the withdrawal token, effectively preventing all Vault withdrawals (causing a DoS).
- Create a large number of pending GMX orders with unrealistic slippage parameters, rendering them non-executable by GMX keepers. This would eventually lead to an Out-of-Gas (OOG) error when `getBalance` is called, causing a DoS.

These are just a few examples, but given the complexity and edge cases within GMX, additional attack vectors likely exist. Because of this, it is a fundamental requirement that this integration remains restricted to trusted managers.

**Recommended Mitigation:**<br>

The attack vectors described are systemic risks of this integration, meaning they cannot be directly fixed but can only be mitigated by ensuring that only trusted managers have access to the GMX integration.

**Mitigation Review:**<br>

The issue is acknowledged by the protocol.

<a name="L-04"></a>
### [L-04] GMX Balance Is Always Underestimated

**Description:**<br>

The `getBalance` function in the GMX asset guard consistently underestimates the value of GMX assets in the Vault. This approach is controversial because it benefits new depositors at the expense of existing Vault users.

Some examples of how GMX assets are underestimated include:

1. **Minimization of LP Token Prices**:

When calculating the price of LP tokens, the function always minimizes their value by passing `false` as the third argument in `getMarketLpTokenPrice`:


```solidity
    if (lpTokenBalance > 0) {
      balance = balance.add(
>>      GmxPriceLib.getMarketLpTokenPrice(priceDependencies, market, false).mul(lpTokenBalance).div(1e18)
      );
    }

    // ...

    balance = balance.add(
      GmxPriceLib
 >>     .getMarketLpTokenPrice(_priceDependencies, _market, false)
        .mul(withdrawal.numbers.marketTokenAmount)
        .div(1e18)
    );
```

2. **Maximization of Collateral Loss Due to PnL and Price Impact**:

The function maximizes potential collateral losses, further underestimating the remaining collateral: 


```solidity
    // use collateralTokenPrice min for loss; similar to the above, GMX protocol prices in favour of itself.
    uint256 lossCollateralAmount = uint256(-positionInfo.pnlAfterPriceImpactUsd).div(
>>    positionInfo.fees.collateralTokenPrice.min
    );
```

3. **Minimization of Extra Collateral Gained from Capped Price Impact**:

The function also minimizes collateral gains from the capped price impact, further underestimating profits:

```solidity
    collateralAmount = collateralAmount.add(
>>    positionInfo.executionPriceResult.priceImpactDiffUsd.div(positionInfo.fees.collateralTokenPrice.max)
    );
```

Overall, the `getBalance` function systematically underestimates the total GMX assets.

This approach benefits new Vault depositors, as they receive more shares due to the Vault’s underestimated total value. However, existing users are negatively affected, as they receive fewer assets per share when withdrawing.

**Recommended Mitigation:**<br>

While this pricing method is not inherently a severe issue, it skews the system in favor of new depositors at the expense of existing ones.

A better approach would be to dynamically adjust asset valuation based on the action being performed. Specifically: 
- **Overestimate the Vault’s value for deposits**, preventing new users from minting an unfairly high number of shares. 
- **Underestimate the Vault’s value for withdrawals**, ensuring that remaining Vault users are not disadvantaged. 

This approach ensures that current Vault users are always protected.

**Mitigation Review:**<br>

The protocol has partially mitigated the issue by using the same `min` and `max` price on the contract guard for its calculations. 
