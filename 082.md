Dry Sepia Dinosaur

Medium

# Mint and Emissions modules register errors with an error code of 1

## Summary
Mint and Emissions modules register errors with an error code of 1

## Vulnerability Detail
Mint and Emissions modules register errors with an error code of 1

## Impact
According to the Cosmos SDK [Errors documentation](https://docs.cosmos.network/main/build/building-modules/errors#registration), the error code :

> Must be greater than one, as a code value of one is reserved for internal errors.

This breaks that rule.

## Code Snippet
[mint errors](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/mint/types/errors.go#L6)
```golang
var (
	ErrUnauthorized                                    = errors.Register(ModuleName, 1, "unauthorized message signer")
	// ...
)
```

[emissions errors](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/types/errors.go#L6)
```golang
var (
	ErrTopicReputerStakeDoesNotExist            = errors.Register(ModuleName, 1, "topic reputer stake does not exist")
	// ...
)
```
## Tool used
Manual Review

## Recommendation
Don't use 1 for error codes.
