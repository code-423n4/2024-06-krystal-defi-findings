### Withdrawer granted DEFAULT_ADMIN_ROLE

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/Common.sol#L119

```solidity
    function initialize(address router, address admin, address withdrawer, address feeTaker, address[] calldata whitelistedNfpms) public virtual {
        require(!_initialized);
        if (withdrawer == address(0)) {
            revert();
        }
        require(msg.sender == _initializer);

        _grantRole(ADMIN_ROLE, admin);
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(WITHDRAWER_ROLE, withdrawer);
>>      _grantRole(DEFAULT_ADMIN_ROLE, withdrawer);
        ---SNIP---
    }
```
According to the Krystal docs:
https://docs.krystal.app/technical-docs/security#access-control

> Withdrawers cannot touch any of the LP positions

However, during the contract initialization, the withdrawer is granted the `DEFAULT_ADMIN_ROLE` which allows them to grant themselves the `OPERATOR_ROLE` and thereby control LP tokens that was approved to the `V3Automation.sol` contract.

### `execute` function is defined as payable

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Automation.sol

```solidity
    function execute(ExecuteParams calldata params) public payable onlyRole(OPERATOR_ROLE) whenNotPaused() {
```

The `execute` function includes the `payable` keyword but does not utilize ETH tokens within its logic.

### `_swap` may revert if the `tokenIn` reverts on zero value approvals

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/Common.sol#L551

```solidity
    function _swap(IERC20 tokenIn, IERC20 tokenOut, uint256 amountIn, uint256 amountOutMin, bytes memory swapData) internal returns (uint256 amountInDelta, uint256 amountOutDelta) {
        if (amountIn != 0 && swapData.length != 0 && address(tokenOut) != address(0)) {
            uint256 balanceInBefore = tokenIn.balanceOf(address(this));
            uint256 balanceOutBefore = tokenOut.balanceOf(address(this));

            // approve needed amount
            _safeApprove(tokenIn, swapRouter, amountIn);
            // execute swap
            (bool success,) = swapRouter.call(swapData);
            if (!success) {
                revert ("swap failed!");
            }

            // reset approval
>>          _safeApprove(tokenIn, swapRouter, 0);
```

After a successful swap, the Krystal contract attempts to reset the approval to zero. However, some tokens (like LEND) may have additional logic that reverts on zero-amount approvals. Consider using the `_safeResetAndApprove` function to avoid this issue.
  
https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-approvals

### `amountIn` validation should be after fees were deducted

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L198-L214
https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L244-L256

The Krystal contracts verify that enough tokens for the swap are provided, but in some cases, the verification occurs before fees are deducted. This may lead to a situation where there are insufficient tokens to complete the swap after the fee is taken.

```solidity
        // validate if amount2 is enough for action
>>      if (address(params.swapSourceToken) != token0
            && address(params.swapSourceToken) != token1
            && params.amountIn0 + params.amountIn1 > params.amount2
        ) {
            revert AmountError();
        }

        _prepareSwap(weth, IERC20(token0), IERC20(token1), params.swapSourceToken, params.amount0, params.amount1, params.amount2);
        SwapAndIncreaseLiquidityParams memory _params = params;
>>      if (params.protocolFeeX64 > 0) {
            (_params.amount0, _params.amount1, _params.amount2,,,) = _deductFees(DeductFeesParams(params.amount0, params.amount1, params.amount2, params.protocolFeeX64, FeeType.PROTOCOL_FEE, address(params.nfpm), params.tokenId, params.recipient, token0, token1, address(params.swapSourceToken)), true);
        }
```

Consider verifying swap amounts after fees have been deducted.

### No incentive for users to pay protocol fees in the `V3Utils.sol`

Users can specify the percentage of their tokens to be paid as fees to the protocol during the `execute` call.

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L69-L70
https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/Common.sol#L657

Currently, only the maximum percentage is checked, allowing users who call the `V3Utils.sol` contract directly to specify fees that are close to zero. Implementing minimum fee validation could address this issue:

```diff
    function _deductFees(DeductFeesParams memory params, bool emitEvent) internal returns(uint256 amount0Left, uint256 amount1Left, uint256 amount2Left, uint256 feeAmount0, uint256 feeAmount1, uint256 feeAmount2) {
        uint256 Q64 = 2 ** 64;
+        if (params.feeX64 < _minFeeX64[params.feeType]) {
+            revert FeeTooLow();

```

### Contracts without receive/fallback function may not be able to receive leftovers from `swapAndMint`, `swapAndIncreaseLiquidity` functions

If a non-zero amount of ETH is sent during `swapAndMint` or `swapAndIncreaseLiquidity` calls, the protocol will unwrap leftover WETH tokens and send them to the recipient.

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L230
https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L258

```solidity
result = _swapAndIncrease(_params, IERC20(token0), IERC20(token1), msg.value != 0);
```

The last argument is a bool value, that will unwrap WETH if `true`. Consider allowing user to specify whether he wants to wrap or unwrap his leftover tokens.

### Orders can be replayed

In the current `V3Automation.sol` implementation, orders can be replayed since there are no nonces or any other mechanic that would prevent operator from using the same signature again. Only signer verification is present:

```solidity
    function _validateOrder(StructHash.Order memory order, bytes memory orderSignature, address actor) internal view {
        address userAddress = recover(order, orderSignature);
        require(userAddress == actor);
        require(!_cancelledOrder[keccak256(orderSignature)]);
    }
``` 

Consider implementing a check that would prevent using the same signature twice:

```diff
    function _validateOrder(StructHash.Order memory order, bytes memory orderSignature, address actor) internal view {
        address userAddress = recover(order, orderSignature);
        require(userAddress == actor);
        require(!_cancelledOrder[keccak256(orderSignature)]);
+       require(!_fulfilledOrder[keccak256(orderSignature)]);
    }
``` 

### Orders and params are separate

When the `execute` function is called in the `V3Automation.sol` contract, a set of parameters is passed, but only the `StructHash.Order userOrder` is used in the signature validation:
```solidity
    function execute(ExecuteParams calldata params) public payable onlyRole(OPERATOR_ROLE) whenNotPaused() {
        require(_isWhitelistedNfpm(address(params.nfpm)));
        address positionOwner = params.nfpm.ownerOf(params.tokenId);
>>      _validateOrder(params.userOrder, params.orderSignature, positionOwner);
        _execute(params, positionOwner);
    }
```
This allows the operator to pass arbitrary parameters in the `ExecuteParams` struct as long as `userOrder` is correct.