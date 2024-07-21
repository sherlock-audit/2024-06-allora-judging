Dry Sepia Dinosaur

High

# RemoveStakes and RemoveDelegateStakes silently handle errors in EndBlocker

## Summary
RemoveStakes and RemoveDelegateStakes silently handle errors in EndBlocker.

## Vulnerability Detail
When finalizing stake removals in the EndBlocker, for each removal :
1) First, the unstaked amount is sent to the staker.
2) Then the state is updated with `RemoveReputerStake` / `RemoveDelegateStake` to reflect the removal (.i.e : update total stakes, update topic stakes, update reputer stakes, delete the processed removal, ...).

If an error happens at any of these stages, it simply continues to the next removal.

## Impact
An error in the first step doesn't lead to an issue, however, an error in the second step at any stage leads to :
* An inconsistent written state as some changes will be written while the ones that are meant to happen after the error will not be written.
* The staker will receive his stakes and still be able to re queue another removal for the same stakes.

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
	removals, err := k.GetStakeRemovalsForBlock(sdkCtx, currentBlock)
	if err != nil {
		sdkCtx.Logger().Error(fmt.Sprintf(
			"Unable to get stake removals for block %d, skipping removing stakes: %v",
			currentBlock,
			err,
		))
		return
	}
	for _, stakeRemoval := range removals {
		// do no checking that the stake removal struct is valid. In order to have a stake removal
		// it would have had to be created in msgServer.RemoveStake which would have done
		// validation of validity up front before scheduling the delay

		// Check the module has enough funds to send back to the sender
		// Bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
		// Send the funds
		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
		err = k.SendCoinsFromModuleToAccount(sdkCtx, emissionstypes.AlloraStakingAccountName, stakeRemoval.Reputer, coins)
		if err != nil {
			sdkCtx.Logger().Error(fmt.Sprintf(
				"Error removing stake: %v | %v",
				stakeRemoval,
				err,
			))
			continue
		}

		// Update the stake data structures
		err = k.RemoveReputerStake( // <===== Audit
			sdkCtx,
			currentBlock,
			stakeRemoval.TopicId,
			stakeRemoval.Reputer,
			stakeRemoval.Amount,
		)
		if err != nil { // <===== Audit
			sdkCtx.Logger().Error(fmt.Sprintf(
				"Error removing stake: %v | %v",
				stakeRemoval,
				err,
			))
			continue // <===== Audit
		}
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
	removals, err := k.GetDelegateStakeRemovalsForBlock(sdkCtx, currentBlock)
	if err != nil {
		sdkCtx.Logger().Error(
			fmt.Sprintf(
				"Unable to get stake removals for block %d, skipping removing stakes: %v",
				currentBlock,
				err,
			))
		return
	}
	for _, stakeRemoval := range removals {
		// do no checking that the stake removal struct is valid. In order to have a stake removal
		// it would have had to be created in msgServer.RemoveDelegateStake which would have done
		// validation of validity up front before scheduling the delay

		// Check the module has enough funds to send back to the sender
		// Bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
		// Send the funds
		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
		err = k.SendCoinsFromModuleToAccount(sdkCtx, emissionstypes.AlloraStakingAccountName, stakeRemoval.Delegator, coins)
		if err != nil {
			sdkCtx.Logger().Error(fmt.Sprintf(
				"Error removing stake: %v | %v",
				stakeRemoval,
				err,
			))
			continue
		}

		// Update the stake data structures
		err = k.RemoveDelegateStake(  // <===== Audit
			sdkCtx,
			currentBlock,
			stakeRemoval.TopicId,
			stakeRemoval.Delegator,
			stakeRemoval.Reputer,
			stakeRemoval.Amount,
		)
		if err != nil {  // <===== Audit
			sdkCtx.Logger().Error(fmt.Sprintf(
				"Error removing stake: %v | %v",
				stakeRemoval,
				err,
			))
			continue // <===== Audit
		}
	}
}
```

[RemoveReputerStake](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L858-L925)
```golang
func (k *Keeper) RemoveReputerStake(
	ctx context.Context,
	blockHeight BlockHeight,
	topicId TopicId,
	reputer ActorId,
	stakeToRemove cosmosMath.Int) error {
	// CHECKS
	if stakeToRemove.IsZero() {
		return nil
	}
	// Check reputerAuthority >= stake
	reputerAuthority, err := k.GetStakeReputerAuthority(ctx, topicId, reputer)
	if err != nil {
		return err
	}
	delegateStakeUponReputerInTopic, err := k.GetDelegateStakeUponReputer(ctx, topicId, reputer)
	if err != nil {
		return err
	}
	reputerStakeInTopicWithoutDelegateStake := reputerAuthority.Sub(delegateStakeUponReputerInTopic)
	if stakeToRemove.GT(reputerStakeInTopicWithoutDelegateStake) {
		return types.ErrIntegerUnderflowTopicReputerStake
	}
	reputerStakeNew := reputerAuthority.Sub(stakeToRemove)

	// Check topicStake >= stake
	topicStake, err := k.GetTopicStake(ctx, topicId)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(topicStake) {
		return types.ErrIntegerUnderflowTopicStake
	}
	topicStakeNew := topicStake.Sub(stakeToRemove)

	// Check totalStake >= stake
	totalStake, err := k.GetTotalStake(ctx)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(totalStake) {
		return types.ErrIntegerUnderflowTotalStake
	}

	// Set topic-reputer stake
	if err := k.SetStakeReputerAuthority(ctx, topicId, reputer, reputerStakeNew); err != nil {
		return errorsmod.Wrapf(err, "Setting removed reputer stake in topic failed")
	}

	// Set topic stake
	if err := k.SetTopicStake(ctx, topicId, topicStakeNew); err != nil {
		return errorsmod.Wrapf(err, "Setting removed topic stake failed")
	}

	// Set total stake
	err = k.SetTotalStake(ctx, totalStake.Sub(stakeToRemove))
	if err != nil {
		return errorsmod.Wrapf(err, "Setting total stake failed")
	}

	// remove stake withdrawal information
	err = k.DeleteStakeRemoval(ctx, blockHeight, topicId, reputer)
	if err != nil {
		return errorsmod.Wrapf(err, "Deleting stake removal from queue failed")
	}

	return nil
}
```

[RemoveDelegateStake](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L931-L1072)
```golang
func (k *Keeper) RemoveDelegateStake(
	ctx context.Context,
	blockHeight BlockHeight,
	topicId TopicId,
	delegator ActorId,
	reputer ActorId,
	stakeToRemove cosmosMath.Int,
) error {
	// CHECKS
	if stakeToRemove.IsZero() {
		return nil
	}

	// stakeSumFromDelegator >= stake
	stakeSumFromDelegator, err := k.GetStakeFromDelegatorInTopic(ctx, topicId, delegator)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(stakeSumFromDelegator) {
		return types.ErrIntegerUnderflowStakeFromDelegator
	}
	stakeFromDelegatorNew := stakeSumFromDelegator.Sub(stakeToRemove)

	// delegatedStakePlacement >= stake
	delegatedStakePlacement, err := k.GetDelegateStakePlacement(ctx, topicId, delegator, reputer)
	if err != nil {
		return err
	}
	unStakeDec, err := alloraMath.NewDecFromSdkInt(stakeToRemove)
	if err != nil {
		return err
	}
	if delegatedStakePlacement.Amount.Lt(unStakeDec) {
		return types.ErrIntegerUnderflowDelegateStakePlacement
	}

	// Get share for this topicId and reputer
	share, err := k.GetDelegateRewardPerShare(ctx, topicId, reputer)
	if err != nil {
		return err
	}

	// Calculate pending reward and send to delegator
	pendingReward, err := delegatedStakePlacement.Amount.Mul(share)
	if err != nil {
		return err
	}
	pendingReward, err = pendingReward.Sub(delegatedStakePlacement.RewardDebt)
	if err != nil {
		return err
	}
	if pendingReward.Gt(alloraMath.NewDecFromInt64(0)) {
		err = k.SendCoinsFromModuleToAccount(
			ctx,
			types.AlloraPendingRewardForDelegatorAccountName,
			delegator,
			sdk.NewCoins(sdk.NewCoin(params.DefaultBondDenom, pendingReward.SdkIntTrim())),
		)
		if err != nil {
			return errorsmod.Wrapf(err, "Sending pending reward to delegator failed")
		}
	}

	newAmount, err := delegatedStakePlacement.Amount.Sub(unStakeDec)
	if err != nil {
		return err
	}
	newRewardDebt, err := newAmount.Mul(share)
	if err != nil {
		return err
	}
	stakePlacementNew := types.DelegatorInfo{
		Amount:     newAmount,
		RewardDebt: newRewardDebt,
	}

	// stakeUponReputer >= stake
	stakeUponReputer, err := k.GetDelegateStakeUponReputer(ctx, topicId, reputer)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(stakeUponReputer) {
		return types.ErrIntegerUnderflowDelegateStakeUponReputer
	}
	stakeUponReputerNew := stakeUponReputer.Sub(stakeToRemove)

	// stakeReputerAuthority >= stake
	stakeReputerAuthority, err := k.GetStakeReputerAuthority(ctx, topicId, reputer)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(stakeReputerAuthority) {
		return types.ErrIntegerUnderflowReputerStakeAuthority
	}
	stakeReputerAuthorityNew := stakeReputerAuthority.Sub(stakeToRemove)

	// topicStake >= stake
	topicStake, err := k.GetTopicStake(ctx, topicId)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(topicStake) {
		return types.ErrIntegerUnderflowTopicStake
	}
	topicStakeNew := topicStake.Sub(stakeToRemove)

	// totalStake >= stake
	totalStake, err := k.GetTotalStake(ctx)
	if err != nil {
		return err
	}
	if stakeToRemove.GT(totalStake) {
		return types.ErrIntegerUnderflowTotalStake
	}
	totalStakeNew := totalStake.Sub(stakeToRemove)

	// SET NEW VALUES AFTER CHECKS

	if err := k.SetStakeFromDelegator(ctx, topicId, delegator, stakeFromDelegatorNew); err != nil {
		return errorsmod.Wrapf(err, "Setting stake from delegator failed")
	}
	if err := k.SetDelegateStakePlacement(ctx, topicId, delegator, reputer, stakePlacementNew); err != nil {
		return errorsmod.Wrapf(err, "Setting delegate stake placement failed")
	}
	if err := k.SetDelegateStakeUponReputer(ctx, topicId, reputer, stakeUponReputerNew); err != nil {
		return errorsmod.Wrapf(err, "Setting delegate stake upon reputer failed")
	}
	if err := k.SetStakeReputerAuthority(ctx, topicId, reputer, stakeReputerAuthorityNew); err != nil {
		return errorsmod.Wrapf(err, "Setting reputer stake authority failed")
	}
	if err := k.SetTopicStake(ctx, topicId, topicStakeNew); err != nil {
		return errorsmod.Wrapf(err, "Setting topic stake failed")
	}
	if err := k.SetTotalStake(ctx, totalStakeNew); err != nil {
		return errorsmod.Wrapf(err, "Setting total stake failed")
	}
	if err := k.DeleteDelegateStakeRemoval(ctx, blockHeight, topicId, reputer, delegator); err != nil {
		return errorsmod.Wrapf(err, "Deleting delegate stake removal from queue failed")
	}

	return nil
}
```
## Tool used
Manual Review

## Recommendation
Try to update the state first using a cache context and only write the changes if there are no errors.

```diff
diff --git a/allora-chain/x/emissions/module/stake_removals.go b/allora-chain/x/emissions/module/stake_removals.go
index 14d45c6..1f235e9 100644
--- a/allora-chain/x/emissions/module/stake_removals.go
+++ b/allora-chain/x/emissions/module/stake_removals.go
@@ -25,15 +25,16 @@ func RemoveStakes(
 		return
 	}
 	for _, stakeRemoval := range removals {
-		// do no checking that the stake removal struct is valid. In order to have a stake removal
-		// it would have had to be created in msgServer.RemoveStake which would have done
-		// validation of validity up front before scheduling the delay
+		cacheSdkCtx, write := sdkCtx.CacheContext()
 
-		// Check the module has enough funds to send back to the sender
-		// Bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
-		// Send the funds
-		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
-		err = k.SendCoinsFromModuleToAccount(sdkCtx, emissionstypes.AlloraStakingAccountName, stakeRemoval.Reputer, coins)
+		// Update the stake data structures
+		err = k.RemoveReputerStake(
+			cacheSdkCtx,
+			currentBlock,
+			stakeRemoval.TopicId,
+			stakeRemoval.Reputer,
+			stakeRemoval.Amount,
+		)
 		if err != nil {
 			sdkCtx.Logger().Error(fmt.Sprintf(
 				"Error removing stake: %v | %v",
@@ -43,14 +44,15 @@ func RemoveStakes(
 			continue
 		}
 
-		// Update the stake data structures
-		err = k.RemoveReputerStake(
-			sdkCtx,
-			currentBlock,
-			stakeRemoval.TopicId,
-			stakeRemoval.Reputer,
-			stakeRemoval.Amount,
-		)
+		// do no checking that the stake removal struct is valid. In order to have a stake removal
+		// it would have had to be created in msgServer.RemoveStake which would have done
+		// validation of validity up front before scheduling the delay
+
+		// Check the module has enough funds to send back to the sender
+		// Bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
+		// Send the funds
+		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
+		err = k.SendCoinsFromModuleToAccount(sdkCtx, emissionstypes.AlloraStakingAccountName, stakeRemoval.Reputer, coins)
 		if err != nil {
 			sdkCtx.Logger().Error(fmt.Sprintf(
 				"Error removing stake: %v | %v",
@@ -59,6 +61,8 @@ func RemoveStakes(
 			))
 			continue
 		}
+
+		write()
 	}
 }
 
@@ -79,15 +83,18 @@ func RemoveDelegateStakes(
 		return
 	}
 	for _, stakeRemoval := range removals {
-		// do no checking that the stake removal struct is valid. In order to have a stake removal
-		// it would have had to be created in msgServer.RemoveDelegateStake which would have done
-		// validation of validity up front before scheduling the delay
+		cacheSdkCtx, write := sdkCtx.CacheContext()
+
+		// Update the stake data structures
+		err = k.RemoveDelegateStake(
+			cacheSdkCtx,
+			currentBlock,
+			stakeRemoval.TopicId,
+			stakeRemoval.Delegator,
+			stakeRemoval.Reputer,
+			stakeRemoval.Amount,
+		)
 
-		// Check the module has enough funds to send back to the sender
-		// Bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
-		// Send the funds
-		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
-		err = k.SendCoinsFromModuleToAccount(sdkCtx, emissionstypes.AlloraStakingAccountName, stakeRemoval.Delegator, coins)
 		if err != nil {
 			sdkCtx.Logger().Error(fmt.Sprintf(
 				"Error removing stake: %v | %v",
@@ -97,15 +104,15 @@ func RemoveDelegateStakes(
 			continue
 		}
 
-		// Update the stake data structures
-		err = k.RemoveDelegateStake(
-			sdkCtx,
-			currentBlock,
-			stakeRemoval.TopicId,
-			stakeRemoval.Delegator,
-			stakeRemoval.Reputer,
-			stakeRemoval.Amount,
-		)
+		// do no checking that the stake removal struct is valid. In order to have a stake removal
+		// it would have had to be created in msgServer.RemoveDelegateStake which would have done
+		// validation of validity up front before scheduling the delay
+
+		// Check the module has enough funds to send back to the sender
+		// Bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
+		// Send the funds
+		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
+		err = k.SendCoinsFromModuleToAccount(sdkCtx, emissionstypes.AlloraStakingAccountName, stakeRemoval.Delegator, coins)
 		if err != nil {
 			sdkCtx.Logger().Error(fmt.Sprintf(
 				"Error removing stake: %v | %v",
@@ -114,5 +121,7 @@ func RemoveDelegateStakes(
 			))
 			continue
 		}
+
+		write()
 	}
 }
```
