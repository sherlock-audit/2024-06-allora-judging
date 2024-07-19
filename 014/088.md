Tame Tan Seagull

High

# InsertBulkReputerPayload can be DoS


## Summary

After discovering the `InsertBulkReputerPayload` transaction, the attacker initiates the same call to reduce the number of `Valuebundles` in the array of `msg.ReputerValueBundles` (keep only one), invalidate most of the Reputer's data.

## Vulnerability Detail

1. `InsertBulkReputerPayload` does not verify `msg.sender`, so anyone can call it.

2. `InsertBulkReputerPayload` passes through` msg.ReputerValueBundles` and verifies the `signature`.

```go
	for _, bundle := range msg.ReputerValueBundles {
		if err := bundle.Validate(); err != nil {
			continue
		}
        .....
    }

func (bundle *ReputerValueBundle) Validate() error {
    .....
	// Check signature from the bundle, throw if invalid!
	pk, err := hex.DecodeString(bundle.Pubkey)
	if err != nil || len(pk) != secp256k1.PubKeySize {
		return errors.Wrapf(sdkerrors.ErrInvalidRequest, "signature verification failed")
	}
	pubkey := secp256k1.PubKey(pk)
    src := make([]byte, 0)
    src, _ = bundle.ValueBundle.XXX_Marshal(src, true)
    if !pubkey.VerifySignature(src, bundle.Signature) {
        return errors.Wrapf(sdkerrors.ErrInvalidRequest, "signature verification failed")
    }
    }
```

But the problem is that attackers can gain access to that data by listening in on the transaction pool,
After discovering `InsertBulkReputerPayload` transaction, the attacker extracts data from one of the `ReputerValueBundle`,
then construct a new request,
If the request initiated by the attacker is executed before the current transaction, the transaction is executed successfully because the attacker's data is valid.

The old transaction, because the attacker's transaction has been executed nonce has been used, the old transaction execution failed.

```go
func (ms msgServer) InsertBulkReputerPayload(
	ctx context.Context,
	msg *types.MsgInsertBulkReputerPayload,
) (*types.MsgInsertBulkReputerPayloadResponse, error) {
    ......
	// Check if the reputer nonce is unfulfilled
	reputerNonceUnfulfilled, err := ms.k.IsReputerNonceUnfulfilled(ctx, msg.TopicId, msg.ReputerRequestNonce.ReputerNonce)
	if err != nil {
		return nil, err
	}
    .....

    // Update the unfulfilled nonces
	_, err = ms.k.FulfillReputerNonce(ctx, msg.TopicId, msg.ReputerRequestNonce.ReputerNonce)
	if err != nil {
		return nil, err
	}

	// Update topic reward nonce
	err = ms.k.SetTopicRewardNonce(ctx, msg.TopicId, msg.ReputerRequestNonce.ReputerNonce.BlockHeight)
	if err != nil {
		return nil, err
	}
    ......
```

## Impact
A DoS attack invalidates data submitted by the Reputer

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/keeper/msgserver/msg_server_losses.go#L19-L216


https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/x/emissions/types/reputer_value_bundle.go#L12-L74

## Tool used

Manual Review

## Recommendation
Validate the msg.sender