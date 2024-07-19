Narrow Arctic Spider

Medium

# `AlloraPendingRewardForDelegator` module account could have insufficient rewards due to truncation

## Summary
`AlloraPendingRewardForDelegator` module account could have insufficient rewards due to truncation.

## Vulnerability Detail
In `GetRewardForReputerFromTotalReward`, the delegator reward for a reputer is calculated and set to `AlloraPendingRewardForDelegator`.
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/module/rewards/reputer_rewards.go#L182-L214
```go
		if delegatorReward.Gt(alloraMath.NewDecFromInt64(0)) {
			// update reward share
			// new_share = current_share + (reward / total_stake)
			totalDelegatorStakeAmountDec, err := alloraMath.NewDecFromSdkInt(totalDelegatorStakeAmount)
			if err != nil {
				return nil, err
			}
			addShare, err := delegatorReward.Quo(totalDelegatorStakeAmountDec)
			if err != nil {
				return nil, err
			}
			currentShare, err := keeper.GetDelegateRewardPerShare(ctx, topicId, reputer)
			if err != nil {
				return nil, err
			}
			newShare, err := currentShare.Add(addShare)
			if err != nil {
				return nil, err
			}
			err = keeper.SetDelegateRewardPerShare(ctx, topicId, reputer, newShare)
			if err != nil {
				return nil, err
			}
			err = keeper.SendCoinsFromModuleToModule(
				ctx,
				types.AlloraRewardsAccountName,
				types.AlloraPendingRewardForDelegatorAccountName,
				sdk.NewCoins(sdk.NewCoin(params.DefaultBondDenom, delegatorReward.SdkIntTrim())),
			)
			if err != nil {
				return nil, errors.Wrapf(err, "failed to send coins to allora pend reward account")
			}
		}
```
Notice that `addShare = delegatorReward / totalDelegatorStakeAmountDec` with full precision (ie. no truncation) and is added to `DelegateRewardPerShare`, while `delegatorReward.SdkIntTrim() <= addShare * totalDelegatorStakeAmountDec = delegatorReward` is sent to `AlloraPendingRewardForDelegator`. This means the recorded total reward for delegators would exceed the amount held by the reward module account, potentially leading to delegators being unable to withdraw their rewards (and breaking an invariant in the README). The chance of this happening is somewhat offset by the fact that the pending reward calculated and sent to the delegator in `RewardDelegateStake` is also truncated, but the repeated increase in delegator reward relative to the funds in the reward account over multiple blocks would heavily increase the discrepancy. 

## Impact
Rewards account may have insufficient funds to pay out to delegators.

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/module/rewards/reputer_rewards.go#L182-L214

## Tool used

Manual Review

## Recommendation
Truncate `addShare` before adding to `newShare`.