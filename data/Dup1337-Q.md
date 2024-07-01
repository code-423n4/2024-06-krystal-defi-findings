|        | Issues                                                                     |
|:------ |:-------------------------------------------------------------------------- |
| [L-01] | No dApp name included in EIP712 contract                                   |
| [L-02] | `swapAndMint` event is not properly emitted if the protocol fee is not set |
| [L-03] | Invalid owner comment or wrong execution                                   |
| [L-04] | Unprocessable orders due to fee change                                     |
| [L-05] | No possibility to auto exit 2 tokens                                       |
| [L-06] | The protocol allows swapAndMint even for the same position                 |
| [L-07] | `_collectFees` function is against AMM implementation                      |
| [L-08] | Incorrect `_maxFeeX64`                                                     |
| [L-09] | SwapRouter fees can be locked if taken                                     |
| [L-10] | Incorrect mint parameters for AlgebraV1 in Swap function                   |
| [L-11] | Equation can be shortened                                                  |
| [L-12] | Swap and mint ETH unwrapping might not be desireable by the user |


## [L-01] No dApp name included in EIP712 contract 
EIP712 contract does not enforce a `name` and `version` state variable.
This leads to not being able to read the dapp name when signing at Metamask/Wallet Extension.
```solidity
Contract: EIP712.sol

13:     constructor(string memory name, string memory version) { /
14:         DOMAIN_SEPARATOR = keccak256(
15:             abi.encode(
16:                 TYPE_HASH, 
17:                 keccak256(bytes(name)), //@audit not implementing a name variable leads the users not being able to see to whom they are signing
18:                 keccak256(bytes(version)), 
19:                 block.chainid, 
20:                 address(this)
21:             )
22:         );
23:     }
```
## [L-02] `swapAndMint` event is not properly emitted if the protocol fee is not set     
In `src/V3Utils.sol#swapAndMint()`, there is an event structure created and then emitted. The problem is that it's only filled in when fees are passed. Because it's permissionless function, a user may pass 0 fees and the event will be emited improperly:
```solidity
        if (params.protocolFeeX64 > 0) {
            //[...]

@>          eventData = DeductFeesEventData({
                token0: address(params.token0),
                token1: address(params.token1),
                token2: address(params.swapSourceToken),
                amount0: params.amount0,
                amount1: params.amount1,
                amount2: params.amount2,
                feeAmount0: feeAmount0,
                feeAmount1: feeAmount1,
                feeAmount2: feeAmount2,
                feeX64: params.protocolFeeX64,
                feeType: FeeType.PROTOCOL_FEE
            });
        }

        result = _swapAndMint(_params, msg.value != 0);

@>      emit DeductFees(address(params.nfpm), result.tokenId, params.recipient, eventData);
```
## [L-03] Invalid owner comment or wrong execution 
According to the natspec:

>    /// @notice Does 1 or 2 swaps from swapSourceToken to token0 and token1 and adds as much as possible liquidity to any existing position (no need to be position owner).


however, few lines below there is this check:
```solidity
        address owner = params.nfpm.ownerOf(params.tokenId);
        require(owner == msg.sender, "sender is not owner of position");

```
So the caller actually have to be the position owner, in contrary to the natspec.

## [L-04] Unprocessable orders due to fee change
When the operator applies max fees (more than user expected), and user `minAmount` will not be possible to meet due to fees change.
OR
`minAmount` can be even be met but the excess tokens could be paid as fees due to fee increase which the user might not desire.
```solidity
Contract: Common.sol

715:     function setMaxFeeX64(FeeType feeType, uint64 feex64) external onlyRole(ADMIN_ROLE) {
716:         _maxFeeX64[feeType] = feex64;
717:     }
```
We recommend applying `whenPaused` modifier for the setter.

## [L-05] No possibility to auto exit 2 tokens 
In `V3Automation` in `auto_exit`, there are swaps to target token. In case of user error - mostly -, swaps can be skipped, and the contract will end up with both user tokens locked in the contract. 
Additionally, if `targetToken` is 0 and `swapData` length is 0, user will not get any tokens :
```solidity
Contract: V3Automation.sol

151:             // send complete target amount
152:             if (targetAmount != 0 && params.targetToken != address(0)) {
153:                 _transferToken(weth, positionOwner, IERC20(params.targetToken), targetAmount, false);
154:             }
```

## [L-06] The protocol allows swapAndMint even for the same position
If the user opts for `AUTO_ADJUST` , it is avoided to have the same ticks for re-minting:
```solidity
Contract: V3Automation.sol

117:         if (params.action == Action.AUTO_ADJUST) {
118:             require(state.tickLower != params.newTickLower || state.tickUpper != params.newTickUpper);
119:             SwapAndMintResult memory result;
120:             if (params.targetToken == state.token0) {
```

But it´s not carried out in `V3Utils`.`swapAndMint`
Accordingly, the user can have several same nfts for same ticks with different tokenId.

It makes cumbersome of the whole process and also not good in terms of fees or gas.

