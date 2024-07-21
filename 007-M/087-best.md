Brave Tweed Ferret

Medium

# Potential race conditions due to usage of ````sdk.Context```` in concurrent goroutines

## Summary
The implementation of the ````TopicsHandler```` methods uses ````sdk.Context```` in concurrent goroutines, which can lead to race conditions and unexpected results.

## Vulnerability Detail
In the ````TopicsHandler```` struct, both ````requestTopicWorkers```` and ````requestTopicReputers```` methods use the ````sdk.Context```` object within goroutines. ````sdk.Context```` is not thread-safe, and concurrent access to it can lead to race conditions and data corruption.

Here's an example of the problematic code:
```go
func (th *TopicsHandler) requestTopicWorkers(ctx sdk.Context, topic emissionstypes.Topic) {
	...
	go generateInferencesRequest(ctx, topic.InferenceLogic, topic.InferenceMethod, topic.DefaultArg, topic.Id, topic.AllowNegative, *nonceCopy)
}

func (th *TopicsHandler) requestTopicReputers(ctx sdk.Context, topic emissionstypes.Topic) {
	...
	go generateLossesRequest(ctx, reputerValueBundle, topic.LossLogic, topic.LossMethod, topic.Id, topic.AllowNegative, *nonceCopy.ReputerNonce, *nonceCopy.WorkerNonce, previousBlockApproxTime)
}
```
In the ````PrepareProposalHandler```` method, these functions are called within goroutines, passing the ````ctx```` object:
```go
func (th *TopicsHandler) PrepareProposalHandler() sdk.PrepareProposalHandler {
	return func(ctx sdk.Context, req *abci.RequestPrepareProposal) (*abci.ResponsePrepareProposal, error) {
		...
		for _, churnableTopicId := range churnableTopics {
			wg.Add(1)
			go func(topicId TopicId) {
				defer wg.Done()
				topic, err := th.emissionsKeeper.GetTopic(ctx, topicId)
				if err != nil {
					Logger(ctx).Error("Error getting topic: " + err.Error())
					return
				}
				th.requestTopicWorkers(ctx, topic)
				th.requestTopicReputers(ctx, topic)
			}(churnableTopicId)
		}
		wg.Wait()
		return &abci.ResponsePrepareProposal{Txs: req.Txs}, nil
	}
}

```
Using ````ctx```` in multiple goroutines without synchronization can cause race conditions.

## Impact
The use of sdk.Context in concurrent goroutines can lead to several issues:
    (1) **Race Conditions**: Concurrent access to ctx can result in inconsistent or corrupted state, leading to unpredictable behavior and potential application crashes.
    (2) **Data Corruption**: Shared state within ctx may be modified by multiple goroutines simultaneously, causing data corruption and unexpected results.

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/app/topics_handler.go#L77

## Tool used

Manual Review

## Recommendation
To avoid race conditions and ensure thread safety, create a copy of ````sdk.Context```` for each goroutine. The ````WithContext```` method can be used to create a new context from an existing one.