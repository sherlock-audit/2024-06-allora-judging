Wobbly Rouge Buffalo

Medium

# Malicious peer can cause a syncing node to panic during blocksync

## Summary
The system uses a vulnerable CometBFT version. An attacker can abuse this to cause a panic during blocksync.

## Vulnerability Detail
In `go.mod`, the CometBFT version is pinned to `v0.38.6`. This version is vulnerable to [GO-2024-2951](https://pkg.go.dev/vuln/GO-2024-2951). More details about the vulnerability are available [here](https://github.com/cometbft/cometbft/security/advisories/GHSA-hg58-rf2h-6rr7).

## Impact
The vulnerability allows an attacker to DoS the network by causing panics in all nodes during blocksync.

## Code Snippet

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/cmd/allorad/main.go#L15

The vulnerability itself is in CometBFT, but the following calls in Allora call into vulnerable code:

       #1: cmd/allorad/main.go:15:26: allorad.main calls cmd.Execute, which eventually calls blocksync.BlockPool.OnStart
       #2: cmd/allorad/main.go:15:26: allorad.main calls cmd.Execute, which eventually calls blocksync.NewReactor
       #3: cmd/allorad/main.go:15:26: allorad.main calls cmd.Execute, which eventually calls blocksync.Reactor.OnStart
       #4: cmd/allorad/main.go:15:26: allorad.main calls cmd.Execute, which eventually calls blocksync.Reactor.Receive
       #5: cmd/allorad/main.go:15:26: allorad.main calls cmd.Execute, which eventually calls blocksync.Reactor.SwitchToBlockSync

## Tool used

Manual Review

## Recommendation
Update CometBFT to `v0.38.8`.