```solidity
Contract: Common.sol

355:     // swap and mint logic
356:     function _swapAndMint(SwapAndMintParams memory params, bool unwrap) internal returns (SwapAndMintResult memory result) { //@audit if the user has already a position for the same nfpm-ticks, then their position will not be added on top if they call this, instead, another nft will be created. the platform needs to prevent this.
357:         (uint256 total0, uint256 total1) = _swapAndPrepareAmounts(params, unwrap);
```


## [L-07] `_collectFees` function is against AMM implementation
`_collectFees` collect fees from the position:
```solidity
Contract: Common.sol

585:     function _collectFees(INonfungiblePositionManager nfpm, uint256 tokenId, IERC20 token0, IERC20 token1, uint128 collectAmount0, uint128 collectAmount1) internal returns (uint256 amount0, uint256 amount1) {
586:         uint256 balanceBefore0 = token0.balanceOf(address(this));
587:         uint256 balanceBefore1 = token1.balanceOf(address(this));
588:         (amount0, amount1) = nfpm.collect(
589:             univ3.INonfungiblePositionManager.CollectParams(tokenId, address(this), collectAmount0, collectAmount1)
590:         );
591:         uint256 balanceAfter0 = token0.balanceOf(address(this));
592:         uint256 balanceAfter1 = token1.balanceOf(address(this));
593: 
594:         // reverts for fee-on-transfer tokens
595:  >      if (balanceAfter0 - balanceBefore0 != amount0) {
596:             revert CollectError();
597:         }
598:  >      if (balanceAfter1 - balanceBefore1 != amount1) {
599:             revert CollectError();
600:         }
601:     }
```
We´re aware that FOT tokens are OOS.
But, if the token becomes a FOT token by implementing their fees, this will revert
This is against Uniswap logic as Uni allows these kind of tokens anyways.

Since the nfpm is whitelisted, it can´t be the issue untill any of these pool tokens start implementing fees like `usdt` or `usdc` 


## [L-08] Incorrect `_maxFeeX64`   

##### Impact
Wrong maths, wrong fees
##### Proof of Concept
As per the Protocol the Max Fee for Gas and Protocol is set to 10%:
```solidity
Contract: Common.sol

102:     constructor() {
103:         _maxFeeX64[FeeType.GAS_FEE] = 1844674407370955264; // 10% 
104:         _maxFeeX64[FeeType.PROTOCOL_FEE] = 1844674407370955264; // 10%
```
with regards to:
```solidity
Contract: Common.sol

101:     mapping (FeeType => uint64) private _maxFeeX64;
```

However, 10% of `uint64` max value results as: `1844674407370955161`

