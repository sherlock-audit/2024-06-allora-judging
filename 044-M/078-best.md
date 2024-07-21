Rich Mint Anteater

Medium

# `GetCurrentTopicWeight` returns `topicFeeRevenue` without accounting `additionalRevenue`

## Summary
`GetCurrentTopicWeight` calculates the topic's weight using the sum of `topicFeeRevenue` and `additionalRevenue`, but it returns `topicFeeRevenue` without adding `additionalRevenue`.

## Vulnerability Detail
`GetCurrentTopicWeight` is used to calculate the weight of each topic and return it's `weight` and `topicFeeRevenue`.

The function uses the topic revenue and combines it with `additionalRevenue` to get the total `feeRevenue` and after that it uses it to calculate it's weight.

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/topic_weight.go#L61
```go
	// Get and total topic fee revenue
	topicFeeRevenue, err := k.GetTopicFeeRevenue(ctx, topicId)
	if err != nil {
		return alloraMath.Dec{}, cosmosMath.Int{}, errors.Wrapf(err, "failed to get topic fee revenue")
	}

	// Calc target weight using fees, epoch length, stake, and params
	newFeeRevenue := additionalRevenue.Add(topicFeeRevenue)
	feeRevenue, err := alloraMath.NewDecFromSdkInt(newFeeRevenue)
	if err != nil {
		return alloraMath.Dec{}, cosmosMath.Int{}, errors.Wrapf(err, "failed to convert topic fee revenue to dec")
	}

	if !feeRevenue.Equal(alloraMath.ZeroDec()) {
		// (topicStakeDec^stakeImportance) * ((feeRevenue / topicEpochLength)^feeImportance)
		targetWeight, err := k.GetTargetWeight(
			topicStakeDec,
			topicEpochLength,
			feeRevenue,
			stakeImportance,
			feeImportance,
		)
```

Later it returns it's estimated average weight (`targetWeight * 50% + previous weight * 50%`) and the total revenue.

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/topic_weight.go#L86
```go
		// Take EMA of target weight with previous weight
		previousTopicWeight, noPrior, err := k.GetPreviousTopicWeight(ctx, topicId)
		if err != nil {
			return alloraMath.Dec{}, cosmosMath.Int{}, errors.Wrapf(err, "failed to get previous topic weight")
		}

		weight, err = alloraMath.CalcEma(topicRewardAlpha, targetWeight, previousTopicWeight, noPrior)
		if err != nil {
			return alloraMath.Dec{}, cosmosMath.Int{}, errors.Wrapf(err, "failed to calculate EMA")
		}

		// weight = targetWeight * 50% + prev weight * 50%
		return weight, topicFeeRevenue, nil
```

However the issue we face here is that our weight is calculated from the sum of `topicFeeRevenue` and `additionalRevenue`, but the returned revenue is only `topicFeeRevenue`. Causing a discrepancy that will:

1. Increase weight - as we are calculating it with a bigger revenue
2. Decrease revenue - as we are returning only the initial revenue

This function is used inside `activateTopicIfWeightAtLeastGlobalMin` to check if a topic can be activated, `GetAndUpdateActiveTopicWeights` to update topic weights and activate any that need be and in `GetTopic`  to be queried by outside entities.

## Impact
Returning the wrong revenue will cause internal accounting errors.

## Code Snippet
```go
    newFeeRevenue := additionalRevenue.Add(topicFeeRevenue)
    feeRevenue, err := alloraMath.NewDecFromSdkInt(newFeeRevenue)
 
    if !feeRevenue.Equal(alloraMath.ZeroDec()) {

        ...
        return weight, topicFeeRevenue, nil
    }
```

## Tool used
Manual Review

## Recommendation
Return `newFeeRevenue` instead of `topicFeeRevenue`.

```diff
-    return weight, topicFeeRevenue, nil
+    return weight, newFeeRevenue, nil
```