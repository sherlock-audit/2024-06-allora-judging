Keen Spruce Condor

Medium

# Silent Failure in MustNewDecFromString Can Lead to Node Crashes

## Summary

The use of MustNewDecFromString in the AlloraExecutor's ExecuteFunction method ignores potential errors, which can lead to invalid parameters being passed to the node, causing crashes.

## Vulnerability Detail

In the ExecuteFunction method of the AlloraExecutor, MustNewDecFromString is used to convert string values to decimal types. This function panics if it encounters an error during conversion, rather than returning an error that can be handled gracefully. The code doesn't have any error handling or recovery mechanism for these potential panics.

For example:

```go
infererValue := alloraMath.MustNewDecFromString(responseValue.InfererValue)
```

If responseValue.InfererValue is not a valid decimal string, this will cause a panic, which can crash the node if not caught.

Similar issues exist for other conversions in the code, such as those for forecaster values and various attributed values.

## Impact

- Malicious actors could potentially exploit this to crash nodes by providing invalid input data.
- Unexpected panics can cause the node to crash.

## Code Snippet

[/cmd/node/main.go#L149](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-inference-base/cmd/node/main.go#L149), [/cmd/node/main.go#L162](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-inference-base/cmd/node/main.go#L162), [/cmd/node/main.go#L262](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-inference-base/cmd/node/main.go#L262)

## Tool used

Manual Review

## Recommendation

Consider adding err check on the node software.

```go
infererValue, err := alloraMath.NewDecFromString(responseValue.InfererValue)
if err != nil {
    return result, fmt.Errorf("invalid inferer value: %w", err)
}
```