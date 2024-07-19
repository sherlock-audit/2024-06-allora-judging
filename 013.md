Keen Spruce Condor

High

# Unchecked Error in ResetChurnableTopics Function

## Summary

The `ResetChurnableTopics` function in the keeper does not check the error returned by the `Clear` method of `churnableTopics`. This could lead to silent failures and potential inconsistencies in the system state.

## Vulnerability Detail

In the current implementation of `ResetChurnableTopics`:

```go
func (k *Keeper) ResetChurnableTopics(ctx context.Context) error {
    k.churnableTopics.Clear(ctx, nil)
    return nil
}
```

The `Clear` method is called on `k.churnableTopics`, but its return value (which could be an error) is not checked. The function always returns `nil`, regardless of whether the clearing operation succeeded or failed.

## Impact

1. Silent Failures: If the `Clear` operation fails for any reason (e.g., concurrency problems), the failure will go unnoticed.
2. Inconsistent State: The system may continue operating under the assumption that the churnable topics have been reset when they haven't, leading to incorrect behavior in subsequent operations.
3. Without cleaning churnable topics, The operation is going to continue on the Endblocker. 

## Code Snippet

[/x/emissions/keeper/keeper.go#L1750](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1750)

```go
// ResetChurnReadyTopics clears all topics from the churn-ready set and resets related states.
func (k *Keeper) ResetChurnableTopics(ctx context.Context) error {
	k.churnableTopics.Clear(ctx, nil)
	return nil
}
```

## Tool used

Manual Review

## Recommendation

1. Modify the `ResetChurnableTopics` function to check and return the error from the `Clear` method:

```go
func (k *Keeper) ResetChurnableTopics(ctx context.Context) error {
    err := k.churnableTopics.Clear(ctx, nil)
    if err != nil {
        return errors.Wrap(err, "failed to clear churnable topics")
    }
    return nil
}
```

2. Ensure that all calls to `ResetChurnableTopics` properly handle the returned error, for example:

```go
err = am.keeper.ResetChurnableTopics(ctx)
if err != nil {
    sdkCtx.Logger().Error("Error resetting churn ready topics: ", err)
    return errors.Wrapf(err, "Resetting churn ready topics error")
}
```

4. Consider adding logging within the `ResetChurnableTopics` function for successful clears as well, to aid in debugging and monitoring:

```go
func (k *Keeper) ResetChurnableTopics(ctx context.Context) error {
    err := k.churnableTopics.Clear(ctx, nil)
    if err != nil {
        return errors.Wrap(err, "failed to clear churnable topics")
    }
    k.Logger(ctx).Info("Successfully cleared churnable topics")
    return nil
}
```