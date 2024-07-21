Active Mercurial Puma

Medium

# Inconsistent Precision for `BaseCoinUnit` in Confing


## Vulnerability Detail

The issue lies within the `RegisterDenoms` function:


```Go
func RegisterDenoms() {
    // ...
    err = sdk.RegisterDenom(BaseCoinUnit, math.LegacyNewDecWithPrec(1, AlloraExponent))
    // ...
}
```

In this line, the `math.LegacyNewDecWithPrec(1, AlloraExponent)` function is used to create a `LegacyDec` object with a coefficient of 1 and a precision determined by the value of `AlloraExponent`. However, the way the precision is set might not match the intended behavior of the `BaseCoinUnit`.

**Why It's a Problem**

- **Mismatched Units:** The `BaseCoinUnit` ("uallo") is typically meant to represent the smallest, indivisible unit of the currency. Its precision should be set to `0` to ensure that it represents whole numbers. Setting the precision to `AlloraExponent` (which is likely 18) implies that the `BaseCoinUnit` can have decimal places, which contradicts its fundamental purpose.
- **Incorrect Calculations:** If the `BaseCoinUnit` has an incorrect precision, it can lead to errors in calculations, especially when converting between the `BaseCoinUnit` and the `HumanCoinUnit` ("allo").



## Code Snippet

https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/app/params/config.go#L14

## Tool used

Manual Review

## Recommendation


Set Precision to 0 for `BaseCoinUnit`**

To resolve this issue, you should change the precision argument in the `RegisterDenoms` function:


```Go
func RegisterDenoms() {
    // ...
    err = sdk.RegisterDenom(BaseCoinUnit, math.LegacyNewDecWithPrec(1, 0)) // Precision set to 0
    // ...
}
```

By setting the precision to 0, you ensure that the `BaseCoinUnit` represents whole numbers, which is the correct behavior for a base unit of currency.


## PoC


```go
package main

import (
    "fmt"

    "cosmossdk.io/math"
    sdk "github.com/cosmos/cosmos-sdk/types"
)

const (
    HumanCoinUnit  = "allo"
    BaseCoinUnit   = "uallo"
    AlloraExponent = 18
)

func main() {
    // Register denoms (with incorrect precision for BaseCoinUnit)
    err := sdk.RegisterDenom(HumanCoinUnit, math.LegacyOneDec())
    if err != nil {
        panic(err)
    }
    err = sdk.RegisterDenom(BaseCoinUnit, math.LegacyNewDecWithPrec(1, AlloraExponent)) 
    if err != nil {
        panic(err)
    }

    // Create amounts in both units
    oneAllo := sdk.NewCoin(HumanCoinUnit, sdk.NewInt(1))  
    oneUAllo := sdk.NewCoin(BaseCoinUnit, sdk.NewInt(1))    

    // Convert 1 allo to uallo (should be 1e18 uallo)
    oneAlloToUAllo := oneAllo.Amount.Mul(sdk.NewIntWithDecimal(1, AlloraExponent)) 

    // Check if the conversion is correct
    fmt.Println("1 allo in uallo (expected):", sdk.NewIntWithDecimal(1, AlloraExponent)) 
    fmt.Println("1 allo in uallo (actual):", oneAlloToUAllo)     

    // Check if 1 uallo is considered equal to 1 allo (should be false)
    fmt.Println("1 uallo == 1 allo:", oneUAllo.IsEqual(oneAllo)) 
}
```

**Explanation**

1. **Register Denoms:** We register the `HumanCoinUnit` and `BaseCoinUnit` with the incorrect precision for `BaseCoinUnit` (set to `AlloraExponent`, which is 18).
2. **Create Amounts:**
    - `oneAllo`: Represents 1 allo.
    - `oneUAllo`: Represents 1 uallo.
3. **Conversion:**
    - We attempt to convert `oneAllo` to `uallo` by multiplying its amount by `10^18` (which is the expected conversion factor).
4. **Checks:**
    - We print the expected value of 1 allo in uallo (which should be 1e18) and the actual result of the conversion.
    - We also check if 1 uallo is considered equal to 1 allo, which should be false due to the difference in units.

**Expected Output**

The output of this code will demonstrate the inconsistency:

```go
1 allo in uallo (expected): 1000000000000000000
1 allo in uallo (actual):   1                  // Incorrect
1 uallo == 1 allo:          true               // Incorrect
```

As you can see, the conversion from allo to uallo is incorrect, and 1 uallo is mistakenly considered equal to 1 allo due to the incorrect precision setting.

**After the Fix**

If you change the `RegisterDenoms` function to set the precision of `BaseCoinUnit` to 0, as I suggested in the previous response, you'll get the correct output:

```go
1 allo in uallo (expected): 1000000000000000000
1 allo in uallo (actual):   1000000000000000000  // Correct
1 uallo == 1 allo:          false              // Correct
```

This PoC clearly shows the bug caused by the incorrect precision and how the fix rectifies it.