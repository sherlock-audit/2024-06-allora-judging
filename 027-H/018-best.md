Keen Spruce Condor

High

# Inconsistency in Stake Removal Delay Calculation

## Summary

This discrepancy means that the actual block time will be shorter than the assumed 5 seconds used in calculating the `RemoveStakeDelayWindow`.


## Vulnerability Detail

In the default parameters, the `RemoveStakeDelayWindow` is set based on an assumed 5-second block time:

```go
RemoveStakeDelayWindow: int64((60 * 60 * 24 * 7 * 3) / 5), // ~approx 3 weeks assuming 5 second block time
```

However, in the configuration script, the `timeout_commit` and `timeout_propose` values are being set to 1 second:

```bash
sed -i -e 's/timeout_commit = "5s"/timeout_commit = "1s"/g' $CHAIN_DIR/$CHAINID/config/config.toml
sed -i -e 's/timeout_propose = "3s"/timeout_propose = "1s"/g' $CHAIN_DIR/$CHAINID/config/config.toml
```

## Impact

1. The stake removal delay will be shorter than intended. Instead of the planned ~3 weeks, it could be as short as ~0.6 weeks (assuming a 1-second block time).
2. This shorter delay could potentially impact the security and stability of the staking mechanism.
3. Users might be able to remove their stake much quicker than the system design intended, which could affect the overall economics and security of the network.


## Code Snippet

[params.go#L17](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/types/params.go#L17)

```go
// DefaultParams returns default module parameters.
func DefaultParams() Params {
	return Params{
		Version:                         "0.0.3",                                   // version of the protocol should be in lockstep with github release tag version
		MinTopicWeight:                  alloraMath.MustNewDecFromString("100"),    // total weight for a topic < this => don't run inference solicatation or loss update
		MaxTopicsPerBlock:               uint64(128),                               // max number of topics to run cadence for per block
		RequiredMinimumStake:            cosmosMath.NewInt(100),                    // minimum stake required to be a worker or reputer
		RemoveStakeDelayWindow:          int64((60 * 60 * 24 * 7 * 3) / 5),         // ~approx 3 weeks assuming 5 second block time, number of blocks to wait before finalizing a stake withdrawal
		MinEpochLength:                  12,                                        // shortest number of blocks per epoch topics are allowed to set as their cadence
		BetaEntropy:                     alloraMath.MustNewDecFromString("0.25"),   // controls resilience of reward payouts against copycat workers
```

## Tool used

Manual Review

## Recommendation

1. Adjust the `RemoveStakeDelayWindow` calculation to match the actual expected block time. If a 1-second block time is intended, update the calculation to:

   ```go
   RemoveStakeDelayWindow: int64(60 * 60 * 24 * 7 * 3), // 3 weeks with 1 second block time
   ```

2. Alternatively, if a 3-week delay is crucial, increase the value to maintain the intended duration with faster block times:

   ```go
   RemoveStakeDelayWindow: int64((60 * 60 * 24 * 7 * 3)), // 3 weeks regardless of block time
   ```

3. Consider making the `RemoveStakeDelayWindow` configurable based on the actual block time, either through a dynamic calculation or by expressing it in time units (e.g., hours or days) rather than blocks.

4. Ensure that all time-based calculations in the system are consistent with the actual block time configuration to prevent similar discrepancies in other parts of the system.

5. Add clear documentation about the relationship between block time configuration and time-based parameters like `RemoveStakeDelayWindow` to prevent future misunderstandings.