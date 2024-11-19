# Issue H-1: Withdrawal fee for UsualX vault will be mis-calculated. 

Source: https://github.com/sherlock-audit/2024-10-usual-labs-v1-judging/issues/34 

## Found by 
0x37, 0xNirix, 0xc0ffEE, 0xmystery, Bigsam, Boy2000, KupiaSec, LonWof-Demon, Ragnarok, almurhasan, blutorque, dany.armstrong90, dhank, lanrebayode77, shaflow01, silver\_eth, super\_jack, uuzall, xiaoming90, zarkk01
### Summary

The `UsualX.withdraw()` function has logic error in calculating the withdrawal fee.

### Root Cause

The [UsualX.withdraw()](https://github.com/sherlock-audit/2024-10-usual-labs-v1/blob/main/pegasus/packages/solidity/src/vaults/UsualX.sol#L319-L343) function is following.
```solidity
    function withdraw(uint256 assets, address receiver, address owner)
        public
        override
        whenNotPaused
        nonReentrant
        returns (uint256 shares)
    {
        UsualXStorageV0 storage $ = _usualXStorageV0();
        YieldDataStorage storage yieldStorage = _getYieldDataStorage();

        // Check withdrawal limit
        uint256 maxAssets = maxWithdraw(owner);
        if (assets > maxAssets) {
            revert ERC4626ExceededMaxWithdraw(owner, assets, maxAssets);
        }
        // Calculate shares needed
335:    shares = previewWithdraw(assets);
336:    uint256 fee = Math.mulDiv(assets, $.withdrawFeeBps, BASIS_POINT_BASE, Math.Rounding.Ceil);

        // Perform withdrawal (exact assets to receiver)
        super._withdraw(_msgSender(), receiver, owner, assets, shares);

        // take the fee
342:    yieldStorage.totalDeposits -= fee;
    }
```
The [UsualX.previewWithdraw()](https://github.com/sherlock-audit/2024-10-usual-labs-v1/blob/main/pegasus/packages/solidity/src/vaults/UsualX.sol#L393-L404) function on `L335` is following.
```solidity
    function previewWithdraw(uint256 assets) public view override returns (uint256 shares) {
        UsualXStorageV0 storage $ = _usualXStorageV0();
        // Calculate the fee based on the equivalent assets of these shares
396:    uint256 fee = Math.mulDiv(
            assets, $.withdrawFeeBps, BASIS_POINT_BASE - $.withdrawFeeBps, Math.Rounding.Ceil
        );
        // Calculate total assets needed, including fee
        uint256 assetsWithFee = assets + fee;

        // Convert the total assets (including fee) to shares
        shares = _convertToShares(assetsWithFee, Math.Rounding.Ceil);
    }
```
As can be seen, The calculations of withdrawal fee is different in `L336` and `L396`.
Since `assets` refers to the asset amount without fee in the `withdraw()` and `previewWithdraw()` functions, the formula of `L336` is wrong and the `fee` of `L336` will be smaller than the `fee` of `L396` when `withdrawFeeBps > 0`. As a result, the `totalDeposits` in `L342` will becomes larger than it should be.



### Internal pre-conditions

`withdrawFeeBps` is larger than zero.

### External pre-conditions

_No response_

### Attack Path

1. Assume that there are multiple users in the `UsualX` vault.
2. A user withdraws some assets from the vault.
3. `totalDeposits` becomes larger than it should be. Therefore, the assets of other users will be inflated.


### Impact

Loss of protocol's fee.
Broken core functionality because later withdrawers can make a profit.

### PoC

Add the following code to `UsualXUnit.t.sol`.
```solidity
    function test_withdrawFee() public {
        // update withdrawFee as 5%
        uint256 fee = 500;
        vm.prank(admin);
        registryAccess.grantRole(WITHDRAW_FEE_UPDATER_ROLE, address(this));
        usualX.updateWithdrawFee(fee);

        // initialize the vault with user1 and user2
        Init memory init;
        init.share[1] = 1e18; init.share[2] = 1e18;
        init.asset[1] = 1e18; init.asset[2] = 1e18;
        init = clamp(init);
        setUpVault(init);
        address user1 = init.user[1];
        address user2 = init.user[2];

        uint256 assetsOfUser2Before = _max_withdraw(user2);

        // user1 withdraw 0.5 ether from the vault
        vm.prank(user1);
        vault_withdraw(0.5e18, user1, user1);

        uint256 assetsOfUser2After = _max_withdraw(user2);

        // compare assets of user2 before withdrawal
        emit log_named_uint("before", assetsOfUser2Before);
        emit log_named_uint("after ", assetsOfUser2After);
    }
```
The output log of the above code is the following.
```bash
Ran 1 test for test/vaults/UsualXUnit.t.sol:UsualXUnitTest
[PASS] test_withdrawFee() (gas: 456410)
Logs:
  before: 1000000000000000100
  after : 1000892857142857243

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.10ms (3.33ms CPU time)

Ran 1 test suite in 20.07ms (13.10ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
As can be seen, after user1 withdraws, the assets of user2 becomes inflated.

### Mitigation

Modify `UsualX.withdraw()` function similar to the `UsualX.previewWithdraw()` function as follows.
```solidity
    function withdraw(uint256 assets, address receiver, address owner)
        public
        override
        whenNotPaused
        nonReentrant
        returns (uint256 shares)
    {
        UsualXStorageV0 storage $ = _usualXStorageV0();
        YieldDataStorage storage yieldStorage = _getYieldDataStorage();

        // Check withdrawal limit
        uint256 maxAssets = maxWithdraw(owner);
        if (assets > maxAssets) {
            revert ERC4626ExceededMaxWithdraw(owner, assets, maxAssets);
        }
        // Calculate shares needed
        shares = previewWithdraw(assets);
--      uint256 fee = Math.mulDiv(assets, $.withdrawFeeBps, BASIS_POINT_BASE, Math.Rounding.Ceil);
++      uint256 fee = Math.mulDiv(assets, $.withdrawFeeBps, BASIS_POINT_BASE - $.withdrawFeeBps, Math.Rounding.Ceil);

        // Perform withdrawal (exact assets to receiver)
        super._withdraw(_msgSender(), receiver, owner, assets, shares);

        // take the fee
        yieldStorage.totalDeposits -= fee;
    }
```

# Issue M-1: Users can claim the whole airdrop amount ahead of schedule without paying any tax amount 

Source: https://github.com/sherlock-audit/2024-10-usual-labs-v1-judging/issues/122 

## Found by 
0xAadi, 0xNirix, 0xc0ffEE, DenTonylifer, Falconhoof, Matin, bozhikov\_, dany.armstrong90, iamandreiski, xiaoming90
### Summary

Due to precision loss, people will be able to claim their fully vested airdrop, from 4.5 hours up to 2 days + before it actually gets vested due to precision error.

To be more exact, the airdrop will be fully claimable 15,724 seconds before it actually is or up to 157,240 seconds, depending on what the tax % is 

Due to how the tax is calculated, this will result in in a tax of 0, which would allow users to “pay” 0 tax, and claim the airdrop prematurely without any value loss.

### Root Cause

https://github.com/sherlock-audit/2024-10-usual-labs-v1/blob/4fb4a64a479e0b9b8f93934220e891c29d54df33/pegasus/packages/solidity/src/airdrop/AirdropTaxCollector.sol#L232-L233

There are multiple options in to how the airdrop can actually be claimed, some of the options include: 

- The whole airdrop is claimable immediately for the bottom 20%;
- For the top 80% - paying a tax which would immediately unlock the full amount (in exchange for the fee/tax);
- Waiting for the vesting period to pass, and having a portion of the airdrop unlocked with each passing month.

Due to how the tax is calculated: 

```solidity
    function _calculateClaimTaxAmount(AirdropTaxCollectorStorage storage $, address account)
        internal
        view
        returns (uint256 claimTaxAmount) // 
    {
        uint256 claimerUsd0PPBalance = $.prelaunchUsd0ppBalance[account];

        uint256 claimingTimeLeft;
        if (block.timestamp > AIRDROP_INITIAL_START_TIME + AIRDROP_CLAIMING_PERIOD_LENGTH) { 
            claimingTimeLeft = 0;
        } else {
            claimingTimeLeft =
                AIRDROP_INITIAL_START_TIME + AIRDROP_CLAIMING_PERIOD_LENGTH - block.timestamp; 
        }
                            
        uint256 taxFee = $.maxChargeableTax.mulDiv(claimingTimeLeft, AIRDROP_CLAIMING_PERIOD_LENGTH); 
        claimTaxAmount = claimerUsd0PPBalance.mulDiv(taxFee, BASIS_POINT_BASE);
    }
```

The above formula is manipulable 4+ hours before the whole period unlocks due to precision loss, which would allow people to be counted as if they’ve paid the tax, while actually paying 0 tax. 

- `uint256 constant AIRDROP_INITIAL_START_TIME = 1_734_004_800;`
- `uint256 constant AIRDROP_CLAIMING_PERIOD_LENGTH = 182 days;`
- 182 days = 15724800;

Whenever the `claimingTimeLeft` would be 15_724 or lower, the tax amount would result as 0 due to precision loss. 

For example: 

- `claimingTimeLeft = AIRDROP_INITIAL_START_TIME + AIRDROP_CLAIMING_PERIOD_LENGTH - block.timestamp;`
- `claimingTimeLeft`  = `1_734_004_800` + `15_724_800` - **`1749713876` = `15_724`**
- Above we assume that the current block.timestamp is `1749713876` or just above 4 hours before the claiming time left passes.

This would result in: 

`uint256 taxFee = $.maxChargeableTax.mulDiv(claimingTimeLeft, AIRDROP_CLAIMING_PERIOD_LENGTH);` 

Or

`uint256 taxFee` = 1_000 * 15_724 / 15_724_800 = 0;`

This is assuming that the `maxChargeableTax` is 1_000 (or around 10%), if the tax is for example 5% or 500, this would increase the time that tax can be manipulated to double, or 8+ hours, and so on. 

And if we have a number as low as 1% for example, this would cause the airdrop to be “claimable”, i.e. users to pay 0 tax, almost 2 days prior to the deadline expiration.

Since `taxFee` is 0, the `claimTaxAmount` would also be 0 due to: 

`claimTaxAmount = claimerUsd0PPBalance.mulDiv(taxFee, BASIS_POINT_BASE);` 

Or

`claimTaxAmount`  = 100_000e18 (example balance) * 0 / 10_000 = 0;

### Internal pre-conditions

Depending on the maxChargeableTax set, due to precision loss, a user’s airdrop will be prematurely available up to 2 days before the expiration date, without paying any tax due to precision loss.

### External pre-conditions

User hasn’t “rage-quit” the airdrop by early unlocking their bonds;

### Attack Path

- Malicious user calls the `payTaxAmount` function 2 days prior to the expiration of the airdrop vesting period (this is assuming that the tax is set at 1%);
- Due to precision loss, the user’s tax is calculated at 0;
- They don’t transfer any Usd0++ while at the same time their `taxedClaimers` status is set to true;
- They claim all of the funds without paying any tax, 2 days prior to the vesting period is finished;

### Impact

Depending on the tax % set up for the airdrop tax, users can pay 0 tax up to 2 days before the deadline expiration and claim the fully vested amount of the airdrop.

### PoC

The `mulDiv` function in the Math.sol lib can be utilized to test the above mentioned scenarios and observe the precision loss:

```solidity
    function test1(uint256 taxPercentage, uint256 timeLeft)public pure returns (uint256){
        return mulDiv(taxPercentage, timeLeft, 15_724_800);
    }
```

### Mitigation

Make sure that this scenario is accounted for when it comes to precision loss by adding 1 wei if the result is 0 but vesting hasn't fully finished yet.

# Issue M-2: USUAL tokens stuck in the `AirdropDistribution` contract 

Source: https://github.com/sherlock-audit/2024-10-usual-labs-v1-judging/issues/140 

## Found by 
xiaoming90
### Summary

_No response_

### Root Cause

_No response_

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

When users claim their vested airdrop tokens via the `AirdropDistribution.claim` function, the `AirdropDistribution.penalty()` function will be executed to compute the penalty amount for a given amount.

https://github.com/sherlock-audit/2024-10-usual-labs-v1/blob/main/pegasus/packages/solidity/src/airdrop/AirdropDistribution.sol#L190

```solidity
File: AirdropDistribution.sol
184:     /// @notice Computes the penalty amount for the given account.
185:     /// @param $ The storage struct of the contract.
186:     /// @param totalAmount Total amount claimable by the user.
187:     /// @param account Address of the account.
188:     /// @param monthsPassed Number of months passed since the start of the vesting period.
189:     /// @return The penalty amount.
190:     function _computePenalty(
191:         AirdropDistributionStorageV0 storage $,
192:         uint256 totalAmount,
193:         address account,
194:         uint256 monthsPassed
195:     ) internal returns (uint256) {
```

The penalty will be deducted from the user's claimable amount. Thus, if the user's total vested USUAL tokens is 10000, but the penalty is 3000, users will only receive 7000 USUAL. The remaining 3000 USUAL will reside within the `AirdropDistribution` contract.

However, the issue is that there is no way for the protocol or the admin to retrieve the 3000 USUAL collected as penalty. Thus, they are stuck in the contract.

> [!IMPORTANT]
>
> Per the Contest's README, it was a known issue that withdrawal fees would be stuck within the `UsualX` contract. However, this issue is related to the penalty being stuck in `AirdropDistribution` contract, not the `UsualX` contract. Thus, it is a different contract, and this issue is technically not a known issue in the context of this audit contest if the rules are followed strictly during the judging.
>
> > Q: Please discuss any design choices you made.
> > We are choosing to not be able to withdraw fees on UsualX for the time being. At the moment, the fees wouldn't be withdrawable. This is a deliberate choice to be implemented in a later contract upgrade, when the protocol governance decides on how the unstaking fees should be dealt with (i.e. burned, redistributed).

### Impact

USUAL tokens stuck in the `AirdropDistribution` contract. No one can collect the USUAL tokens collected from the users as a penalty.

Severity: High. Assets stuck in contract.

### PoC

_No response_

### Mitigation

Consider implementing a function to retrieve the penalty fee OR document this issue as one of the known issues.

