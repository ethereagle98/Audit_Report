# Title 1: In `ZivoeITO`, since `claimAirdrop` doesn't use total supply of junior & senior tranch tokens at the time of the token migration, the owners of credit may receive less $ZVE than expected.

## Severity

Medium

## Summary

In `ZivoeITO`, the depositors will receive $ZVE according to the amount of purchased amount of junior & senior tranch tokens. This amount of $ZVE is calculated based on the proportion of owned junior & senior tranch tokens against those total supply at the moment of ITO is ended.
However, since getting total supply of tranch tokens is incorrect in `claimAirdrop` function, participants of ITO may receive less $ZVE than expected.

## Vulnerability Detail

In `claimAirDrop` function of `ZivoeITO`, the amount of $ZVE that a depositor will receive is calculated as follow:

```javascript
    function claimAirdrop(address depositor) external returns (
        uint256 zSTTClaimed, uint256 zJTTClaimed, uint256 ZVEVested
    ) {
        require(end != 0, "ZivoeITO::claimAirdrop() end == 0");
        require(block.timestamp > end || migrated, "ZivoeITO::claimAirdrop() block.timestamp <= end && !migrated");

        // ...

        // Calculate proportion of $ZVE awarded based on $pZVE credits.
@>      uint256 upper = seniorCreditsOwned + juniorCreditsOwned;
@>      uint256 middle = IERC20(IZivoeGlobals_ITO(GBL).ZVE()).totalSupply() / 20;
@>      uint256 lower = IERC20(IZivoeGlobals_ITO(GBL).zSTT()).totalSupply() * 3 + (
            IERC20(IZivoeGlobals_ITO(GBL).zJTT()).totalSupply()
        );

        emit AirdropClaimed(depositor, seniorCreditsOwned / 3, juniorCreditsOwned, upper * middle / lower);

        IERC20(IZivoeGlobals_ITO(GBL).zJTT()).safeTransfer(depositor, juniorCreditsOwned);
        IERC20(IZivoeGlobals_ITO(GBL).zSTT()).safeTransfer(depositor, seniorCreditsOwned / 3);

@>      if (upper * middle / lower > 0) {
@>          ITO_IZivoeRewardsVesting(IZivoeGlobals_ITO(GBL).vestZVE()).createVestingSchedule(
                depositor, 0, 360, upper * middle / lower, false
            );
        }
        
        // ...
    }
```
As you can see in above codes, the large total supply of junior or senior tranch tokens, the less amount of $ZVE will be received.
In addition, to be correct, they should use the total supply at the moment of ITO is ended (token migrated). However, in above codebase, latest total supply of tranch tokens is used.
As a result, following issue is available.

1. ZVL migrate tokens in ITO. It will also make tranches be unlocked.
```javascript
    function migrateDeposits() external {
        require(end != 0, "ZivoeITO::migrateDeposits() end == 0");
        if (_msgSender() != IZivoeGlobals_ITO(GBL).ZVL()) {
            require(block.timestamp > end, "ZivoeITO::migrateDeposits() block.timestamp <= end");
        }
        require(!migrated, "ZivoeITO::migrateDeposits() migrated");
        
@>      migrated = true;

        // ...

        ITO_IZivoeYDL(IZivoeGlobals_ITO(GBL).YDL()).unlock();
@>      ITO_IZivoeTranches(IZivoeGlobals_ITO(GBL).ZVT()).unlock();
    }
```
2. As soon as performed migrations in ITO and tranches are unlocked, the other normal depositors start depositing to junior & senior tranches. As a result, the total supply of tranch tokens will increase.
3. After that, if the participants of ITO try to claim their $ZVE, the amount will becomes less than expected since the calculated `lower` is greater than the time of migrated.

## Impact

The participants of ITO may receive less $ZVE than expected.

## Code Snippet

Described in vulnerability details.

## Tool used

Manual Review

## Recommendation

Consider introducing new state that store total supply of tranch tokens when ITO is migrated. 
