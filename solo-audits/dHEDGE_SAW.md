# Audit Report - dHEDGE

<img src="https://2347102836-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Ff03kK69OTEEthfwi6VoC%2Fuploads%2FDxBkbtjuJRGbqlegKwPU%2FDHEDGE_word_mark_blue.svg?alt=media&token=bf0bd0da-a32d-43be-b912-ecb0679e085d" width="250" height="150" align="left"/>

|                |                                                                           |
| -------------- | ------------------------------------------------------------------------- |
| **Audit Date** | 08/010/2024 - 12/10/2024                                                   |
| **Auditor**    | Santipu ([@santipu_](https://x.com/santipu_)) |
| **Version 1**  | 12/10/2024 Initial Report                                                 |
| **Version 2**  | 15/10/2024 Mitigation Review                                              |

<br clear="both" />

# Disclaimer

_This audit report is based on a time-limited review of the provided code and information. While efforts were made to identify potential issues, it does not guarantee the discovery of all vulnerabilities or the security of the smart contract. The findings are for informational purposes only and should not be considered as investment, legal, or financial advice._

_The audit does not constitute an endorsement or certification of the project. I disclaim any liability for losses or damages arising from the use or reliance on this report, including issues related to smart contract exploitation or changes made after the audit. The project team is responsible for any actions taken based on this report and should seek additional professional advice if needed. By using this report, the team acknowledges and agrees to these terms._


# About Santipu

Santipu is a highly ranked Smart Contract Auditor, primarily conducting audits through [Sherlock](https://www.sherlock.xyz/), while also contributing to platforms like [Code4rena](https://code4rena.com/) and [Cantina](https://cantina.xyz/). With multiple podium finishes in audit contests, including some top-1 placements, Santipu has established a reputation for excellence in the field. Beyond competitions, Santipu has collaborated with leading audit firms, such as [Pashov Audit Group](https://www.pashov.net/) and [Bailsec](https://bailsec.io/).

For inquiries, reach out via [@santipu_](https://x.com/santipu_) on X (formerly Twitter).

# Scope

The scope of the audit is the new version for Single-Asset Withdrawals within the [dHEDGE](https://dhedge.org/) protocol. This scope includes the following files:

- `WithdrawalVault`
- `EasySwapperV2` (limited to the withdrawal flow)

The commit for the initial audit is [_da0e4216ebb678f012c1f36ccbcd60a1a5e7c600_](https://github.com/dhedge/dhedge-v2/tree/da0e4216ebb678f012c1f36ccbcd60a1a5e7c600).

The mitigation review has been conducted in the following PR: https://github.com/dhedge/dhedge-v2/pull/1004

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
| High       | 3     | 2     |  1            | 0        |
| Medium     | 3     | 1     |  2            | 0        |
| Low        | 2     | 2     |  0            | 0        |

<br>

| # | Title | Severity | Status |
| --- | ----- | -------- | ------ |
| [H-01](#H-01)   | Vault Manager Can Block All Withdrawals by Creating a `WithdrawalVault` with a Malicious Asset            | High   | Fixed  |
| [H-02](#H-02)   | Anyone Can Block All Withdrawals by Regularly Depositing Funds in a Nested Vault                           | High | Acknowledged        |
| [H-03](#H-03)   | Missing Validation of `_swapData` Allows Vault Manager to Steal Funds                           | High | Fixed        |
| [M-01](#M-01)   | Lack of Support for Asset Type 30 Prevents Single-Asset Withdrawals When the Vault Holds Shares of Another Vault      | Medium | Fixed |
| [M-02](#M-02)   | Vault Owner Can Prevent Another Vault from Selling Shares by Adding an Unsupported Asset Type                           | Medium | Acknowledged |
| [M-03](#M-03)   | Missing Slippage Checks Allow Manager Exploitation When Using `EasySwapperV2` to Sell Shares of a Nested Vault | Medium | Acknowledged |
| [L-01](#L-01)   | Function `_recoverAllAssets` Does Not Clean Up `srcAssets` When Balance is Zero                           | Low | Fixed |
| [L-02](#L-02)   | Missing Slippage Check on `initWithdrawal` Risks Value Loss When Withdrawing from Nested Vaults      | Low | Fixed |

# Findings
## High Risk Findings

<a name="H-01"></a>
### [H-01] Vault Manager Can Block All Withdrawals by Creating a `WithdrawalVault` with a Malicious Asset
**Description:**<br>

The manager of a Vault has the ability to use `EasySwapperV2` to buy and sell shares of other dHEDGE Vaults. When the manager initiates a withdrawal of shares from another Vault, the function `initWithdrawal` is called, which deploys a `WithdrawalVault` to receive and unroll the withdrawn assets.

Assets held in the `WithdrawalVault` still contribute to the total value of the original Vault to maintain the share price. When users request withdrawals from the Vault, they are entitled to a pro-rata share of all the Vault's assets, including those in the `WithdrawalVault`. As part of the withdrawal process, `EasySwapperV2::partialWithdraw` is called to transfer the correct portion of assets to the user.

However, a vulnerability in `EasySwapperV2` allows a malicious Vault manager to cause a permanent denial-of-service (DoS) on all withdrawals through the following steps:
1. The manager calls `EasySwapperV2::initWithdrawal`, providing a malicious contract address as the `_dHedgeVault` argument.
2. The malicious contract behaves like a dHEDGE Vault but does not perform any actions when the functions `safeTransferFrom` and `withdrawToSafe` are called. When `unrollAssets` is executed, the malicious contract returns a list of "supported assets" that includes an ERC20 token designed to revert on transfers.
3. When a user attempts to withdraw from the Vault, `partialWithdraw` triggers a call to `recoverAssets`, which fails when attempting to transfer the malicious ERC20 token.

At this point, all withdrawals from the Vault are effectively blocked because the `WithdrawalVault` cannot transfer the problematic ERC20 token, which consistently reverts during the transfer attempt. As a result, no users can withdraw their funds from the Vault.

**Recommendation:**<br>

To mitigate this issue, it is recommended to implement a validation check in `EasySwapperV2::initWithdrawal` to ensure that the `_dHedgeVault` argument is indeed a legitimate dHEDGE Vault, rather than a malicious contract that could disrupt the withdrawal process.

**Mitigation Review:**<br>

The issue has been fixed at https://github.com/dhedge/dhedge-v2/commit/607ffd88b277586297dac088a6a484ea40eefee2.

<a name="H-02"></a>
### [H-02] Anyone Can Block All Withdrawals by Regularly Depositing Funds in a Nested Vault

**Description:**<br>

The manager of a Vault can use `EasySwapperV2` to buy and sell shares in other Vaults. When a user requests a Single-Asset Withdrawal (SAW) to sell shares, and the Vault holds shares in another Vault, the function `unrollAssets` attempts to unroll the nested Vault’s shares.

This process is handled by `WithdrawalVault::_detectDhedgeVault`:

```solidity
  function _detectDhedgeVault(
    address _asset,
    address _poolFactory,
    uint256 _slippageTolerance
  ) internal returns (address[] memory unrolledAssets) {
    if (IPoolFactory(_poolFactory).isPool(_asset)) {
      uint256 balance = IPoolLogic(_asset).balanceOf(address(this));
      if (balance > 0) {
        IPoolLogic(_asset).withdrawSafe(balance, _slippageTolerance);
        _unrollAssets(_asset, _slippageTolerance);
      }
    } else {
      unrolledAssets = _arraify(_asset);
    }
  }
```

However, withdrawing from any dHEDGE Vault requires that the cooldown period has elapsed, a restriction enforced in `PoolLogic::_beforeTokenTransfer`:

```solidity
  function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual override {
    // ...
    require(getExitRemainingCooldown(from) == 0, "cooldown active");
  }
```

If a Vault holds shares in another Vault that are still under the cooldown period, users will be unable to withdraw. Here’s how this issue can manifest:
1. A user wants to withdraw shares from `Vault1`.
2. The manager of `Vault1` had previously acquired shares in `Vault2`, resetting the cooldown (typically to one day).
3. When the user tries to perform a Single-Asset Withdrawal, the `WithdrawalVault` attempts to redeem shares from `Vault2`, but this fails because the cooldown has not yet expired.
4. Consequently, withdrawals from `Vault1` are blocked until the cooldown for `Vault2` expires.

This vulnerability can be exploited by any attacker, as anyone can deposit a minimal (dust) amount of assets into `Vault2` on behalf of `Vault1`, continually resetting the cooldown and preventing users from withdrawing their funds from `Vault1`. This issue also can be exploited by a malicious Vault manager, which can withdraw and deposit funds in `Vault2` every 24 hours, thus resetting the cooldown and blocking withdrawals from `Vault1`. 

Furthermore, the issue affects not just Single-Asset Withdrawals but also normal withdrawals, as users will be unable to redeem shares from `Vault1` due to the active cooldown in `Vault2`. The root cause of this issue is a feature enforced in the dHEDGE core contracts, but it can be leveraged to cause an almost permanent DoS on all Vault withdrawals. 

**Recommendation:**<br>

Addressing this issue is challenging because once a Vault (`Vault1`) acquires shares in another Vault (`Vault2`), any party can keep depositing a small amount of assets in `Vault2`, thereby extending the cooldown and effectively causing a denial-of-service (DoS) on all withdrawals from `Vault1`.

There is no simple fix because the root cause is embedded in the current dHEDGE system. 

**Mitigation Review:**<br>

This issue has been acknowledged because of the existing mitigations in place. The protocol commented:

> Yes, this is all true, but we’ve been living with it and accepting this risk. 
> 
> Actually, managers can’t acquire shares of other vaults with a one day cooldown. For such scenarios custom cooldown exists, which initially was 5 minutes, then increased to 15 minutes, and now it’s set to 60 minutes. This logic is baked into EasySwapperV2 (and V1) deposits flow. Manager can’t acquire shares of other vaults directly using PoolLogic contract, because no contract guard exists for it. They have to use EasySwapperV2 (or V1), because there are contract guards EasySwapperV2ContractGuard (or EasySwapperGuard), which allow depositing into other vaults, and as you might have seen, it allows for managers to use only deposit functions postfixed -withCustomCooldown. These functions in its turn work only on whitelisted vaults, which are set by the owner to be deposited with custom (lower) lockup time.

With the cooldown for Vaults investing in other Vaults set to 60 minutes instead of 24 hours, and given the griefing nature of the issue, it is difficult to exploit and provides no incentive for attackers beyond causing disruption, as there is no financial gain to be made. This reduces the overall impact of the issue

<a name="H-03"></a>
### [H-03] Missing Validation of `_swapData` Allows Vault Manager to Steal Funds

**Description:**<br>

A Vault manager can use the function `EasySwapperV2::completeWithdrawal` to execute a swap, converting all tokens in `WithdrawalVault` into a single token and sending it back to the Vault. However, due to the lack of validation for the `_swapData` argument, a manager can manipulate this process to include a malicious token and steal the funds held in the `WithdrawalVault`.

The attack path is as follows:
1. The manager of `Vault1` acquires shares from another dHEDGE Vault (`Vault2`).
2. The manager uses `EasySwapperV2` to withdraw assets from `Vault2` and transfers them to `WithdrawalVault`.
3. The manager then calls `completeWithdrawal` to swap all tokens to a destination token using a `Swapper`. However, one of the tokens involved is a custom contract that allows the manager to perform a reentrancy attack on `PoolManagerLogic` and remove the destination token as a supported asset from the Vault.
4. After the swap, all tokens are converted to the destination token and sent back to the Vault.
5. Since the destination token is no longer a supported asset, it is not included in the total asset calculation for the Vault, causing a sudden drop in the Vault's share price.
6. The manager can then exploit this situation by using a 1Inch swap with 100% slippage, performing a self-sandwich attack to extract all the funds from the Vault.

In summary, the lack of validation to ensure that the swapped tokens are supported in the Vault allows a manager to use a custom token, reenter the Vault, remove the destination token as a supported asset, and steal funds from users. The reentrancy can be triggered through calls made by `WithdrawalVault` to input tokens, such as `balanceOf` or `safeIncreaseAllowance`:

```solidity
  function swapToSingleAsset(
    MultiInSingleOutData calldata _swapData,
    uint256 _expectedDestTokenAmount
  ) external override onlyCreator {
    ISwapper swapper = IEasySwapperV2(creator).swapper();

    for (uint256 i; i < _swapData.srcData.length; ++i) {
      // Can only swap all tracked assets at one go
      require(
>>      _swapData.srcData[i].token.balanceOf(address(this)) == _swapData.srcData[i].amount,
        "src amount mismatch"
      );
>>    _swapData.srcData[i].token.safeIncreaseAllowance(address(swapper), _swapData.srcData[i].amount);
    }

    // ...
  }
```

**Recommendation:**<br>

To mitigate this issue, add a validation step to ensure that all input tokens being swapped are supported by the Vault. Additionally, include a check at the end of `swapToSingleAsset` to verify that the destination token remains a supported asset in the Vault.

**Mitigation Review:**<br>

The issue has been fixed at https://github.com/dhedge/dhedge-v2/commit/1d8a126b31348a185cee59f54ffabaac5f352e56.


## Medium Risk Findings

<a name="M-01"></a>
### [M-01] Lack of Support for Asset Type 30 Prevents Single-Asset Withdrawals When the Vault Holds Shares of Another Vault

**Description:**<br>

To facilitate buying and selling shares in other Vaults, a Vault manager follows these steps:
1. Add the `EasySwapperV2` contract as a supported asset within the Vault.
2. Call `initWithdrawal` to initiate the withdrawal of assets from another Vault.
3. Use `completeWithdrawal` to transfer the unrolled assets from the `WithdrawalVault` back to the Vault.

The problem arises because, although the `EasySwapperV2` contract is added as a supported asset in the Vault, the Single-Asset Withdrawal feature cannot be used due to limitations in the `unrollAssets` function. This function retrieves all assets in a Vault, determines their type, and processes each asset type accordingly. However, certain asset types are not supported by `unrollAssets`, including type 30, which corresponds to `EasySwapperV2`

When `unrollAssets` encounters an asset type that is not supported, the entire transaction fails. Here’s a relevant code snippet from the function:.

```solidity
      if (assetType == 0) {
        // ...
      }
      else if (assetType == 1 || assetType == 4 || assetType == 14 || assetType == 22 || assetType == 200) {
        // ...
      }
      else if (assetType == 2) {
        // ...
      }
      // ...
      else if (assetType == 102) {
        // ...
      } else {
>>      revert("assetType not handled");
      }
```

Since asset type 30 is not supported, as long as a Vault includes `EasySwapperV2` as a supported asset, users will be unable to perform Single-Asset Withdrawals to redeem their shares. This limitation also affects managers of other Vaults that hold shares in the affected Vault.

**Recommendation:**<br>

To resolve this issue, support for asset type 30 should be added in `WithdrawalVault::unrollAssets`. No special handling is needed, as the assets in the Vault's `WithdrawalVault` are transferred directly to the user's `WithdrawalVault` during the withdrawal process, eliminating the need for additional unrolling steps.

**Mitigation Review:**<br>

The issue has been fixed at https://github.com/dhedge/dhedge-v2/commit/161ba3114e89e449d988549188157a9cc42d1bc2.

<a name="M-02"></a>
### [M-02] Vault Owner Can Prevent Another Vault from Selling Shares by Adding an Unsupported Asset Type

**Description:**<br>

A manager of a Vault (`Vault1`) can buy and sell shares in other Vaults (e.g., `Vault2`) using `EasySwapperV2`. When a depositor in `Vault1` attempts a Single-Asset Withdrawal (SAW) to sell their shares, the function `unrollAssets` calls `_detectDhedgeVault`, which tries to unroll the assets held in the nested `Vault2`.

However, `unrollAssets` does not support all asset types, and if a Vault contains an asset with an unsupported type, the entire transaction will fail, thereby blocking the withdrawal.

Consider this scenario:
1. The manager of `Vault1` acquires shares in `Vault2`.
2. Subsequently, the manager of `Vault2` adds a supported asset with a type not handled by `unrollAssets`.
3. As a result, users in `Vault1`, including its manager, are unable to execute a Single-Asset Withdrawal to sell shares from `Vault2` because of the unsupported asset.

By exploiting this issue, the manager of `Vault2` can effectively prevent the manager and users of `Vault1` from selling their shares using Single-Asset Withdrawals.

**Recommendation:**<br>

To mitigate this issue, an additional parameter should be introduced in both `unrollAssets` and `_detectDhedgeVault`, allowing users to specify whether to unroll shares from the nested Vault. This approach ensures that even if the manager of `Vault2` adds an unsupported asset type in the `WithdrawalVault`, users can still perform Single-Asset Withdrawals while opting to skip the unrolling of shares from `Vault2`.

An example of this fix could look like the following snippet:
```diff
  function _detectDhedgeVault(
    address _asset,
    address _poolFactory,
    uint256 _slippageTolerance,
+   bool _unrollNestedVault    
  ) internal returns (address[] memory unrolledAssets) {
    if (IPoolFactory(_poolFactory).isPool(_asset)) {
      uint256 balance = IPoolLogic(_asset).balanceOf(address(this));
-     if (balance > 0) {
+     if (balance > 0 && _unrollNestedVault) 
        IPoolLogic(_asset).withdrawSafe(balance, _slippageTolerance);
        _unrollAssets(_asset, _slippageTolerance);
      }
+     else unrolledAssets = _arraify(_asset);
    } else {
      unrolledAssets = _arraify(_asset);
    }
  }
```

**Mitigation Review:**<br>

The issue has been acknowledged, the protocol team commented:

> Makes sense, but I’d prefer not to fix this right now. Other vaults available for managers to buy are always TRUSTED (controlled by [Toros](https://toros.finance/)), i.e. right now all of such vaults are Toros products, either leveraged or yield tokens.

<a name="M-03"></a>
### [M-03] Missing Slippage Checks Allow Manager Exploitation When Using `EasySwapperV2` to Sell Shares of a Nested Vault

**Description:**<br>

A Vault manager can use `EasySwapperV2` to buy and sell shares in other Vaults. When selling shares from a nested Vault, the manager first calls `initWithdrawal` to transfer assets to the `WithdrawalVault`, then uses `completeWithdrawal` to swap all assets into a single token and return them to the Vault.

However, there is a vulnerability in this process: the manager can set an excessively high slippage value for the swap and perform a self-sandwich attack to extract most of the value from the swap. This manipulation enables the manager to effectively steal the majority of the Vault's value, retaining the profit.

This concern was already noted in `easySwapperv2.md`:

> "Also, seems like some bad slippage protection from trades if withdrawing to single token should be added."

The audited version of the contracts lacks slippage protection, resulting in this vulnerability.

**Recommendation:**<br>

To address this issue, it is recommended to implement a slippage check in `EasySwapperV2ContractGuard` to ensure that the slippage value set by the Vault manager is within a reasonable range, preventing the potential extraction of value through the swap.

**Mitigation Review:**<br>

The issue has been acknowledged, the protocol team commented:

> Makes sense, but I will add the fix some time later, thus it won't be included in fix review. Currently Slippage Accumulator flow is being overhauled, there is no point in adding existing slippage checks. Once we [finalize and merge it](https://github.com/dhedge/dhedge-v2/pull/969), new version will be applied to EasySwapperV2ContractGuard.

## Low Risk Findings

<a name="L-01"></a>
### [L-01] Function `_recoverAllAssets` Does Not Clean Up `srcAssets` When Balance is Zero

The function `_recoverAllAssets` in the `WithdrawalVault` is responsible for transferring the entire balance of all tracked assets to a specified address. When assets are sent, the token addresses are removed from the `srcAssets` _AddressSet_. However, assets with a zero balance are not removed from the _AddressSet_, resulting in unused entries remaining in the set.

```solidity
function _recoverAllAssets(address _to) internal {
    uint256 setLength = srcAssets.length();
    address srcAsset;
    uint256 balance;
    for (uint256 i; i < setLength; ) {
      srcAsset = srcAssets.at(i);
      balance = IERC20(srcAsset).balanceOf(address(this));
      if (balance > 0) {
>>      srcAssets.remove(srcAsset);
        setLength--;
        IERC20(srcAsset).safeTransfer(_to, balance);
      } else {
>>      i++;
      }
    }
  }
```

Retaining assets with zero balance in _AddressSet_ serves no purpose and can be detrimental, especially if the same `WithdrawalVault` is reused in the future. A list cluttered with zero-balance assets can increase gas costs and add overhead, leading to a less efficient user experience.

As a recommendation, modify `_recoverAllAssets` to remove all assets from `srcAssets`, regardless of their balance. This change will help optimize gas usage and improve performance when the `WithdrawalVault` is used again.

**Mitigation Review:**<br>

The issue has been fixed at https://github.com/dhedge/dhedge-v2/commit/3e675a21a53aa1fdf658fa1a3c3fe0cf04f8ab77.

<a name="L-02"></a>
### [L-02] Missing Slippage Check on `initWithdrawal` Risks Value Loss When Withdrawing from Nested Vaults

During asset withdrawals from a Vault, a slippage parameter is used to ensure the user receives the expected amount of assets, protecting against value loss due to market fluctuations. This is particularly important for assets deposited in Aave, where market volatility could affect the withdrawal amount.

A Vault manager can perform deposits and withdrawals from other Vaults. However, when initiating a withdrawal, the manager can set an excessively high slippage tolerance, potentially causing Vault users to lose value.

```solidity
  function initWithdrawal(address _dHedgeVault, uint256 _amountIn, uint256 _slippageTolerance) external {
    IERC20(_dHedgeVault).safeTransferFrom(msg.sender, address(this), _amountIn);

    address vault = withdrawalContracts[msg.sender];
    if (vault == address(0)) {
      vault = _createWithdrawalVault(msg.sender);
    }

>>  IPoolLogic(_dHedgeVault).withdrawToSafe(vault, _amountIn, _slippageTolerance);

    // ...
  }
```

Without proper checks, the slippage argument can be set too high during `EasySwapperV2::initWithdrawal`, allowing for potential exploitation and value extraction from the Vault.

As a recommendation, introduce a slippage check in `EasySwapperV2ContractGuard` to ensure that the slippage parameter is set within a reasonable range when managers call `EasySwapperV2::initWithdrawal`. This will help protect users from unnecessary value loss during withdrawals.

**Mitigation Review:**<br>

The issue has been fixed at https://github.com/dhedge/dhedge-v2/commit/6d0275b8127e406073ef479203dbc135faa87255.
