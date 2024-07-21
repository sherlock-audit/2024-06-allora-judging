Proud Ginger Hare

High

# Current APY is not stable long-term according to whitepaper

## Summary
Current APY is not stable long-term according to whitepaper 
## Vulnerability Detail
According to whitepaper, page 13, there should be `the addition of fee revenue` for APY to be smooth which is not present currently
>Including emission smoothing remedies this jump, resulting in a smooth
evolution of the APY that incentivizes token holders to continue staking their tokens even around major unlocks (dashed
line). Finally, the addition of fee revenue (here assumed at a level of 2% of the unstaked circulating supply per month)
results in a stable APY long-term. The APY remains high, because fee revenue is used to pay out rewards before minting
new tokens.

There is no addition of fee revenue inside BeginBlocker
```go
func BeginBlocker(ctx context.Context, k keeper.Keeper) error {
...
	// if the expected amount of emissions is greater than the balance of the ecosystem module account
	if blockEmission.GT(ecosystemBalance) {
		// mint the amount of tokens required to pay out the emissions
		tokensToMint := blockEmission.Sub(ecosystemBalance)
		coins := sdk.NewCoins(sdk.NewCoin(params.MintDenom, tokensToMint))
		err = k.MintCoins(sdkCtx, coins)
		if err != nil {
			return err
		}
		err = k.MoveCoinsFromMintToEcosystem(sdkCtx, coins)
		if err != nil {
			return err
		}
		// then increment the recorded history of the amount of tokens minted
		err = k.AddEcosystemTokensMinted(ctx, tokensToMint)
		if err != nil {
			return err
		}
	}
	// pay out the computed block emissions from the ecosystem account
	// if it came from collected fees, great, if it came from minting, also fine
	// we pay both reputers and cosmos validators, so each payment should be
	// half as big (divide by two). Integer division truncates, and that's fine.
	validatorCut := vPercent.Mul(blockEmission.ToLegacyDec()).TruncateInt()
	coinsValidator := sdk.NewCoins(sdk.NewCoin(params.MintDenom, validatorCut))
	alloraRewardsCut := blockEmission.Sub(validatorCut)
	coinsAlloraRewards := sdk.NewCoins(sdk.NewCoin(params.MintDenom, alloraRewardsCut))
	err = k.PayValidatorsFromEcosystem(sdkCtx, coinsValidator)
	if err != nil {
		return err
	}
	err = k.PayAlloraRewardsFromEcosystem(sdkCtx, coinsAlloraRewards)
	if err != nil {
		return err
	}
	if updateEmission {
		// set the previous emissions to this block's emissions
		k.PreviousRewardEmissionPerUnitStakedToken.Set(ctx, e_i)
		k.PreviousBlockEmission.Set(ctx, blockEmission)
	}
	return nil
}

```
[x/mint/module/abci.go#L105](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/mint/module/abci.go#L105)
## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
It seems like there should be an additional function like this to get unstacked values and track unstaked numbers
```go
func GetNumUnStakedTokens(ctx context.Context, k Keeper) (math.Int, error) {
	cosmosValidatorsUnStaked, err := k.CosmosValidatorUnStakedSupply(ctx)
	if err != nil {
		return math.Int{}, err
	}
	reputersUnStaked, err := k.emissionsKeeper.GetTotalUnStake(ctx)
	if err != nil {
		return math.Int{}, err
	}
	return cosmosValidatorsUnStaked.Add(reputersUnStaked), nil
}
```

```diff
	if blockEmission.GT(ecosystemBalance) {
		// mint the amount of tokens required to pay out the emissions
		tokensToMint := blockEmission.Sub(ecosystemBalance)
		coins := sdk.NewCoins(sdk.NewCoin(params.MintDenom, tokensToMint))
		err = k.MintCoins(sdkCtx, coins)
		if err != nil {
			return err
		}
		err = k.MoveCoinsFromMintToEcosystem(sdkCtx, coins)
		if err != nil {
			return err
		}
		// then increment the recorded history of the amount of tokens minted
		err = k.AddEcosystemTokensMinted(ctx, tokensToMint)
		if err != nil {
			return err
		}
	}
+	numUnStakedTokens, err := k.GetNumUnStakedTokens(ctx)
+	blockEmission += numUnStakedTokens * 0.02;

	// pay out the computed block emissions from the ecosystem account
	// if it came from collected fees, great, if it came from minting, also fine
	// we pay both reputers and cosmos validators, so each payment should be
	// half as big (divide by two). Integer division truncates, and that's fine.
	validatorCut := vPercent.Mul(blockEmission.ToLegacyDec()).TruncateInt()
	coinsValidator := sdk.NewCoins(sdk.NewCoin(params.MintDenom, validatorCut))
	alloraRewardsCut := blockEmission.Sub(validatorCut)
	coinsAlloraRewards := sdk.NewCoins(sdk.NewCoin(params.MintDenom, alloraRewardsCut))
	err = k.PayValidatorsFromEcosystem(sdkCtx, coinsValidator)
	if err != nil {
		return err
	}
	err = k.PayAlloraRewardsFromEcosystem(sdkCtx, coinsAlloraRewards)
	if err != nil {
		return err
	}
	if updateEmission {
		// set the previous emissions to this block's emissions
		k.PreviousRewardEmissionPerUnitStakedToken.Set(ctx, e_i)
		k.PreviousBlockEmission.Set(ctx, blockEmission)
	}
	return nil
```
