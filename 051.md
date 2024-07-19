Plain Graphite Scallop

High

# GetIdsOfActiveTopics Fails to Return Non-Consecutive Values

## Summary

This is a bug report being submitted by the Allora Team. We found an issue where the `GetIdsOfActiveTopics` function would skip returning non-consecutive values of active topics. The attached PR that fixes this issue is https://github.com/allora-network/allora-chain/pull/406.

## Vulnerability Detail

The allora chain implements our own "SimpleCursorPagination" for queries with potentially large amounts of returned data. This cursor pagination would not return the full list of items in cases where the returned values were non-sequential, e.g. topic ids={1,2,3} but 2 is not active => GetIdsOfActiveTopics({limit:2}) would return {1} when it should return {1,3}, for any passed limit≥2.

## Impact

High

Queries to get active topics would not return the correct full list of active topics. This impacts what the off-chain components would work on, thereby preventing some queries from ever getting filled.

## Code Snippet

https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/keeper/keeper.go#L1611-L1633

also see https://github.com/allora-network/allora-chain/pull/406

## Tool used

This bug was manually found during the team's internal testing.

Manual Review

## Recommendation

We already fixed the bug making it iterate over the full list of topics and returning the full list. See https://github.com/allora-network/allora-chain/pull/406