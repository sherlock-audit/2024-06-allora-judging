Plain Graphite Scallop

Medium

# AlloraInferenceBase Panic when receiving unexpected string responses from WASM

## Summary

This is a bug report filed by the Allora team. We found an issue where the WASM used to run loss-functions for reputer scores were returning error strings. This would in turn cause a panic in the allora-inference base software, preventing the inference base from ever uploading scores for reputation. We fixed this issue in PR https://github.com/allora-network/allora-inference-base/pull/135

## Vulnerability Detail

Workers that produce inferences are given reputation scores that are a product of a "loss function" that grades how good their inferences were relative to a ground truth. This loss function is implemented in WASM, run off-chain, and must be then pushed on-chain by the allora-inference-base. In this case, we found that the WASM would, when experiencing errors, return a string, which would then crash the allora-inference-base software, which expected a numeric.

## Impact

Medium. The inference base would crash and prevent it from doing reputation in the future, thereby preventing reputers in the future from uploading reputation and getting paid.

## Code Snippet

https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-inference-base/cmd/node/main.go#L149

https://github.com/sherlock-audit/2024-06-allora/blob/4e1bc73db32873476f8b0a88945815d3978d931c/allora-inference-base/cmd/node/main.go#L162

See also https://github.com/allora-network/allora-inference-base/pull/135

## Tool used

We discovered this while running a private testnet.

## Recommendation

This bug has been fixed by catching the error and continuing, rather than allowing the error to panic and crash the allora-inference-base. That work is in PR https://github.com/allora-network/allora-inference-base/pull/135
