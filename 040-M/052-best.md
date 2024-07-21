Proud Ginger Hare

High

# SendDataWithRetry doesn't work properly(Retries will not happen)

## Summary
Cosmos's `BroadcastTx` returns an error in the first argument instead of the second one.
## Vulnerability Detail
Here is the current Allora implementation for `.BroadcastTx(` where its trying to handle retries
```go
func (ap *AppChain) SendDataWithRetry(ctx context.Context, req sdktypes.Msg, MaxRetries, MinDelay, MaxDelay int, SuccessMsg string) (*cosmosclient.Response, error) {
	var txResp *cosmosclient.Response
	var err error
	for retryCount := 0; retryCount <= MaxRetries; retryCount++ {
		txResponse, err := ap.Client.BroadcastTx(ctx, ap.Account, req)
		txResp = &txResponse
		if err == nil {
			ap.Logger.Info().Str("Tx Hash:", txResp.TxHash).Msg("Success: " + SuccessMsg)
			break
		}
		// Log the error for each retry.
		ap.Logger.Info().Err(err).Msgf("Failed: "+SuccessMsg+", retrying... (Retry %d/%d)", retryCount, MaxRetries)
		// Generate a random number between MinDelay and MaxDelay
		randomDelay := rand.Intn(MaxDelay-MinDelay+1) + MinDelay
		// Apply exponential backoff to the random delay
		backoffDelay := randomDelay << retryCount
		// Wait for the calculated delay before retrying
		time.Sleep(time.Duration(backoffDelay) * time.Second)
	}
	return txResp, err
}
```
[cmd/node/appchain.go#L311](https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-inference-base/cmd/node/appchain.go#L311)

Here is how `BroadcastTx` function works under the hood where it returns errors in the first argument and nill as the second variable which will pass in the current Allora implementation
```go
func (ctx Context) BroadcastTxAsync(txBytes []byte) (*sdk.TxResponse, error) {
	node, err := ctx.GetNode()
	if err != nil {
		return nil, err
	}

	res, err := node.BroadcastTxAsync(context.Background(), txBytes)
	if errRes := CheckCometError(err, txBytes); errRes != nil {
		return errRes, nil // @audit returns errors in first argument
	}

	return sdk.NewResponseFormatBroadcastTx(res), err
}
```
[/client/broadcast.go#L99](https://github.com/cosmos/cosmos-sdk/blob/a923e13ce96568807e0b6b162d4d65bfcef1fef8/client/broadcast.go#L99)
## Impact
Retries will not happen, error will be passed
## Code Snippet

## Tool used

Manual Review

## Recommendation
Copy how its done in sei
```go
		grpcRes, err := TX_CLIENT.BroadcastTx(
			context.Background(),
			&typestx.BroadcastTxRequest{
				Mode:    typestx.BroadcastMode_BROADCAST_MODE_SYNC,
				TxBytes: txBytes,
			},
		)
		if err != nil {
			panic(err)
		}
		for grpcRes.TxResponse.Code == sdkerrors.ErrMempoolIsFull.ABCICode() {
			// retry after a second until either succeed or fail for some other reason
			fmt.Printf("Mempool full\n")
			time.Sleep(1 * time.Second)
			grpcRes, err = TX_CLIENT.BroadcastTx(
				context.Background(),
				&typestx.BroadcastTxRequest{
					Mode:    typestx.BroadcastMode_BROADCAST_MODE_SYNC,
					TxBytes: txBytes,
				},
			)
			if err != nil {
				panic(err)
			}
		}
		if grpcRes.TxResponse.Code != 0 {
			fmt.Printf("Error: %d\n", grpcRes.TxResponse.Code)
		} else {
			mu.Lock()
			defer mu.Unlock()
			TX_HASH_FILE.WriteString(fmt.Sprintf("%s\n", grpcRes.TxResponse.TxHash))
		}
```
[/loadtest/tx.go#L16](https://github.com/mmc6185/sei-chain/blob/0e313eb8f30977b558082580d8a1c628dcc312f5/loadtest/tx.go#L38)