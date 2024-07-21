Dry Sepia Dinosaur

Medium

# Some Iterators are not closed in emissions module Keeper

## Summary
Some Iterators are not closed in emissions module Keeper

## Vulnerability Detail
`GetStakeRemovalsForBlock`, `GetDelegateStakeRemovalsForBlock`, `GetInferenceScoresUntilBlock`, `GetForecastScoresUntilBlock` do not Close the Iterator they create.

For reference : https://docs.cosmos.network/main/build/packages/collections#iterateaccounts

## Impact
Unless `Keys`, `Values`, `KeyValues` or `Walk` are used, a collection Iterator must be explicitly closed and here they aren't.

## Code Snippet
[GetStakeRemovalsForBlock](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1269-L1287)
```golang
func (k *Keeper) GetStakeRemovalsForBlock(
	ctx context.Context,
	blockHeight BlockHeight,
) ([]types.StakeRemovalInfo, error) {
	ret := make([]types.StakeRemovalInfo, 0)
	rng := collections.NewPrefixedTripleRange[BlockHeight, TopicId, ActorId](blockHeight)
	iter, err := k.stakeRemovalsByBlock.Iterate(ctx, rng) // <===== Audit
	if err != nil {
		return ret, err
	}
	for ; iter.Valid(); iter.Next() {
		val, err := iter.Value()
		if err != nil {
			return ret, err
		}
		ret = append(ret, val)
	}
	return ret, nil
}
```

[GetDelegateStakeRemovalsForBlock](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1373-L1391)
```golang
func (k *Keeper) GetDelegateStakeRemovalsForBlock(
	ctx context.Context,
	blockHeight BlockHeight,
) ([]types.DelegateStakeRemovalInfo, error) {
	ret := make([]types.DelegateStakeRemovalInfo, 0)
	rng := NewSinglePrefixedQuadrupleRange[BlockHeight, TopicId, ActorId, ActorId](blockHeight)
	iter, err := k.delegateStakeRemovalsByBlock.Iterate(ctx, rng)  // <===== Audit
	if err != nil {
		return ret, err
	}
	for ; iter.Valid(); iter.Next() {
		val, err := iter.Value()
		if err != nil {
			return ret, err
		}
		ret = append(ret, val)
	}
	return ret, nil
}
```

[GetInferenceScoresUntilBlock](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1885-L1917)
```golang
func (k *Keeper) GetInferenceScoresUntilBlock(ctx context.Context, topicId TopicId, blockHeight BlockHeight) ([]*types.Score, error) {
	rng := collections.
		NewPrefixedPairRange[TopicId, BlockHeight](topicId).
		EndInclusive(blockHeight).
		Descending()

	scores := make([]*types.Score, 0)
	iter, err := k.infererScoresByBlock.Iterate(ctx, rng)  // <===== Audit
	if err != nil {
		return nil, err
	}

	// Get max number of time steps that should be retrieved
	moduleParams, err := k.GetParams(ctx)
	if err != nil {
		return nil, err
	}
	maxNumTimeSteps := moduleParams.MaxSamplesToScaleScores

	count := 0
	for ; iter.Valid() && count < int(maxNumTimeSteps); iter.Next() {
		existingScores, err := iter.KeyValue()
		if err != nil {
			return nil, err
		}
		for _, score := range existingScores.Value.Scores {
			scores = append(scores, score)
			count++
		}
	}

	return scores, nil
}
```

[GetForecastScoresUntilBlock](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/keeper.go#L1954-L1986)
```golang
func (k *Keeper) GetForecastScoresUntilBlock(ctx context.Context, topicId TopicId, blockHeight BlockHeight) ([]*types.Score, error) {
	rng := collections.
		NewPrefixedPairRange[TopicId, BlockHeight](topicId).
		EndInclusive(blockHeight).
		Descending()

	scores := make([]*types.Score, 0)
	iter, err := k.forecasterScoresByBlock.Iterate(ctx, rng)  // <===== Audit
	if err != nil {
		return nil, err
	}

	// Get max number of time steps that should be retrieved
	moduleParams, err := k.GetParams(ctx)
	if err != nil {
		return nil, err
	}
	maxNumTimeSteps := moduleParams.MaxSamplesToScaleScores

	count := 0
	for ; iter.Valid() && count < int(maxNumTimeSteps); iter.Next() {
		existingScores, err := iter.KeyValue()
		if err != nil {
			return nil, err
		}
		for _, score := range existingScores.Value.Scores {
			scores = append(scores, score)
			count++
		}
	}

	return scores, nil
}
```

## Tool used
Manual Review

## Recommendation
Explicitly Close the Iterators with "defer iter.Close()"
