Dry Sepia Dinosaur

Medium

# RemoveDelegateStake silently handles the error when checking for existing removals

## Summary
RemoveDelegateStake silently handles the error when checking for existing removals

## Vulnerability Detail
`RemoveDelegateStake` silently handles the error when checking for existing removals while `RemoveStake` blocks further processing and returns the error.

## Impact
Delegate stake removal will be created even if an error happens when checking for existing removals.

## Code Snippet
[RemoveDelegateStake](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/msgserver/msg_server_stake.go#L174-L240)
```golang
func (ms msgServer) RemoveDelegateStake(ctx context.Context, msg *types.MsgRemoveDelegateStake) (*types.MsgRemoveDelegateStakeResponse, error) {
	// ...
	removal, found, err := ms.k.GetDelegateStakeRemovalForDelegatorReputerAndTopicId(
		sdkCtx, msg.Sender, msg.Reputer, msg.TopicId,
	)
	if err != nil {
		errorsmod.Wrap(err, "error during finding delegate stake removal") // <===== Audit : Not returned
	}
        // ...
}
```

[RemoveStake](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/msgserver/msg_server_stake.go#L61-L112)
```golang
func (ms msgServer) RemoveStake(ctx context.Context, msg *types.MsgRemoveStake) (*types.MsgRemoveStakeResponse, error) {
	// ...
	removal, found, err := ms.k.GetStakeRemovalForReputerAndTopicId(sdkCtx, msg.Sender, msg.TopicId)
	if err != nil {
		return nil, errorsmod.Wrap(err, "error while searching previous stake removal") // <===== Audit : Returned
	}
	// ...
}
```

## Tool used
Manual Review

## Recommendation
Return the error from `RemoveDelegateStake`
