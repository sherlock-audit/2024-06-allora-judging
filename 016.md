Proud Ginger Hare

High

# The formula for forecast normalization differs from the one in the whitepaper.

## Summary
The formula for forecast normalization differs from the one in the whitepaper.
## Vulnerability Detail
According to docs
>We ask that auditors look for both typical security vulnerabilities, but also departures of the codebase from the intentions defined in the whitepaper. This includes but is not limited to functions not correctly implemented or tied together as defined in the whitepaper. This requires an understanding of the math of the whitepaper, which we urge auditors to develop.
Here is the code for formula below
```go
	medianRegrets, err := alloraMath.Median(regrets)
	if err != nil {
		return RegretInformedWeights{}, errorsmod.Wrapf(err, "Error calculating median of regrets")
	}
	medianTimesFTolerance, err := medianRegrets.Mul(p.Tolerance)
	if err != nil {
		return RegretInformedWeights{}, errorsmod.Wrapf(err, "Error calculating median times tolerance")
	}
	// Add tolerance to standard deviation
	stdDevRegretsPlusMedianTimesFTolerance, err := stdDevRegrets.Abs().Add(medianTimesFTolerance)
	if err != nil {
		return RegretInformedWeights{}, errorsmod.Wrapf(err, "Error adding tolerance to standard deviation of regrets")
	}
	stdDevRegretsPlusMedianTimesFTolerancePlusEpsilon, err := stdDevRegretsPlusMedianTimesFTolerance.Add(p.Epsilon)
	if err != nil {
		return RegretInformedWeights{}, errorsmod.Wrapf(err, "Error adding epsilon to standard deviation of regrets")
	}

	// Normalize the regrets and find the max normalized regret among them
	normalizedInfererRegrets := make(map[Worker]Regret)
	maxRegret := alloraMath.ZeroDec()
	maxRegretInitialized := false
	for address, worker := range p.InfererRegrets {
		regretFrac, err := worker.regret.Quo(stdDevRegretsPlusMedianTimesFTolerancePlusEpsilon)
		if err != nil {
			return RegretInformedWeights{}, errorsmod.Wrapf(err, "Error calculating regret fraction")
		}
		normalizedInfererRegrets[address] = regretFrac
		if !maxRegretInitialized {
			maxRegretInitialized = true
			maxRegret = regretFrac
		} else if regretFrac.Gt(maxRegret) {
			maxRegret = regretFrac
		}
	}
```
[synth_palette_weight.go#L33](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/inference_synthesis/synth_palette_weight.go#L33)
Which results in
```math
\hat{R}_{ijk} = \frac{R_{ijk}}{\sigma_j(R_{ijk}) + \mu(R_{ijk}) \cdot \text{Tolerance} + \epsilon}
```

while in whitepaper its this one, equation (8)
```math
\hat{R}_{ijk} = \frac{R_{ijk}}{\sigma_j(R_{ijk}) + \epsilon}

```
## Impact
The incorrect base formula for calculating weights based on regrets
## Code Snippet

## Tool used

Manual Review

## Recommendation
Implement from whitepaper or change whitepaper to one in code