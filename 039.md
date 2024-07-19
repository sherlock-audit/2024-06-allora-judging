Keen Spruce Condor

Medium

# Inconsistency between NUM_REPUTER_RETRIES constant and MaxRetriesToFulfilNoncesReputer chain parameter

## Summary

The discrepancy between these two values introduces several potential problems:
1. Inconsistent retry behavior across different parts of the system
2. Possible violation of chain-wide settings at the application level
3. Increased difficulty in maintaining and debugging the codebase
4. Potential performance issues if more retries are attempted than necessary

## Vulnerability Detail

There is a discrepancy between the hardcoded constant NUM_REPUTER_RETRIES and the chain parameter MaxRetriesToFulfilNoncesReputer. This inconsistency may lead to confusion and potential issues in the retry logic for reputer operations.

Current state:
- NUM_REPUTER_RETRIES constant: 5
- MaxRetriesToFulfilNoncesReputer chain parameter: 3

This misalignment could result in unexpected behavior, as the code might attempt more retries than the chain parameter suggests is appropriate.

## Impact

1. Inconsistent retry behavior across different parts of the system.

## Code Snippet

[appchain.go#L30](https://github.com/allora-network/allora-inference-base/blob/9630a7d691d48a8b0fbdda34dc1c13c3188b0706/cmd/node/appchain.go#L30)

```go
// Exponential backoff retry settings
const NUM_WORKER_RETRIES = 5
const NUM_REPUTER_RETRIES = 5
const NUM_REGISTRATION_RETRIES = 3
const NUM_STAKING_RETRIES = 3
const NUM_WORKER_RETRY_MIN_DELAY = 0
const NUM_WORKER_RETRY_MAX_DELAY = 2
const NUM_REPUTER_RETRY_MIN_DELAY = 0
const NUM_REPUTER_RETRY_MAX_DELAY = 2
const NUM_REGISTRATION_RETRY_MIN_DELAY = 1
const NUM_REGISTRATION_RETRY_MAX_DELAY = 2
const NUM_STAKING_RETRY_MIN_DELAY = 1
const NUM_STAKING_RETRY_MAX_DELAY = 2
const REPUTER_TOPIC_SUFFIX = "/reputer"
```

[params.go#L38](https://github.com/allora-network/allora-chain/blob/3a97afe7af027c96749fac7c4327ae85359a61c8/x/emissions/types/params.go#L38)

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

To resolve this issue, we recommend the following steps:

1. Determine the correct number of retries that should be used for reputer operations.

2. Update either the constant or the chain parameter to ensure consistency:
   a. If the constant should be authoritative:
      - Update the MaxRetriesToFulfilNoncesReputer chain parameter to 5
      - Document why this value differs from other retry counts in the system
   b. If the chain parameter should be authoritative:
      - Update the NUM_REPUTER_RETRIES constant to 3
      - Alternatively, remove the constant and directly use the chain parameter value in the code
