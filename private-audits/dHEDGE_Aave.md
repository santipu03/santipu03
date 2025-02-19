# Audit Report - dHEDGE Integration Aave

<img src="https://2347102836-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Ff03kK69OTEEthfwi6VoC%2Fuploads%2FDxBkbtjuJRGbqlegKwPU%2FDHEDGE_word_mark_blue.svg?alt=media&token=bf0bd0da-a32d-43be-b912-ecb0679e085d" width="250" height="150" align="left"/>

|                |                                                                           |
| -------------- | ------------------------------------------------------------------------- |
| **Audit Date** | 16/01/2025 - 20/01/2025                                                   |
| **Auditor**    | Santipu ([@santipu_](https://x.com/santipu_)) |
| **Version 1**  | 20/01/2025 Initial Report                                                 |
| **Version 2**  | 21/01/2025 Mitigation Review                                              |

<br clear="both" />

# Disclaimer

_This audit report is based on a time-limited review of the provided code and information. While efforts were made to identify potential issues, it does not guarantee the discovery of all vulnerabilities or the security of the smart contract. The findings are for informational purposes only and should not be considered as investment, legal, or financial advice._

_The audit does not constitute an endorsement or certification of the project. I disclaim any liability for losses or damages arising from the use or reliance on this report, including issues related to smart contract exploitation or changes made after the audit. The project team is responsible for any actions taken based on this report and should seek additional professional advice if needed. By using this report, the team acknowledges and agrees to these terms._


# About Santipu

Santipu is a highly ranked Smart Contract Auditor, primarily conducting audits through [Sherlock](https://www.sherlock.xyz/), while also contributing to platforms like [Code4rena](https://code4rena.com/) and [Cantina](https://cantina.xyz/). With multiple podium finishes in audit contests, including some top-1 placements, Santipu has established a reputation for excellence in the field. Beyond competitions, Santipu has collaborated with leading audit firms, such as [Pashov Audit Group](https://www.pashov.net/) and [Bailsec](https://bailsec.io/).

For inquiries, reach out via [@santipu_](https://x.com/santipu_) on X (formerly Twitter).

# Scope

The scope of the review is the new withdrawals version that focuses on [dHEDGE](https://dhedge.org/) Vaults which have open positions in Aave.

The scope includes the following files:

- `contracts/PoolLogic.sol` (limited to the withdrawal flow)
- `contracts/guards/assetGuards/AaveLendingPoolAssetGuard.sol`
- `contracts/swappers/easySwapperV2/EasySwapperV2.sol`
- `contracts/swappers/easySwapperV2/WithdrawalVault.sol`

The PR to review for the audit is [_985_](https://github.com/dhedge/dhedge-v2/pull/985).

The mitigation review has been conducted in the following PR: [1077](https://github.com/dhedge/dhedge-v2/pull/1077)

# Severity Classification
| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | High      | High       | Medium   |
| **Likelihood: Medium** | High      | Medium      | Low      |
| **Likelihood: Low**    | Medium    | Low         | Low      |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability is discovered and exploited

# Summary

| Severity       | Total | Fixed  | Acknowledged | Disputed |
| -------------- | ----- | ----- | ------------ | -------- |
| High       | 0     | 0     |  0            | 0        |
| Medium     | 1     | 1     |  0            | 0        |
| Low        | 5     | 1     |  4            | 0        |

<br>

| # | Title | Severity | Status |
| --- | ----- | -------- | ------ |
| [M-01](#M-01)   | DoS on Withdrawals in Vaults with Aave Positions Due to Rounding Issues      | Medium | Fixed |
| [L-01](#L-01)   | New Withdrawals Mechanism Vulnerable to Griefing by Vault Managers                           | Low | Acknowledged |
| [L-02](#L-02)   | Missing Support for Asset Type 28 on the New Version of `_unrollAssets`                           | Low | Fixed |
| [L-03](#L-03)   | DoS on Vault Withdrawals Due to LTV-0 Aave Collaterals                           | Low | Acknowledged |
| [L-04](#L-04)   | New Withdrawal Mechanism is More Prone to Failure During Market Volatility                           | Low | Acknowledged |
| [L-05](#L-05)   | Flash Loan Check Does Not Account for the Premium                           | Low | Acknowledged |


# Findings

## Medium Risk Findings

<a name="M-01"></a>
### [M-01] DoS on Withdrawals in Vaults with Aave Positions Due to Rounding Issues

**Description:**<br>

When a withdrawal occurs in a dHEDGE Vault with a position in Aave, the Vault repays a portion of its debt and withdraws a corresponding portion of its collateral, transferring the remaining assets to the user.

Both the debt repayment amount and collateral withdrawal amount are rounded down, which can result in a denial of service (DoS) on withdrawals due to the following invariant check:

```solidity
// Invariant state check: actual difference between total vault value before and after withdrawal can not be more than value of portion withdrawn,
// i.e. value of assets which actually left the vault can not be more than value of portion withdrawn.
require(
  execution.fundValue.sub(IPoolManagerLogic(poolManagerLogic).totalFundValueMutable()) <= valueWithdrawn.add(1),
  "value mismatch"
);
```

This check ensures that users cannot withdraw more assets than permitted based on their share of the Vault. However, this invariant check may revert when attempting to withdraw from a Vault with an Aave position.

**Example Scenario:**

Consider a Vault with an Aave position using `cbBTC` as debt and another token as collateral. During a withdrawal, a portion of the debt is repaid, and a portion of the collateral is withdrawn. Since both amounts are rounded down—and given that the debt token `cbBTC` has only 8 decimals—the repaid debt amount is slightly lower relative to the withdrawn collateral.

The rounding down of the debt repayment occurs in the following code snippet:

```solidity
borrowAsset.amount = borrowAsset.amount.mul(_portion).div(10 ** 18);
```

As a result, the Vault’s final value after the withdrawal is slightly lower than expected. This discrepancy triggers the invariant check, which interprets the withdrawal as exceeding the allowed value (due to the under-repayment of debt).

This issue primarily affects Vaults with Aave positions where the debt token has low decimals and high value, such as `wBTC` and `cbBTC`. When the precision loss in the debt token exceeds that of the collateral token, the Vault effectively transfers more value than permitted, causing the invariant check to fail.

**Mitigation Recommendations:**

1. **Allow a buffer in the invariant check**

To account for rounding errors, adjust the invariant check to include a buffer:

```diff
    require(
-     execution.fundValue.sub(IPoolManagerLogic(poolManagerLogic).totalFundValueMutable()) <= valueWithdrawn.add(1),
+     execution.fundValue.sub(IPoolManagerLogic(poolManagerLogic).totalFundValueMutable()) <= valueWithdrawn.add(1e6),
      "value mismatch"
    );
```

While this adjustment prevents the DoS issue, it does not resolve the underlying problem: users can withdraw slightly more value than allowed. This flaw could be exploited by attackers making small withdrawals to gradually reduce the Aave position’s Health Factor, negatively impacting Vault users.


2. **Round Up the Debt Repayment Calculation**

To address the root cause, modify the calculation for the debt repayment amount to round up instead of down:

```diff
-   borrowAsset.amount = borrowAsset.amount.mul(_portion).div(10 ** 18);
+   borrowAsset.amount = borrowAsset.amount.mul(_portion).add(1e18 - 1).div(1e18);
```

By implementing both changes, the system can prevent DoS on withdrawals while ensuring users cannot withdraw more value than allowed, safeguarding the Vault’s integrity and user assets.


**Mitigation Review:**<br>

The protocol has fixed the issue by applying both recommended mitigations.

## Low Risk Findings

<a name="L-01"></a>
### [L-01] New Withdrawals Mechanism Vulnerable to Griefing by Vault Managers

The new mechanism for Vault withdrawals relies heavily on users providing accurate arguments that align with the current state of the Vault.

For instance, when a user invokes the new withdrawal function, they must supply an array of `ComplexAsset` structs. The length of this array must correspond precisely to the number of supported assets in the Vault:

```solidity
  function withdrawToSafe(
    address _recipient,
    uint256 _fundTokenAmount,
>>  IPoolLogic.ComplexAsset[] memory _complexAssetsData
  ) external {
    _withdrawTo(_recipient, _fundTokenAmount, 10_000, _complexAssetsData);
  }
```

If the provided array is shorter than expected, the transaction reverts when attempting to access an out-of-bounds index:

```solidity
  function _withdrawTo(
    address _recipient,
    uint256 _fundTokenAmount,
    uint256 _slippageTolerance,
>>  IPoolLogic.ComplexAsset[] memory _complexAssetsData
  ) internal nonReentrant whenNotFactoryPaused whenNotPaused {
    // ...

    for (uint256 i = 0; i < supportedAssets.length; i++) {
      (address asset, uint256 portionOfAssetBalance, bool externalWithdrawProcessed) = _withdrawProcessing(
        supportedAssets[i].asset,
        _recipient,
        portion,
        _slippageTolerance,
>>      _complexAssetsData[i]
      );
    // ...
  }
```

A manager can exploit this vulnerability by frontrunning a user's withdrawal transaction. By adding a new supported asset to the Vault just before the transaction is executed, the withdrawal transaction will fail.

This type of griefing can occur in other scenarios as well. For example, a manager could frontrun a user's withdrawal by taking actions such as supplying additional collateral or borrowing more debt, causing the user's transaction to revert.

Currently, this issue is only exploitable on chains with a public mempool. Fortunately, dHEDGE is currently deployed on four chains—Arbitrum, Base, Optimism, and Polygon—none of which have a public mempool at present. As such, the issue is currently infeasible under these conditions.

However, if dHEDGE expands to chains where frontrunning is possible, or if one of the existing Layer 2 (L2) chains introduces a public mempool, this vulnerability could become exploitable. Users may then encounter significant difficulties withdrawing from Vaults managed by malicious actors.

This issue should be acknowledged now and revisited as the protocol expands to new chains or if L2 chains adopt public mempools. Proactive measures will be critical to mitigate potential risks to users in the future.

**Mitigation Review:**<br>

The protocol has acknowledged the issue.


<a name="L-02"></a>
### [L-02] Missing Support for Asset Type 28 on the New Version of `_unrollAssets`

In the `WithdrawalVault` contract, the `_unrollAssets` function is responsible for converting assets from a specific dHEDGE Vault into basic ERC20 tokens that can be swapped via decentralized exchanges (DEXes) and subsequently sent to the end user.

As part of this update, a new version of `_unrollAssets` was introduced to decouple this functionality from the soon-to-be-deprecated withdrawal function. However, this new implementation does not support unrolling assets of type 28.

Below is the previous version of `_unrollAssets`, which included support for asset type 28:

```solidity
  /// @dev Kept for backward compatibility. To be removed after transition period
  function _unrollAssets(address _dHedgeVault, uint256 _slippageTolerance) internal {
    // ...

    for (uint256 i; i < supportedAssets.length; ++i) {
      // Velodrome/Aerodrome CL
      else if (assetType == 26) {
        unrolledAssets = EasySwapperVelodromeCLHelpers.getUnsupportedCLAssetsAndRewards(_dHedgeVault, asset);
      }
      // Compound V3 Comet
>>    else if (assetType == 28) {
>>      unrolledAssets = _arraify(SwapperV2Helpers.getCompoundV3BaseAsset(asset));
      }
      // EasySwapperV2UnrolledAssets
      else if (assetType == 30) {
        unrolledAssets = SwapperV2Helpers.getUnrolledAssets(asset, _dHedgeVault);
      }
    }
  }
```

Here is the updated version of `_unrollAssets`, which no longer includes support for asset type 28:

```solidity
  function _unrollAssets(address _dHedgeVault) internal {
    // ...

    for (uint256 i; i < supportedAssets.length; ++i) {
      // ...
      // Velodrome/Aerodrome CL
      else if (assetType == 26) {
        unrolledAssets = EasySwapperVelodromeCLHelpers.getUnsupportedCLAssetsAndRewards(_dHedgeVault, asset);
      }
      // EasySwapperV2UnrolledAssets
      else if (assetType == 30) {
        unrolledAssets = SwapperV2Helpers.getUnrolledAssets(asset, _dHedgeVault);
      }
      // ...
  }
```

The omission of support for asset type 28 in the new implementation creates a significant issue: users will be unable to withdraw from Vaults containing assets of this type using the `WithdrawalVault` contract.

To address this issue, the new version of `_unrollAssets` should be updated to include support for asset type 28. Specifically, the relevant logic from the old implementation should be carried over to the new one.

**Mitigation Review:**<br>

The protocol has fixed the issue by adding support for asset type 28 in the new version of `_unrollAssets`. 

<a name="L-03"></a>
### [L-03] DoS on Vault Withdrawals Due to LTV-0 Aave Collaterals

When a dHEDGE Vault holds a position in Aave and a user initiates a withdrawal from the Vault, the Aave position is proportionally closed based on the amount of shares being withdrawn.

However, this process fails when the Vault contains a collateral token in Aave with a Loan-to-Value (LTV) of 0. This failure occurs because Aave enforces a special rule: when a user holds multiple collaterals and one has an LTV of 0, the user is only allowed to withdraw the LTV-0 collateral. Attempting to withdraw any other collateral results in a transaction failure.

This limitation can lead to a complete denial of service (DoS) for withdrawals from the Vault until the manager manually removes the LTV-0 collateral from Aave.


**Current LTV-0 Tokens in dHEDGE**

Several tokens supported by dHEDGE currently have an LTV of 0 on Aave, including:

- Polygon: BAL, CRV, GHST, SUSHI
- Optimism: LUSD

These tokens should not be allowed to interact with Aave due to their LTV-0 status.

**Recommendations**

1. Remove Asset Type 4 for LTV-0 Tokens
dHEDGE should disallow the interaction of LTV-0 tokens with Aave by removing their classification as asset type 4 within the Vault context. This measure will prevent the inclusion of these tokens in Aave-based positions.

2. Monitor Aave Governance Updates
    To prevent similar issues in the future, dHEDGE should continuously monitor Aave governance updates. Tokens that have their LTV set to 0 should be promptly disabled for Aave interactions.

3. Mitigate Systemic Risks
    If a malicious Vault manager has already deposited a token that later becomes LTV-0, there is currently no mechanism to enforce the withdrawal of these tokens, leaving the Vault vulnerable to a DoS attack. Users should be made aware of this systemic risk when interacting with Vaults that integrate with Aave.

**Mitigation Review:**<br>

The protocol has acknowledged the issue.

<a name="L-04"></a>
### [L-04] New Withdrawal Mechanism is More Prone to Failure During Market Volatility

The new withdrawal mechanism requires users to provide an array of `ComplexAsset` structs containing off-chain routing information for swapping tokens withdrawn from Aave. This user-provided swapping data must align with the current on-chain state, which makes the system vulnerable to failures during volatile markets or high network congestion.

The core issue lies in the delay between when a user queries the data and when the transaction is executed. If market conditions or blockchain states change within this short timeframe, the withdrawal will fail due to outdated information. For example, price fluctuations between query and execution can lead to a mismatch, causing the withdrawal to revert.

To account for minor mismatches, the system includes a buffer parameter called `mismatchDeltaNumerator`:


```solidity
/// @dev Used to calculate tolerance for mismatch between the amounts passed in swap data and the amounts calculated at the moment of execution
uint256 private immutable mismatchDeltaNumerator;
```

This parameter defines the tolerance for discrepancies between the user-provided data and the actual on-chain state. However, because the variable is immutable, it cannot be adjusted dynamically based on current market conditions or blockchain congestion.

To address this issue, it is recommended to make the `mismatchDeltaNumerator` variable mutable. By allowing dynamic adjustments, the system can better accommodate changing circumstances, reducing the likelihood of withdrawal failures during volatile or busy periods.

For example:

- Normal Market Conditions: Set `mismatchDeltaNumerator` to a low value, such as 0.05%.
- Volatile Market Conditions: Increase it to a higher value, such as 1%, to account for greater fluctuations.

**Mitigation Review:**<br>

The protocol has acknowledged the issue and commented the following:

> No code changes for this issue as we don’t set any guards as Ownable. Once we need to change `mismatchDeltaNumerator`, we will re-deploy new instance of `AaveLendingPoolAssetGuard` and set it into Governance contract. It’s still one transaction at multisig, equal to calling a setter on Ownable contract.

<a name="L-05"></a>
### [L-05] Flash Loan Check Does Not Account for the Premium

In the withdrawal process, the Vault utilizes a flash loan from Aave to repay a portion of its debt, enabling the withdrawal of a corresponding portion of the collateral. During this operation, Aave invokes the `executeOperation` function, which includes a check to ensure that the contract does not hold fewer funds than it did before the transaction. This check is intended to prevent funds from being taken from other users to repay the flash loan.

The relevant portion of the code is shown below:

```solidity
  function executeOperation(
    address[] calldata assets,
    uint256[] calldata amounts,
    uint256[] calldata premiums,
    address initiator,
    bytes calldata params
  ) external override returns (bool success) {
    // ...

>>  uint256 withdrawAssetBalanceBefore = IERC20Upgradeable(assets[0]).balanceOf(address(this));

    // ...

    // Liquidation of collateral not enough to pay off debt, flashloan repayment stealing pool's asset
>>  require(withdrawAssetBalanceBefore <= IERC20Upgradeable(assets[0]).balanceOf(address(this)), "high slippage");
  }
```

The current implementation of the check does not account for the flash loan premium, which must also be repaid to Aave as part of the transaction. This oversight creates a vulnerability where an attacker could exploit the mechanism to shift the burden of the flash loan premium onto other users' funds rather than the funds of the user initiating the withdrawal.

While this issue exists, it is currently mitigated by a new invariant check introduced for withdrawals. This check ensures that users cannot withdraw more value from the Vault than they are entitled to, effectively blocking this specific attack. However, relying solely on this invariant could lead to potential issues in the future if additional changes are introduced to the withdrawal logic.

To eliminate this vulnerability, the check in `executeOperation` should be updated to account for the flash loan premium. This change ensures that the funds required to repay both the principal and the premium of the flash loan are not taken from other users' assets. The following modification is recommended:

```diff
    // Liquidation of collateral not enough to pay off debt, flashloan repayment stealing pool's asset
-   require(withdrawAssetBalanceBefore <= IERC20Upgradeable(assets[0]).balanceOf(address(this)), "high slippage");
+   require(withdrawAssetBalanceBefore + premiums[0] <= IERC20Upgradeable(assets[0]).balanceOf(address(this)), "high slippage");
```

This adjustment ensures that the contract's balance after the operation accounts for the premium, preventing any unintended impact on other users' funds.

**Mitigation Review:**<br>

The protocol has acknowledged the issue and commented the following:

> This piece of code `withdrawAssetBalanceBefore.add(premiums[0])` makes `PoolLogic` exceed max size. 
> We’ll add it after transition period and clean up which will offload size, given that invariant prevents the issue. 
