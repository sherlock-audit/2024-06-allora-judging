Tame Tan Seagull

Medium

# topic_rewards/SafeApplyFuncOnAllActiveEpochEndingTopics used the wrong parameters

## Summary

`SafeApplyFuncOnAllActiveEpochEndingTopics` used the wrong parameters, can lead to acbi deal with too little or too much topic at a time,
handling too many topics, in the worst case, in a halt of the chain.

## Vulnerability Detail

topic_rewards.go `GetAndUpdateActiveTopicWeights` function calls in the `SafeApplyFuncOnAllActiveEpochEndingTopics` function,

`SafeApplyFuncOnAllActiveEpochEndingTopics` function in the last two parameters for `topicPageLimit` and `maxTopicPages`

The number of topics that can be processed at a time is limited to topicPageLimit * maxTopicPages :

```go
    if topicsActive == nil || i > maxTopicPages {
        break
    }
```

But the problem is that call `SafeApplyFuncOnAllActiveEpochEndingTopics` function used the wrong parameters:

```go
	err = SafeApplyFuncOnAllActiveEpochEndingTopics(ctx, k, block, fn, moduleParams.DefaultPageLimit, moduleParams.DefaultPageLimit)
```

The caller USES moduleParams.DefaultPageLimit as maxTopicPages

This limits the number of topics to be processed each time: topicPageLimit * topicPageLimit

This can cause `acbi/EndBlocker` to process too many topics at a time.

acbi/EndBlocker -> rewards.GetAndUpdateActiveTopicWeights -> SafeApplyFuncOnAllActiveEpochEndingTopics -> with error parameters


Another problem is that if DefaultPageLimit > MaxPageLimit, `CalcAppropriatePaginationForUint64Cursor` function, can let the limit = MaxPageLimit:

```go
func (k Keeper) CalcAppropriatePaginationForUint64Cursor(ctx context.Context, pagination *types.SimpleCursorPaginationRequest) (uint64, uint64, error) {
	moduleParams, err := k.GetParams(ctx)
	if err != nil {
		return uint64(0), uint64(0), err
	}
@>  limit := moduleParams.DefaultPageLimit
	cursor := uint64(0)

	if pagination != nil {
		if len(pagination.Key) > 0 {
			cursor = binary.BigEndian.Uint64(pagination.Key)
		}
		if pagination.Limit > 0 {
			limit = pagination.Limit
		}
		if limit > moduleParams.MaxPageLimit {
@>			limit = moduleParams.MaxPageLimit
		}
	}

	return limit, cursor, nil
}
```

However, if maxTopicPages = DefaultPageLimit, there is no such restriction,since `maxTopicPages` is in the outer for loop, the problem is made worse.


## Impact
acbi/EndBlocker handling too many topics, in the worst case, in a halt of the chain.

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/rewards/topic_rewards.go#L206-L213

## Tool used
Manual Review

## Recommendation
Use the correct parameters