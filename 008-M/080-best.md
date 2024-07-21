Keen Spruce Condor

Medium

# Incomplete Topic Processing Due to Continuous Retry on Pagination Error

## Summary

The `SafeApplyFuncOnAllActiveEpochEndingTopics` function continues to the next iteration when failing to get IDs of active topics, potentially causing an infinite loop or skipping all topics.

## Vulnerability Detail

In the current implementation, when `k.GetIdsOfActiveTopics()` fails, the function logs a warning and continues to the next iteration of the main loop. This behavior can lead to repeated failures and potentially skip processing all topics.

Description:
The problematic code section is:

```go
topicsActive, topicPageResponse, err := k.GetIdsOfActiveTopics(ctx, topicPageRequest)
if err != nil {
    Logger(ctx).Warn(fmt.Sprintf("Error getting ids of active topics: %s", err.Error()))
    continue  // This should be 'break'
}
```

This `continue` statement causes the function to retry getting the same page of topic IDs indefinitely if there's a persistent error, without moving to the next page or terminating the loop.

## Impact

1. Potential infinite loop: If the error persists, the function may never terminate.
2. Skipped topic processing: All topics may be skipped if the first page consistently fails to load.
3. Resource waste: Continuous retries of a failing operation waste computational resources.
4. Misleading behavior: The function appears to complete successfully but may not have processed any topics.

## Code Snippet

[topic_rewards.go#L75](https://github.com/allora-network/allora-chain/blob/3a97afe7af027c96749fac7c4327ae85359a61c8/x/emissions/keeper/msgserver/msg_server_demand.go#L51)


```go
// Apply a function on all active topics that also have an epoch ending at this block
// Active topics have more than a globally-set minimum weight, a function of revenue and stake
// "Safe" because bounded by max number of pages and apply running, online operations.
func SafeApplyFuncOnAllActiveEpochEndingTopics(
	ctx sdk.Context,
	k keeper.Keeper,
	block BlockHeight,
	fn func(sdkCtx sdk.Context, topic *types.Topic) error,
	topicPageLimit uint64,
	maxTopicPages uint64,
) error {
	topicPageKey := make([]byte, 0)
	i := uint64(0)
	for {
		topicPageRequest := &types.SimpleCursorPaginationRequest{Limit: topicPageLimit, Key: topicPageKey}
		topicsActive, topicPageResponse, err := k.GetIdsOfActiveTopics(ctx, topicPageRequest)
		if err != nil {
			Logger(ctx).Warn(fmt.Sprintf("Error getting ids of active topics: %s", err.Error()))
			continue
		}

		for _, topicId := range topicsActive {
			topic, err := k.GetTopic(ctx, topicId)
			if err != nil {
				Logger(ctx).Warn(fmt.Sprintf("Error getting topic: %s", err.Error()))
				continue
			}

			if k.CheckCadence(block, topic) {
				// All checks passed => Apply function on the topic
				err = fn(ctx, &topic)
				if err != nil {
					Logger(ctx).Warn(fmt.Sprintf("Error applying function on topic: %s", err.Error()))
					continue
				}
			}
		}

		// if pageResponse.NextKey is empty then we have reached the end of the list
		if topicsActive == nil || i > maxTopicPages {
			break
		}
		topicPageKey = topicPageResponse.NextKey
		i++
	}
	return nil
}
```

## Tool used

Manual Review

## Recommendation

Change the `continue` statement to `break` when failing to get IDs of active topics:

```go
topicsActive, topicPageResponse, err := k.GetIdsOfActiveTopics(ctx, topicPageRequest)
if err != nil {
    Logger(ctx).Error(fmt.Sprintf("Error getting ids of active topics: %s", err.Error()))
    break  // Exit the loop instead of continuing
}
```