Active Mercurial Puma

Medium

# Inconsistent `valueBundles` Data in `MsgInsertBulkReputerPayload`


## Vulnerability Detail


The issue arises in the `SendReputerModeData` function, specifically in how the `valueBundles` slice is populated and used:


```Go
req := &emissionstypes.MsgInsertBulkReputerPayload{
    Sender: ap.Address,
    ReputerRequestNonce: &emissionstypes.ReputerRequestNonce{
        ReputerNonce: nonceCurrent,
    },
    TopicId:           topicId,
    ReputerValueBundles: valueBundles, // <-- This is the problematic line
}
```

The `valueBundles` slice is initially populated with all `ReputerValueBundle` objects received from reputers, regardless of whether their `blockCurrentHeight` matches the selected `blockCurrentHeight`. Later, a filtered slice `valueBundlesFiltered` is created, containing only the bundles with the correct block height.

However, the code then proceeds to use the original, unfiltered `valueBundles` slice in the `MsgInsertBulkReputerPayload` request. This means that the request might include `ReputerValueBundle` objects with incorrect block heights, leading to potential inconsistencies or errors when processed on the blockchain.

**Why It's a Problem**

- Sending incorrect data to the blockchain can compromise the integrity of the data stored on-chain and potentially lead to incorrect calculations or rewards.
- If different nodes receive different sets of `valueBundles` (due to the filtering logic), it could lead to consensus issues on the blockchain.


## Code Snippet

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-inference-base/cmd/node/appchain.go#L635

## Tool used

Manual Review

## Recommendation

```go
// ...
req := &emissionstypes.MsgInsertBulkReputerPayload{
    // ... (other fields remain the same)
    ReputerValueBundles: valueBundlesFiltered, // Use the filtered slice
}
// ...
```

**Key Improvements**

- The fix ensures that only the correct and relevant `ReputerValueBundle` objects are sent to the blockchain.
-  By sending consistent data to all nodes, the fix helps maintain consensus on the blockchain.


## PoC

```go
package main

import (
    "fmt"

    emissionstypes "github.com/allora-network/allora-chain/x/emissions/types"
)

func main() {
    // Simulate ReputerValueBundle objects with different block heights
    valueBundles := []*emissionstypes.ReputerValueBundle{
        {ValueBundle: &emissionstypes.ValueBundle{ReputerRequestNonce: &emissionstypes.ReputerRequestNonce{ReputerNonce: &emissionstypes.Nonce{BlockHeight: 10}}}},
        {ValueBundle: &emissionstypes.ValueBundle{ReputerRequestNonce: &emissionstypes.ReputerRequestNonce{ReputerNonce: &emissionstypes.Nonce{BlockHeight: 20}}}},
    }
    blockCurrentHeight := int64(10) // Assume this is the correct block height

    // Filter bundles (this is done correctly in the original code)
    var valueBundlesFiltered []*emissionstypes.ReputerValueBundle
    for _, bundle := range valueBundles {
        if bundle.ValueBundle.ReputerRequestNonce.ReputerNonce.BlockHeight == blockCurrentHeight {
            valueBundlesFiltered = append(valueBundlesFiltered, bundle)
        }
    }

    // Create the request (this is where the bug occurs in the original code)
    req := &emissionstypes.MsgInsertBulkReputerPayload{
        ReputerValueBundles: valueBundles, // Should be valueBundlesFiltered
    }

    // Check if the request contains the incorrect bundle
    for _, bundle := range req.ReputerValueBundles {
        if bundle.ValueBundle.ReputerRequestNonce.ReputerNonce.BlockHeight != blockCurrentHeight {
            fmt.Println("Error: Request contains ReputerValueBundle with incorrect block height!")
            return
        }
    }
    fmt.Println("No error found")
}
```

**Explanation:**

1. **Simulated Data:** We create two `ReputerValueBundle` objects, one with `BlockHeight` 10 (correct) and another with `BlockHeight` 20 (incorrect).
2. **Filtering:** We filter the bundles to keep only the one with `BlockHeight` 10.
3. **Request Creation:** We create a `MsgInsertBulkReputerPayload` and intentionally use the unfiltered `valueBundles` slice, which contains the incorrect bundle.
4. **Check:** We iterate over the bundles in the request and print an error message if any bundle has the wrong block height.

**Expected Output:**

When you run this code, you will see the output:

```go
Error: Request contains ReputerValueBundle with incorrect block height!
```

This output demonstrates that the original code would include the incorrect bundle in the request, confirming the bug.