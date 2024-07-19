Stable Blue Sloth

High

# Loss of Delegated Rewards Due to Malicious Reputer Deregistration

## Summary
The Allora AppChain that allows a  reputer to intentionally deregister from a topic, resulting in the loss of all rewards for delegators. This issue occurs because the reward delegation function checks if the reputer is still registered in the topic, and if not, rewards cannot be claimed. 

## Vulnerability Detail
In the Allora AppChain, reputers and workers are essential roles. Reputers can register to a topic, add stakes, and have other participants delegate stakes to them. However, a vulnerability exists where a reputer can remove their registration from a topic after stakes and delegations have been made. This action results in the inability of delegators to claim their rewards, as the reward delegation function verifies the reputer's registration status before processing the rewards.
https://github.com/allora-network/allora-chain/blob/3a97afe7af027c96749fac7c4327ae85359a61c8/x/emissions/keeper/msgserver/msg_server_registrations.go#L14-L117
```js
// Registers a new network participant to the network for the first time for worker or reputer
func (ms msgServer) Register(ctx context.Context, msg *types.MsgRegister) (*types.MsgRegisterResponse, error) {
    // ... (registration logic)
}

// Remove registration from a topic for worker or reputer
func (ms msgServer) RemoveRegistration(ctx context.Context, msg *types.MsgRemoveRegistration) (*types.MsgRemoveRegistrationResponse, error) {
    // Check if topic exists
    // ... (remove registration logic)
}
```
## Impact
The identified vulnerability allows a malicious reputer to deregister from a topic after stakes and delegations have been made. This will lead Delegators lose their rewards as the system fails to distribute rewards to a non-registered reputer.
https://github.com/allora-network/allora-chain/blob/3a97afe7af027c96749fac7c4327ae85359a61c8/x/emissions/keeper/msgserver/msg_server_stake.go#L271-L278
```js
func (ms msgServer) RewardDelegateStake(ctx context.Context, msg *types.MsgRewardDelegateStake) (*types.MsgRewardDelegateStakeResponse, error) {
	// Check the target reputer exists and is registered
	isRegistered, err := ms.k.IsReputerRegisteredInTopic(ctx, msg.TopicId, msg.Reputer)
	if err != nil {
		return nil, err
	}
	if !isRegistered {
		return nil, types.ErrAddressIsNotRegisteredInThisTopic
	}
	// Existing Code ...
}
```
Here is the gist for the test case. 
https://gist.github.com/abdulsamijay/353b19294877d516cfe5e0b2fc8a2d40
### Result
![image](https://github.com/user-attachments/assets/3693c750-b21d-4791-b0c7-bcfe700a7ad5)


## Tool used
Manual Code Review

## Recommendation
To mitigate this vulnerability, it is recommended to implement a mechanism where rewards are automatically distributed back to the delegators before allowing a reputer to remove their registration from a topic. This ensures that all participants receive their due rewards and prevents malicious actors from exploiting the system.