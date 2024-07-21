Expert Daisy Griffin

Medium

# Topics wont activate even with a sufficient stake

## Summary

Topics wont activate even with a sufficient stake

## Vulnerability Detail

The `AddStake` function is used to add stake to a topic. If the stake amount reaches a certain threshold, the topic gets activated.

```go
err = activateTopicIfWeightAtLeastGlobalMin(ctx, ms, msg.TopicId, msg.Amount)
```

This function however uses the `GetCurrentTopicWeight` function to get the current topic weight. This is calculated with an EMA filter, so the current deposit does not have the full weight. The result is a weighted average of the last topic weight and the current weight including the fresh stake.

```go
weight, err = alloraMath.CalcEma(topicRewardAlpha, targetWeight, previousTopicWeight, noPrior)
```

The topic is only activated if this new weight is higher than the setting in the params.

```go
if newTopicWeight.Gte(params.MinTopicWeight) {
    err = ms.k.ActivateTopic(ctx, topicId)
    if err != nil {
        return err
    }
}
```

So due to the topic using an EMA filter, even if the actual weight of the topic exceeds the threshold, the topic wont get activated since the EMA value will still be below threshold.

At the end of the block, the EMA weights are updated in the `EndBlocker` function.

```go
weights, sumWeight, totalRevenue, err := rewards.GetAndUpdateActiveTopicWeights(sdkCtx, am.keeper, blockHeight)
```

This is where the weights are updated, and after a few blocks the EMA will rise enough to clear the threshold.

However, while the `EndBlocker` function has the functionality to de-activate topics if their weight is below the threshold, it does not have the functionality to activate topics which cross the threshold.

The scenario looks like the following:

Assume the current stake is 0. Alice stakes 100 tokens. The threshold is 0.5. `alpha` for EMA is 0.8.

1. In block 1, Alice deposits 100 tokens. Current weight is = 0x0.5 + 100x0.5 = 50. Since this is below threshold, the `ActivateTopic` function never gets called.
2. Block 2, EMA = 50x0.5 + 100x0.5 = 75.
3. Block 3, EMA = 75x0.5 + 100x0.5 = 87.5. Now the threshold is cleared. However, the endblocker has no functionality to activate the topic.

Thus Alice deposits the necessary tokens but the topic never gets activated.

## Impact

Topic does not get activated immediately on stake deposit due to EMA not clearing the threshold. Since the `EndBlocker` funciton due to turn on topics automatically, the topic will stay deactivated even though its weight is below the threshold.

## Code Snippet

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/rewards/topic_rewards.go#L165-L173

## Tool used

Manual Review

## Recommendation

Consider enabling topics the moment their weight goes above the threshold instead of using an EMA.

If EMA is still desirable, then consider creating a delayed queue for topics which clear the raw weight but not the EMA weight and process them in the Endblocker and activate them once its EMA weight is above the threshold.
