Proud Ginger Hare

Medium

# errors are not being handled which can lead to executing code with errors(silencing them) or a halt chain due to panic

## Summary
returns from ".Set" ".Get" are not being handled which can lead to executing code with errors or a halt chain.
## Vulnerability Detail
error from SetDelegateStakePlacement is not handling like it is done in another part of the codebase
```go
		if err != nil {
			return nil, err
		}
		ms.k.SetDelegateStakePlacement(ctx, msg.TopicId, msg.Sender, msg.Reputer, delegateInfo)
	}
```
[msg_server_stake.go#L303](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/msgserver/msg_server_stake.go#L303)

```go
	if err := k.SetDelegateStakePlacement(ctx, topicId, delegator, reputer, stakePlacementNew); err != nil {
		return errorsmod.Wrapf(err, "Setting delegate stake placement failed")
	}
```
[emissions/keeper/keeper.go#L1052](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1052)

Other examples
```go
					am.keeper.PruneReputerNonces(sdkCtx, topic.Id, reputerPruningBlock)

```
[emissions/module/abci.go#L92](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/abci.go#L92)

```go
func (am AppModule) InitGenesis(ctx sdk.Context, cdc codec.JSONCodec, data json.RawMessage) {
	var genesisState types.GenesisState
	cdc.MustUnmarshalJSON(data, &genesisState)

	am.keeper.InitGenesis(ctx, am.authKeeper, &genesisState)
}
```
[x/mint/module/module.go#L126](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/mint/module/module.go#L126)

```go
						am.keeper.PruneWorkerNonces(sdkCtx, topic.Id, workerPruningBlock)

```
[emissions/module/abci.go#L99](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/abci.go#L99)

```go
		k.AddWhitelistAdmin(ctx, addr)
```
[genesis.go#L59](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/genesis.go#L59)

```go
		k.SetInfererNetworkRegret(ctx, topicId, infererLoss.Worker, newInfererRegret)
```
[/inference_synthesis/network_regrets.go#L121](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/inference_synthesis/network_regrets.go#L121)

```diff
func (k *Keeper) ResetChurnableTopics(ctx context.Context) error {
-	k.churnableTopics.Clear(ctx, nil)
-	return nil
+    return k.churnableTopics.Clear(ctx, nil)
}
```
[emissions/keeper/keeper.go#L1749](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1749)

multiple others in code
## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
Implement correct error handling