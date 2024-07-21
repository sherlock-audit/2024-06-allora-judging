Proud Ginger Hare

Medium

# incorrect condition for the iterative update of Equation 34

## Summary
incorrect condition for the iterative update of Equation 34
## Vulnerability Detail
According to whitepaper, iterative process should be done until value would not be less 0.001 `< 0.001`
![image](https://github.com/sherlock-audit/2024-06-allora-0xVolodya/assets/6043510/0467f4a1-fcc0-416c-b46b-284a3c74a422)
But in the code it will stop at 0.001`maxGradientThreshold`
```go
	for maxGradient.Gt(maxGradientThreshold) && i < gradientDescentMaxIters {
		copy(oldCoefficients, coefficients)

```
[rewards/rewards_internal.go#L457](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/rewards/rewards_internal.go#L457)
## Impact
Incorrect computation for listening coefficient.
## Code Snippet

## Tool used

Manual Review

## Recommendation
```diff
-	for maxGradient.Gt(maxGradientThreshold) && i < gradientDescentMaxIters {
+	for maxGradient.Gte(maxGradientThreshold) && i < gradientDescentMaxIters {
		copy(oldCoefficients, coefficients)

```