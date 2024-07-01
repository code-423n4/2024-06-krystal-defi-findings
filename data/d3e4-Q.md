### `_collectFees()` is not used and inconsistent with the behaviour of `_decreaseLiquidityAndCollectFees()`
`_collectFees()` is not used anywhere.
It explicitly and deliberately [reverts for fee-on-transfer tokens](https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L594-L600). `_decreaseLiquidityAndCollectFees()` similarly collects fees but does not revert for fee-on-transfer tokens.
`_collectFees()` may therefore be removed, or perhaps it was intended to be used in `_decreaseLiquidityAndCollectFees()` to make sure fee-on-transfer tokens cause a revert.

### Setting `_maxFeeX64` to `0` forces a revert in `_deductFees()`
```solidity
if (params.feeX64 > _maxFeeX64[params.feeType]) {
    revert TooMuchFee();
}

// to save gas, we always need to check if fee exists before deductFees
if (params.feeX64 == 0) {
    revert NoFees();
}
```
Either it should be ensured that `_maxFeeX64[params.feeType] > 0` by not allowing `setMaxFeeX64()` to set it to `0`, or, perhaps more reasonably, the latter check above should just `return` instead of `revert`.

This is currently not much of an issue since all calls to `_deductFees()` with `params.feeX64 = 0` are skipped.

