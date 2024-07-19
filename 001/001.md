Keen Spruce Condor

High

# Topic Activation Failure Due to Unhandled Error

## Summary

The `activateTopicIfWeightAtLeastGlobalMin` function returns an error if any of the internal operations encounter an error. However, the returned error is not being properly handled or propagated in the `FundTopic` method.


## Vulnerability Detail

In the provided code snippet, the `activateTopicIfWeightAtLeastGlobalMin` function is called within the `FundTopic` method of the `msgServer` to potentially activate a topic if its weight meets the global minimum threshold. However, there is a risk that the topic may not be activated even if it meets the necessary criteria due to an unhandled error.

The specific issue lies in the following line of code:
```go
err = activateTopicIfWeightAtLeastGlobalMin(ctx, ms, msg.TopicId, msg.Amount)
```

## Impact

If an error occurs during the topic activation process, it will be silently ignored, and the topic may remain inactive even if it should have been activated based on the weight criteria. 

## Code Snippet

[msg_server_demand.go#L51](https://github.com/allora-network/allora-chain/blob/3a97afe7af027c96749fac7c4327ae85359a61c8/x/emissions/keeper/msgserver/msg_server_demand.go#L51)

```go
func (ms msgServer) FundTopic(ctx context.Context, msg *types.MsgFundTopic) (*types.MsgFundTopicResponse, error) {
	// Check the topic is valid
	topicExists, err := ms.k.TopicExists(ctx, msg.TopicId)
	if err != nil {
		return nil, err
	}
	if !topicExists {
		return nil, types.ErrInvalidTopicId
	}

	// Check that the request isn't spam by checking that the amount of funds it bids is greater than a global minimum demand per request
	params, err := ms.k.GetParams(ctx)
	if err != nil {
		return nil, err
	}
	amountDec, err := alloraMath.NewDecFromSdkInt(msg.Amount)
	if err != nil {
		return nil, err
	}
	if amountDec.Lte(params.Epsilon) {
		return nil, types.ErrFundAmountTooLow
	}
	// Check sender has funds to pay for the inference request
	// bank module does this for us in module SendCoins / subUnlockedCoins so we don't need to check
	// Send funds
	coins := sdk.NewCoins(sdk.NewCoin(appParams.DefaultBondDenom, msg.Amount))
	err = ms.k.SendCoinsFromAccountToModule(ctx, msg.Sender, minttypes.EcosystemModuleName, coins)
	if err != nil {
		return nil, err
	}

	// Account for the revenue the topic has generated
	err = ms.k.AddTopicFeeRevenue(ctx, msg.TopicId, msg.Amount)
	if err != nil {
		return nil, err
	}

	// Activate topic if it exhibits minimum weight
	err = activateTopicIfWeightAtLeastGlobalMin(ctx, ms, msg.TopicId, msg.Amount)
	return &types.MsgFundTopicResponse{}, err
}
```


## Tool used

Manual Review

## Recommendation

Modify the `FundTopic` method to properly handle the error returned by `activateTopicIfWeightAtLeastGlobalMin`.
   - If an error occurs during topic activation, it should be logged, and an appropriate error response should be returned to the caller.

   ```go
   err = activateTopicIfWeightAtLeastGlobalMin(ctx, ms, msg.TopicId, msg.Amount)
   if err != nil {
       // Log the error for visibility and debugging purposes
       ms.k.Logger(ctx).Error("Failed to activate topic", "error", err)
       return nil, err
   }
   ```