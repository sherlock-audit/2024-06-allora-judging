Plain Graphite Scallop

High

# Non-Deterministic Map Range Iteration

## Summary

This is a bug report being filed by the allora team. We found instances in the allora-chain inference-synthesis package of iteration over a golang map using the range function. This function is non-deterministic in how it iterates (uses a psuedorandom order to return the iteration) which, over a long period of time, would cause different nodes to iterate in different orders, and cause nodes to disagree on chain-state which would eventually lead to a chain-halt. We fixed this in PR https://github.com/allora-network/allora-chain/pull/408

## Vulnerability Detail

When you call the golang built-in `range` function over the standard language type of `map` the resulting order of iterations is not guaranteed to be in any deterministic order. Different golang implementations and different computers may return any iteration order they wish. This is a problem when doing math that it is then put into the blockchain's state machine - the order of insertions or edits to the IAVL state tree stored in the database will be different on different nodes, leading to a disagreement about the final hash of the block produced by the blockchain. This would lead ultimately to a chain halt or fork of the chain based on this disagreement over order. 

## Impact

High impact, as this could cause a denial of service of the entire blockchain.

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/keeper/inference_synthesis/common.go#L39-L41

many more such cases as well. See the PR for the full list. https://github.com/allora-network/allora-chain/pull/408

## Tool used

This was discovered by manual review while looking at other code.

## Recommendation

Map iteration should populate an array, which should then be sorted into a deterministic order, and then returned as an iterator over that order. This is done in PR https://github.com/allora-network/allora-chain/pull/408
