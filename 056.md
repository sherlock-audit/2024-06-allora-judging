Dry Sepia Dinosaur

High

# Attacker can slow down / halt the chain by queuing multiple stake removals or delegate stake removals

## Summary
Attacker can slow down / halt the chain by queuing multiple stake removals or delegate stake removals.

## Vulnerability Detail
All stake removals and delegate stake removals for a given block are processed in the EndBlocker in a loop.

Since there is no minimum restriction on the stake amount, an attacker can either :
* Depending on the registration fee, register multiple reputers, add a `1 uallo` stake to each one of them and then cancel his stakes for each one of them.
* Delegate stake `1 uallo` from multiple addresses to each registered reputer then cancel all of them.

## Impact
Slow down / halt the chain.

## Code Snippet
[EndBlocker](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/abci.go#L14-L120)
```golang
func EndBlocker(ctx context.Context, am AppModule) error {
	sdkCtx := sdk.UnwrapSDKContext(ctx)
	blockHeight := sdkCtx.BlockHeight()
	sdkCtx.Logger().Debug(
		fmt.Sprintf("\n ---------------- Emissions EndBlock %d ------------------- \n",
			blockHeight))

	// Remove Stakers that have been wanting to unstake this block. They no longer get paid rewards
	RemoveStakes(sdkCtx, blockHeight, am.keeper) // <===== Audit
	RemoveDelegateStakes(sdkCtx, blockHeight, am.keeper) // <===== Audit

	// ...
}
```

[RemoveStakes](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/stake_removals.go#L13-L63)
```golang
func RemoveStakes(
	sdkCtx sdk.Context,
	currentBlock int64,
	k emissionskeeper.Keeper,
) {
	removals, err := k.GetStakeRemovalsForBlock(sdkCtx, currentBlock)  // <===== Audit
	if err != nil {
		sdkCtx.Logger().Error(fmt.Sprintf(
			"Unable to get stake removals for block %d, skipping removing stakes: %v",
			currentBlock,
			err,
		))
		return
	}
	for _, stakeRemoval := range removals {  // <===== Audit
		// ...
	}
}
```

[RemoveDelegateStakes](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/stake_removals.go#L66-L118)
```golang
func RemoveDelegateStakes(
	sdkCtx sdk.Context,
	currentBlock int64,
	k emissionskeeper.Keeper,
) {
	removals, err := k.GetDelegateStakeRemovalsForBlock(sdkCtx, currentBlock)  // <===== Audit
	if err != nil {
		sdkCtx.Logger().Error(
			fmt.Sprintf(
				"Unable to get stake removals for block %d, skipping removing stakes: %v",
				currentBlock,
				err,
			))
		return
	}
	for _, stakeRemoval := range removals {  // <===== Audit
		// ...
	}
}
```
## Tool used
Manual Review

## Recommendation
Process stake removals and delegate stake removals that reached maturity in batches with a predefined size over multiple blocks.
