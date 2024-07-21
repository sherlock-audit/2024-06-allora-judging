Proud Ginger Hare

High

# anybody can halt chain in `nsertBulkWorkerPayload`

## Summary
anybody can halt chain in `nsertBulkWorkerPayload` by providing `bundle.InferenceForecastsBundle.Inference = nil`
## Vulnerability Detail
Anybody can call `InsertBulkWorkerPayload`

```go
func (ms msgServer) InsertBulkWorkerPayload(ctx context.Context, msg *types.MsgInsertBulkWorkerPayload) (*types.MsgInsertBulkWorkerPayloadResponse, error) {
...
	acceptedInferers, err := verifyAndInsertInferencesFromTopInferers(
		ctx,
		ms,
		msg.TopicId,
		*msg.Nonce,
		msg.WorkerDataBundles,
		moduleParams.MaxTopInferersToReward,
	)
```
[msgserver/msg_server_worker_payload.go#L219](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/msgserver/msg_server_worker_payload.go#L219)
bundle.InferenceForecastsBundle.Inference can be assigned to nil by sender, so there will an error when trying to access its property.
There is no validation for it

```go
func verifyAndInsertInferencesFromTopInferers(
	ctx context.Context,
	ms msgServer,
	topicId uint64,
	nonce types.Nonce,
	// inferences []*types.Inference,
	workerDataBundles []*types.WorkerDataBundle,
	maxTopWorkersToReward uint64,
) (map[string]bool, error) {
...
		if inference.TopicId != topicId ||
			inference.BlockHeight != nonce.BlockHeight {
			errors[workerDataBundle.Worker] = "Worker data bundle does not match topic or nonce"
			continue
		}
```

Iniside validation it will be passed
```go
func (bundle *WorkerDataBundle) Validate() error {
...

	// Validate the inference and forecast of the bundle
	if bundle.InferenceForecastsBundle.Inference == nil && bundle.InferenceForecastsBundle.Forecast == nil {
		return errors.Wrap(sdkerrors.ErrInvalidRequest, "inference and forecast cannot both be nil")
	}
	if bundle.InferenceForecastsBundle.Inference != nil {
		if err := bundle.InferenceForecastsBundle.Inference.Validate(); err != nil {
			return err
		}
	}
	if bundle.InferenceForecastsBundle.Forecast != nil {
		if err := bundle.InferenceForecastsBundle.Forecast.Validate(); err != nil {
			return err
		}
	}

```
## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
Implement the same validation like inside `verifyAndInsertForecastsFromTopForecasters` function
```diff
		inference := workerDataBundle.InferenceForecastsBundle.Inference

		// Check if the topic and nonce are correct
+		if  inference == nil ||
                        inference.TopicId != topicId ||
			inference.BlockHeight != nonce.BlockHeight {
			errors[workerDataBundle.Worker] = "Worker data bundle does not match topic or nonce"
			continue
		}
```