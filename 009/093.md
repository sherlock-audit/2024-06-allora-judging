Rich Mint Anteater

High

# coefficients math mistakenly calculates the coefficient diff with the same value

## Summary
`GetAllReputersOutput`, calculates each reputer scores and coefficients, however while doing that calculation it mistakenly calculated the coeff diff between the new and old coefficients using the same old value, meaning that the diff will be always 0.

```go
458:    copy(oldCoefficients, coefficients)

...

548:   coeffDiff, err := coefficients[l].Sub(oldCoefficients[l])
```

## Vulnerability Detail
When calculating the coefficient `GetAllReputersOutput` has a custom `if` where if `listenedStakeFraction < minStakeFraction` it will do some math and increase the coefficients by `coefDiffTimesListenedDiffOverStakedFracDiff`.

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/rewards/rewards_internal.go#L563-L574
```go
    coeffDiffTimesListenedDiff, err := coeffDiff.Mul(listenedDiff)
    if err != nil {
        return nil, nil, err
    }

    coefDiffTimesListenedDiffOverStakedFracDiff, err := coeffDiffTimesListenedDiff.Quo(stakedFracDiff)
    if err != nil {
        return nil, nil, err
    }

    coefficients[l], err = oldCoefficients[l].Add(coefDiffTimesListenedDiffOverStakedFracDiff)
    if err != nil {
        return nil, nil, err
    }
```

However that will never happen as before that when we calculate the `coeffDiff` between our new and old coefficients, we use 2 different arrays, but they are copied  with the same parameters - our old coeff. Essentially calculating the `coeffDiff` between our old and old coefficient, resulting in 0 diff 100% of the time. 

It will make `coeffDiffTimesListenedDiff == 0` and `coefDiffTimesListenedDiffOverStakedFracDiff == 0`, making our `coefficient == oldCoefficients`.

This can be seen here where we calculate our diff:
https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/rewards/rewards_internal.go#L548-L551
```go
    //@audit calculate 0 diff as `coefficients[l] == oldCoefficients[l]`
    coeffDiff, err := coefficients[l].Sub(oldCoefficients[l])
    if err != nil {
        return nil, nil, err
    }
```

And in here where we set the `coefficients` and `oldCoefficients` arrays:

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/module/rewards/rewards_internal.go#L448-L458
```go
    coefficients := make([]alloraMath.Dec, len(initialCoefficients))
    copy(coefficients, initialCoefficients)

    oldCoefficients := make([]alloraMath.Dec, numReputers)
    var i uint64 = 0
    var maxGradient alloraMath.Dec = alloraMath.OneDec()

    // finalScores := make([]alloraMath.Dec, numReputers)
    newScores := make([]alloraMath.Dec, numReputers)

    for maxGradient.Gt(maxGradientThreshold) && i < gradientDescentMaxIters {
        // @audit copy `coefficients` into `oldCoefficients`, making them ==
        copy(oldCoefficients, coefficients)

```

## Impact
The custom math for adjusting coeff when `listenedStakeFraction < minStakeFraction` won't actually change anything, as it will set the coeff to it's old value. This is dangerous as our new coeff could have been way smaller or bigger than our old one. This change will impact reputer rewards, as they are calculated based on scores, and score math includes  coefficients. 

## Code Snippet
```go
458:    copy(oldCoefficients, coefficients)

...

548:   coeffDiff, err := coefficients[l].Sub(oldCoefficients[l])
```

## Tool used
Manual Review

## Recommendation
Change the math to get the difference (preferably absolute -`.abs()`) between the new and old coefficients.