Difference: [`102`](https://www.wolframalpha.com/input?i=%28%28%282**64%29-1%29%2F10%29+-+1844674407370955264) 
##### Tools Used
Manual Review, Wolfram Alpha
##### Recommended Mitigation Steps
Recommend setting it to: `1844674407370955161`

## [L-09] SwapRouter fees can be locked if taken
Any action that user desires to perform via Krystal, allows optioonally for swapping tokens, which is performed via low level call to `swapRouter` in `Common._swap()`:
```javascript
    function _swap(IERC20 tokenIn, IERC20 tokenOut, uint256 amountIn, uint256 amountOutMin, bytes memory swapData) internal returns (uint256 amountInDelta, uint256 amountOutDelta) {
        //[...]
            // execute swap
@>          (bool success,) = swapRouter.call(swapData);
            if (!success) {
                revert ("swap failed!");
            }
```

We confirmed with the sponsor, that `SwapRouter` is another Krystal contract. The contract implements multiple functions to ease swapping, and it also allows taking fees in case that `params.feeMode` is set. The fees are accounted for `platformWallet`, which is `msg.sender`:

```javascript
    function swapInternal(
        address payable swapContract,
        uint256 srcAmount,
        uint256 minDestAmount,
        address[] calldata tradePath,
        address payable recipient,
        FeeMode feeMode,
        uint256 platformFee,
        address payable platformWallet,
        bytes calldata extraArgs
    ) internal returns (uint256 destAmount) {
//[...]
        uint256 actualSrcAmount = safeTransferWithFee(
            msg.sender,
            swapContract,
            tradePath[0],
            srcAmount,
@>          feeMode == FeeMode.FROM_SOURCE ? platformFee : 0,
            platformWallet
        );
//[...]
    function safeTransferWithFee(
        address payable from,
        address payable to,
        address token,
        uint256 amount,
        uint256 platformFeeBps,
        address payable platformWallet
    ) internal returns (uint256 amountTransferred) {
//[...]
@>      addFeeToPlatform(platformWallet, tokenErc, fee);
    }

    function addFeeToPlatform(
        address payable platformWallet,
        IERC20Ext token,
        uint256 amount
    ) internal {
        if (amount > 0) {
            require(supportedPlatformWallets.contains(platformWallet), "unsupported platform");
@>          platformWalletFees[platformWallet][token] = platformWalletFees[platformWallet][token] // @audit platformWallet is msg.sender - V3Automation or V3Utils
            .add(amount);
        }
    }
```

Later, the `platformWallet` can claim accrued fees:
```javascript
    /// Claim fee, must be done by platform address itself to avoid confusion
    function claimPlatformFee(IERC20Ext[] calldata tokens)
        external
        override
        nonReentrant
    {
        address platformWallet = msg.sender;
        for (uint256 j = 0; j < tokens.length; j++) {
            uint256 fee = platformWalletFees[platformWallet][tokens[j]];
            if (fee > 1) {
                // fee set to 1 to avoid the SSTORE initial gas cost
                platformWalletFees[platformWallet][tokens[j]] = 1;
                transferToken(payable(platformWallet), tokens[j], fee - 1);
            }
        }
        address[] memory arr = new address[](1);
        arr[0] = platformWallet;
        emit ClaimedPlatformFees(arr, tokens, msg.sender);
    }
```

Because calldata passed is raw, and the protocol doesn't verify it -  `swapRouter.call(swapData)` - users can pass their own calldata, calling `claimPlatformFee()` and transferring the fees to the  `V3Utils`. The contract is not prepared to hold any tokens, so it will be locked in the contract.

## [L-10] Incorrect mint parameters for AlgebraV1Swap function 
`_swapAndMint` swaps and mints either on UniV3 or on Algebra:
```solidity
Contract: Common.sol

356:     function _swapAndMint(SwapAndMintParams memory params, bool unwrap) internal returns (SwapAndMintResult memory result) { 
357:         (uint256 total0, uint256 total1) = _swapAndPrepareAmounts(params, unwrap);
358:         
359:         if (params.protocol == Protocol.UNI_V3) {
360:             // mint is done to address(this) because it is not a safemint and safeTransferFrom needs to be done manually afterwards
361:             (result.tokenId,result.liquidity,result.added0,result.added1) = _mintUniv3(params.nfpm, univ3.INonfungiblePositionManager.MintParams(
362:                 address(params.token0),
363:                 address(params.token1),
364:                 params.fee,
365:                 params.tickLower,
366:                 params.tickUpper,
367:                 total0, 
368:                 total1,
369:                 params.amountAddMin0,
370:                 params.amountAddMin1,
371:                 address(this), // is sent to real recipient aftwards
372:                 params.deadline
373:             ));
374:         } else if (params.protocol == Protocol.ALGEBRA_V1) {
375:             // mint is done to address(this) because it is not a safemint and safeTransferFrom needs to be done manually afterwards
376:    >>       (result.tokenId,result.liquidity,result.added0,result.added1) = _mintAlgebraV1(params.nfpm, univ3.INonfungiblePositionManager.MintParams( //@audit wrong struct - need to use algebraV1 mint params
377:                 address(params.token0),
378:                 address(params.token1),
379:                 params.fee,
380:                 params.tickLower,
381:                 params.tickUpper,
382:                 total0, 
383:                 total1,
384:                 params.amountAddMin0,
385:                 params.amountAddMin1,
386:                 address(this), // is sent to real recipient aftwards
387:                 params.deadline
388:             ));
```
As seen at L:376 the mint params for Algebra is `Univ3.mintParams`
Better to use below to avoid future troubles:
```solidity
Contract: Common.sol

14:     struct AlgebraV1MintParams { 
15:         address token0;
16:         address token1;
17:         int24 tickLower;
18:         int24 tickUpper;
19:         uint256 amount0Desired;
20:         uint256 amount1Desired;
21:         uint256 amount0Min;
22:         uint256 amount1Min;
23:         address recipient;
24:         uint256 deadline;
25:     }
``` 
## [L-11] Equation can be shortened  
The snippet below can be shortened due to there is only one condition fulfilling L:97:
```solidity
Contract: V3Automation.sol

91:     function _execute(ExecuteParams calldata params, address positionOwner) internal {
92:         params.nfpm.transferFrom(positionOwner, address(this), params.tokenId);
93: 
94:         ExecuteState memory state;
95:         (state.token0, state.token1, state.liquidity, state.tickLower, state.tickUpper, state.fee) = _getPosition(params.nfpm, params.protocol, params.tokenId);
96: 
97:  >>       require(state.liquidity != params.liquidity || params.liquidity != 0);
```

The result will be false only when the  `state.liquidity` and `params.liquidity` is 0

## [L-12] Swap and mint ETH unwrapping might not be desireable by the user
`V3Utils.swapAndMint()` is payable, and defines if `_swapAndMint()` should unwrap the amounts by checking if `msg.value != 0`:
```javascript
    function swapAndMint(SwapAndMintParams calldata params)  whenNotPaused() external payable returns (SwapAndMintResult memory result) {
//[...]
        result = _swapAndMint(_params, msg.value != 0); // @audit deciding whether to unwrap based on msg.value sent
//[...]
    function _swapAndMint(SwapAndMintParams memory params, bool unwrap) internal returns (SwapAndMintResult memory result) {
//[...]
```

However, user might not want to receive ETH back in case that they provide input `msg.value`. This can lead to unexpected scenarios.