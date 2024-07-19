Keen Spruce Condor

High

# Inconsistent BlocksPerMonth Calculation with Actual Block Time

## Summary

The `BlocksPerMonth` parameter is set based on a 5-second block time, which is inconsistent with the actual 1-second block time configuration. This discrepancy leads to incorrect calculations for monthly operations and emissions.

## Vulnerability Detail

In the default parameters, `BlocksPerMonth` is set to 525,960, assuming a 5-second block time:

```go
BlocksPerMonth: uint64(525960), // ~5 seconds block time, 6311520 per year, 525960 per month
```

However, as previously identified, the actual block time is configured to be 1 second:

Location  = [init-allorad.sh#L64-L65](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/ibc/scripts/init-allorad.sh#L64-L65)

```bash
sed -i -e 's/timeout_commit = "5s"/timeout_commit = "1s"/g' $CHAIN_DIR/$CHAINID/config/config.toml
sed -i -e 's/timeout_propose = "3s"/timeout_propose = "1s"/g' $CHAIN_DIR/$CHAINID/config/config.toml
```

This mismatch results in incorrect calculations for any operations or emissions that rely on the `BlocksPerMonth` parameter. With a 1-second block time, the actual number of blocks per month should be approximately 2,629,800 (assuming a 30.44-day average month).

The inconsistency affects various parts of the system, including:

1. Emission calculations in the `GetEmissionPerMonth` function.
2. Any monthly-based operations or checks in the system.
3. Potentially affecting the timing of reward distributions and other time-based functionalities.

## Impact

1. Incorrect emission rates: The system will distribute rewards at 1/5th the intended rate.
2. Misaligned timeframes: Any monthly-based operations will occur much more frequently than intended.
3. Potential economic imbalances due to faster-than-intended token distribution.

## Code Snippet

[params.go#L44](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/types/params.go#L44)

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
		LearningRate:                    alloraMath.MustNewDecFromString("0.05"),   // speed of gradient descent
		GradientDescentMaxIters:         uint64(10),                                // max iterations on gradient descent
		MaxGradientThreshold:            alloraMath.MustNewDecFromString("0.001"),  // gradient descent stops when gradient falls below this
		MinStakeFraction:                alloraMath.MustNewDecFromString("0.5"),    // minimum fraction of stake that should be listened to when setting consensus listening coefficients
		Epsilon:                         alloraMath.MustNewDecFromString("0.0001"), // 0 threshold to prevent div by 0 and 0-approximation errors
		MaxUnfulfilledWorkerRequests:    uint64(100),                               // maximum number of outstanding nonces for worker requests per topic from the chain; needs to be bigger to account for varying topic ground truth lag
		MaxUnfulfilledReputerRequests:   uint64(100),                               // maximum number of outstanding nonces for reputer requests per topic from the chain; needs to be bigger to account for varying topic ground truth lag
		TopicRewardStakeImportance:      alloraMath.MustNewDecFromString("0.5"),    // importance of stake in determining rewards for a topic
		TopicRewardFeeRevenueImportance: alloraMath.MustNewDecFromString("0.5"),    // importance of fee revenue in determining rewards for a topic
		TopicRewardAlpha:                alloraMath.MustNewDecFromString("0.5"),    // alpha for topic reward calculation; coupled with blocktime, or how often rewards are calculated
		TaskRewardAlpha:                 alloraMath.MustNewDecFromString("0.1"),    // alpha for task reward calculation used to calculate  ~U_ij, ~V_ik, ~W_im
		ValidatorsVsAlloraPercentReward: alloraMath.MustNewDecFromString("0.25"),   // 25% rewards go to cosmos network validators
		MaxSamplesToScaleScores:         uint64(10),                                // maximum number of previous scores to store and use for standard deviation calculation
		MaxTopInferersToReward:          uint64(48),                                // max this many top inferers by score are rewarded for a topic
		MaxTopForecastersToReward:       uint64(6),                                 // max this many top forecasters by score are rewarded for a topic
		MaxTopReputersToReward:          uint64(12),                                // max this many top reputers by score are rewarded for a topic
		CreateTopicFee:                  cosmosMath.NewInt(10),                     // topic registration fee
		MaxRetriesToFulfilNoncesWorker:  int64(1),                                  // max throttle of simultaneous unfulfilled worker requests
		MaxRetriesToFulfilNoncesReputer: int64(3),                                  // max throttle of simultaneous unfulfilled reputer requests
		RegistrationFee:                 cosmosMath.NewInt(10),                     // how much workers and reputers must pay to register per topic
		DefaultPageLimit:                uint64(100),                               // how many topics to return per page during churn of requests
		MaxPageLimit:                    uint64(1000),                              // max limit for pagination
		MinEpochLengthRecordLimit:       int64(3),                                  // minimum number of epochs to keep records for a topic
		MaxSerializedMsgLength:          int64(1000 * 1000),                        // maximum size of data to msg and query server in bytes
		BlocksPerMonth:                  uint64(525960),                            // ~5 seconds block time, 6311520 per year, 525960 per month
		PRewardInference:                alloraMath.NewDecFromInt64(1),             // fiducial value for rewards calculation
		PRewardForecast:                 alloraMath.NewDecFromInt64(3),             // fiducial value for rewards calculation
		PRewardReputer:                  alloraMath.NewDecFromInt64(3),             // fiducial value for rewards calculation
		CRewardInference:                alloraMath.MustNewDecFromString("0.75"),   // fiducial value for rewards calculation
		CRewardForecast:                 alloraMath.MustNewDecFromString("0.75"),   // fiducial value for rewards calculation
		CNorm:                           alloraMath.MustNewDecFromString("0.75"),   // fiducial value for inference synthesis
		TopicFeeRevenueDecayRate:        alloraMath.MustNewDecFromString("0.025"),  // rate at which topic fee revenue decays over time
	}
}
```

## Tool used

Manual Review

## Recommendation

Review and adjust all functions and operations that rely on the `BlocksPerMonth` parameter to ensure they behave correctly with the updated value.
