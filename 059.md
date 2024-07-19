Keen Spruce Condor

Medium

# AutoCLI Does Not Expose All Query Methods, Limiting CLI Functionality

## Summary

The AutoCLI configuration in the given code defines a set of RPC command options for various query and transaction methods. However, upon review, it's evident that several query methods present in the keeper are not included in the AutoCLI configuration. This discrepancy means that users cannot access these query functions through the CLI, potentially hindering their ability to interact with and retrieve important information from the blockchain.


## Vulnerability Detail

The current implementation of AutoCLI in the provided code does not expose all available query methods. This omission can lead to reduced functionality and accessibility of certain features through the command-line interface.

## Impact

1. Reduced CLI functionality: Users are unable to access all available query methods through the command line.
2. Inconsistent user experience: The disparity between available backend methods and CLI commands may confuse users.
3. Potential underutilization of features: Important query functions that are not exposed may be overlooked or underused.
4. Increased reliance on alternative interfaces: Users may need to resort to other means (e.g., direct API calls) to access missing functionality.

## Code Snippet

[autocli.go#L1](https://github.com/allora-network/allora-chain/blob/3a97afe7af027c96749fac7c4327ae85359a61c8/x/emissions/module/autocli.go#L1)

```go
// AutoCLIOptions implements the autocli.HasAutoCLIConfig interface.
func (am AppModule) AutoCLIOptions() *autocliv1.ModuleOptions {
	return &autocliv1.ModuleOptions{
		Query: &autocliv1.ServiceCommandDescriptor{
			Service: statev1.Query_ServiceDesc.ServiceName,
			RpcCommandOptions: []*autocliv1.RpcCommandOptions{}
```

Unavailable Example Query : 

[keeper.go#L1919-L1920](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1919-L1920)

```go
func (k *Keeper) GetWorkerInferenceScoresAtBlock(ctx context.Context, topicId TopicId, block BlockHeight) (types.Scores, error) {
	key := collections.Join(topicId, block)
	scores, err := k.infererScoresByBlock.Get(ctx, key)
	if err != nil {
		if errors.Is(err, collections.ErrNotFound) {
			return types.Scores{}, nil
		}
		return types.Scores{}, err
	}
	return scores, nil
}
```



## Tool used

Manual Review

## Recommendation

1. Conduct a comprehensive review of all available query methods in the keeper.
2. Update the AutoCLI configuration to include all relevant query methods.
3. Ensure that newly added query methods are consistently reflected in the AutoCLI configuration.
4. Consider implementing a process or tooling to automatically sync keeper methods with AutoCLI configurations to prevent future discrepancies.
5. Document all available CLI commands, including newly added ones, to ensure users are aware of the full range of functionality.

