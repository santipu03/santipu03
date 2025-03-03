# Audit Report - Beraborrow Oracle-less Vault

<img src="https://1570492309-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FffzDCMBDa391vIMqruBP%2Fuploads%2F6HytOnGMnHU5RMzqdq3J%2FBeraborrow%20Logo%20with%20text%20(2).png?alt=media&token=3f71d18c-c3f2-4214-bea7-9fb73abc0344" height="160" align="center"/>

<br>
<br>

|                |                                                                           |
| -------------- | ------------------------------------------------------------------------- |
| **Audit Date** | 27/02/2025 - 28/02/2025                                                   |
| **Auditor**    | Santipu ([@santipu_](https://x.com/santipu_)) |
| **Version 1**  | 01/03/2025 Initial Report                                                 |
| **Version 2**  | 03/03/2025 Mitigation Review                                              |

<br>

# Disclaimer

_This audit report is based on a time-limited review of the provided code and information. While efforts were made to identify potential issues, it does not guarantee the discovery of all vulnerabilities or the security of the smart contract. The findings are for informational purposes only and should not be considered as investment, legal, or financial advice._

_The audit does not constitute an endorsement or certification of the project. I disclaim any liability for losses or damages arising from the use or reliance on this report, including issues related to smart contract exploitation or changes made after the audit._

_The project team is responsible for any actions taken based on this report and should seek additional professional advice if needed. By using this report, the team acknowledges and agrees to these terms._

# About Santipu

Santipu is a highly ranked Smart Contract Auditor, primarily conducting audits through [Sherlock](https://www.sherlock.xyz/), while also contributing to platforms like [Code4rena](https://code4rena.com/) and [Cantina](https://cantina.xyz/). With multiple podium finishes in audit contests, including some top-1 placements, Santipu has established a reputation for excellence in the field. Beyond competitions, Santipu has collaborated with leading audit firms, such as [Pashov Audit Group](https://www.pashov.net/) and [Bailsec](https://bailsec.io/).

For inquiries, reach out via [@santipu_](https://x.com/santipu_) on X (formerly Twitter).

# Scope

The audit scope focuses on the new oracle-less vault on Beraborrow. This vault is a variation of the classic Collateral Vault, which stakes its primary asset into Infrared to earn rewards. However, unlike traditional vaults, it does not use an oracle to price its assets. Instead, all rewards from Infrared are treated as donations. A keeper regularly swaps these rewards back into the main asset and vests the swapped amount to ensure fair distribution to vault users over time.

The decision to create an oracle-less vault was driven by the absence of an appropriate on-chain oracle feed for pricing Balancer Pool Tokens (BPT), the underlying protocol that BEX has forked from. While this vault cannot currently be used as collateral within Beraborrow due to the lack of an oracle, it still functions effectively as a yield product. If a suitable price feed becomes available, the vault can be upgraded to enable its use as collateral within Beraborrow.

This scope includes the following files:

- `src/core/vaults/MainAssetCompoundingInfraredCollVault.sol`
- `src/core/vaults/InfraredCollateralVault.sol` (minor changes)
- `src/core/vaults/BaseCollateralVault.sol` (minor changes)

The PR for the initial audit is [207](https://github.com/Beraborrowofficial/blockend/pull/207).

The mitigation review has been conducted in the following commit: [90a7561b7d3619c8496097653d033ef169fc4a31](https://github.com/Beraborrowofficial/blockend/pull/207/commits/90a7561b7d3619c8496097653d033ef169fc4a31)

# Severity Classification
| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | High      | High       | Medium   |
| **Likelihood: Medium** | High      | Medium      | Low      |
| **Likelihood: Low**    | Medium    | Low         | Low      |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability is discovered and exploited

# Summary

| Severity       | Total | Fixed | Acknowledged | Disputed | 
| -------------- | ----- | ----- | ------------ | -------- |
| High           | 0     | 0     | 0            | 0        |
| Medium         | 0     | 0     | 0            | 0        |
| Low            | 3     | 1     | 2            | 0        | 


<br>

| # | Title | Severity | Status |
| - | ----- | -------- | ------ |
| [L-01](#l-01)   | Missing Price Feed Check During Collateral Vault Deployment                           | Low | Fixed |
| [L-02](#l-02)   | Unaccounted Infrared Rewards When Matching the Main Asset                           | Low | Acknowledged |
| [L-03](#l-03)   | Griefed Rewards Should Not Be Vested When Internalized                         | Low | Acknowledged |


# Findings

## Low Risk Findings

<a name="l-01"></a>
### [L-01] Missing Price Feed Check During Collateral Vault Deployment

**Description:**<br>

The new oracle-less vault inherits from the classic Collateral Vaults, but the price feed check typically performed during vault deployment has been commented out to facilitate the creation of vaults without oracles.

```solidity
function __BaseCollateralVault_init_unchained(BaseInitParams calldata params) internal onlyInitializing {
    // ...

    IPriceFeed priceFeed = IPriceFeed(params._metaBeraborrowCore.priceFeed());
>>  // require(priceFeed.fetchPrice(address(params._asset)) != 0, "CollVault: asset price feed not set up");
    
    // ...
}
```

The absence of this check introduces a risk: a Collateral Vault (with an oracle) could be deployed without a properly configured price feed, leading to a Denial of Service (DoS) on the vault. While this scenario would only occur if Beraborrow developers mistakenly failed to set up the price feed before deployment, it is crucial to enforce safeguards that prevent such errors.

**Recommended Mitigation:**<br>

To address this issue, it is recommended to encapsulate the price feed check within a virtual function and override this function in the oracle-less vault implementation. This approach maintains safety while preserving the flexibility needed for oracle-less deployments.

**Mitigation Review:**<br>

The issue has been fixed by following the recommended mitigation. 

<a name="l-02"></a>
### [L-02] Unaccounted Infrared Rewards When Matching the Main Asset

**Description:**<br>

When a Collateral Vault is interacted with, the rewards accrued in the underlying Infrared Vault are harvested and added to the virtual balance. However, rewards are not accounted for when the reward token lacks a configured price feed:

```solidity
function _harvestRewards() internal override {
    // ...
    
    for (uint i; i < tokens.length; i++) {
        // ...
        
>>      // Meanwhile the token doesn't has an oracle mapped, it will be processed as a donation
>>      // This will avoid returns meanwhile a newly Infrared pushed reward token is not mapped
>>      if (_hasPriceFeed(_token) && _token != _iRedToken && !_isCollVault(_token)) {
            uint fee = rewards * _performanceFee / BP;
            uint netRewards = rewards - fee;

            if (_token == asset()) {
                _stake(netRewards);
            }

            _increaseBalance(_token, netRewards);

            // ...
        }
    }
    // ...
}
```

This design choice ensures that the absence of a price feed does not block future vault operations. However, in the context of oracle-less vaults, this approach introduces a flaw: rewards are not accounted for when the reward token is the same as the main asset of the vault, which inherently lacks a price feed.

For example, in an oracle-less vault where a BPT token is the primary asset, any rewards received in the same BPT token would not be distributed to users in the vault, resulting in lost yield.

**Recommended Mitigation:**<br>

To address this issue, it is recommended to vest the Infrared rewards directly when they match the main asset of the vault (in the context of oracle-less vaults). This approach ensures that all rewards contribute to the user's yield, even in the absence of a price feed.

**Mitigation Review:**<br>

The issue has been acknowledged, the developers commented:

> Acknowledged, very unlikely any rewarded token is the asset itself, and if it were, we can set the priceFeed.

<a name="l-03"></a>
### [L-03] Griefed Rewards Should Not Be Vested When Internalized

**Description:**<br>

Collateral Vaults on Beraborrow have a known issue that allows a griefer to directly claim rewards on Infrared on behalf of the Collateral Vault. Since the vault tracks balances virtually, these griefed rewards are not properly accounted for. To address this, the `internalizeDonations` function currently handles griefed rewards by vesting them, which distributes them gradually over time.

```solidity
function internalizeDonations(address[] memory tokens, uint128[] memory amounts) external virtual onlyOwner {
    // ...

    for (uint i; i < tokensLength; i++) {
        // ...

>>      _getBalanceData().addEmissions(token, uint128(netAmounts));
    }
}
```

However, this approach is not suitable for oracle-less vaults. In these vaults, Infrared rewards are only accounted for once they are swapped to the main asset. The current design would result in griefed rewards being vested first through `internalizeDonations`. The keeper would then need to wait for the vesting period to complete before swapping them to the main asset and vesting them again to distribute to users.

While this is a minor issue given the default vesting period is only 10 seconds, it still introduces unnecessary friction for keepers.

**Recommended Mitigation:**<br>

To resolve this issue, it is recommended to override the `internalizeDonations` function on `MainAssetCompoundingInfraredCollVault`. Instead of vesting the internalized donations, they should be directly added to the virtual balance. This change would enable the keeper to swap and distribute rewards to users as quickly as possible, improving efficiency and a fair distribution.

**Mitigation Review:**<br>

The issue has been acknowledged, the developers commented:

> Acknowledged, the default unlock rate is good enough for the vesting to be meaningless
