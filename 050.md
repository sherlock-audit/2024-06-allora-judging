Proud Ginger Hare

Medium

# infererValue value not normalized like other values when topicAllowsNegative is false

## Summary
infererValue value not normalized like other values when topicAllowsNegative is false
## Vulnerability Detail
All values inside `func (e *AlloraExecutor) ExecuteFunction` normalized by  `alloraMath.Log10` when `!topicAllowsNegative` is true, but some normalization missed - `infererValue := alloraMath.MustNewDecFromString`
```go
				if responseValue.InfererValue != "" {
					infererValue := alloraMath.MustNewDecFromString(responseValue.InfererValue)
					inference := &types.Inference{
						TopicId:     topicId,
						Inferer:     e.appChain.Address,
						Value:       infererValue, // @audit should be normalized when !topicAllowsNegative
						BlockHeight: alloraBlockHeightCurrent,
					}
					inferenceForecastsBundle.Inference = inference
				}
				// Build Forecast
				if len(responseValue.ForecasterValues) > 0 {
					var forecasterElements []*types.ForecastElement
					for _, val := range responseValue.ForecasterValues {
						decVal := alloraMath.MustNewDecFromString(val.Value)
						if !topicAllowsNegative {
							decVal, err = alloraMath.Log10(decVal)
							if err != nil {
								fmt.Println("Error Log10 forecasterElements: ", err)
								return result, err
							}
						}
```
[allora-inference-base/cmd/node/main.go#L149](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-inference-base/cmd/node/main.go#L149) 
## Impact
flag `topicAllowsNegative` doesn't work properly
## Code Snippet

## Tool used

Manual Review

## Recommendation
```diff
				if responseValue.InfererValue != "" {
					infererValue := alloraMath.MustNewDecFromString(responseValue.InfererValue)
+					if !topicAllowsNegative {
+						infererValue, err = alloraMath.Log10(infererValue)
+						if err != nil {
+						fmt.Println("Error Log10 infererValue: ", err)
+						return result, err
+					}
+				}
					inference := &types.Inference{
						TopicId:     topicId,
						Inferer:     e.appChain.Address,
						Value:       infererValue,
						BlockHeight: alloraBlockHeightCurrent,
					}
```