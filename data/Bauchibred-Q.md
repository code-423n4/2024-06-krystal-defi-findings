# QA Report for **Krystal Defi**
## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-some-addresses-wouldnt-be-able-to-have-the-native-token-balance-withdrawn-to-them) | Some addresses wouldn't be able to have the native token balance withdrawn to them |
| [QA-02](#qa-02-the-if-(params.token0-==-params.token1)-can-be-sidestepped-since-tokens-with-multiple-addresses-are-supported) | The `if (params.token0 == params.token1)` can be sidestepped since tokens with multiple addresses are supported |
| [QA-02](#qa-02-the-if-(params.token0-==-params.token1)-can-be-sidestepped-since-tokens-with-multiple-addresses-are-supported) | The `if (params.token0 == params.token1)` can be sidestepped since tokens with multiple addresses are supported |
| [QA-04](#qa-04-functionalities-in-krystal-defi-could-be-dosd-when-some-tokens-are-integrated) | Functionalities in Krystal Defi could be DOS'd when some tokens are integrated |
| [QA-05](#qa-05-redundant-check-in-common#_deductfees()) | Redundant check in `Common#_deductFees()` |
| [QA-06](#qa-06-make-v3utils#execute()-more-effective) | Make `V3Utils#execute()` more effective |
| [QA-07](#qa-07-comments-should-lead-to-valid-links) | Comments should lead to valid links |
| [QA-08](#qa-08-common.sol-might-never-get-initialised) | Common.sol might never get initialised |
| [QA-09](#qa-09-open-todos-should-be-sorted-before-live-deployment) | Open todos should be sorted before live deployment |
## QA-01 Some addresses wouldn't be able to have the native token balance withdrawn to them

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L274-L279

```solidity
    function withdrawNative(address to) external onlyRole(WITHDRAWER_ROLE) {
        uint256 nativeBalance = address(this).balance;
        if (nativeBalance > 0) {
            payable(to).transfer(nativeBalance);//@audit using payable.transfer to send eth?
        }
    }
```

As hinted by the @audit tag we can see that there is an implementation of using `payable.transfer()` to send ETH when claiming the withdrawals however this is heavily frowned upon cause it can lead to the having this attempt revert since the `transfer() `call requires that the recipient is either an EOA account, or is a contract that has a payable callback. For the contract case, the `transfer()` call only provides 2300 gas for the contract to complete its operations. This means the following cases can cause the transfer to fail:

- The contract does not have a payable callback
- The contract’s payable callback spends more than 2300 gas (which is only enough to emit something)
- The contract is called through a proxy which itself uses up the 2300 gas Use OpenZeppelin’s Address.sendValue() instead

  [Reference](https://code4rena.com/reports/2022-07-golom#l02--dont-use-payabletransferpayablesend)

### Impact

DOS to withdraw native token in some cases as the withdrawals can no longer be finalized.

### Recommended Mitigation Steps

Consider not using the `payable.transfer()` and instead send the ether via the `.call()` method.

## QA-02 The `if (params.token0 == params.token1)` can be sidestepped since tokens with multiple addresses are supported

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Utils.sol#L191-L232

```solidity
    function swapAndMint(SwapAndMintParams calldata params)  whenNotPaused() external payable returns (SwapAndMintResult memory result) {
        if (params.token0 == params.token1) {
            revert SameToken();
        }
        require(_isWhitelistedNfpm(address(params.nfpm)));
        IWETH9 weth = _getWeth9(params.nfpm, params.protocol);

        // validate if amount2 is enough for action
        if (params.swapSourceToken != params.token0
            && params.swapSourceToken != params.token1
            && params.amountIn0 + params.amountIn1 > params.amount2
        ) {
            revert AmountError();
        }
        _prepareSwap(weth, params.token0, params.token1, params.swapSourceToken, params.amount0, params.amount1, params.amount2);
        SwapAndMintParams memory _params = params;

        DeductFeesEventData memory eventData;
        if (params.protocolFeeX64 > 0) {
            uint256 feeAmount0;
            uint256 feeAmount1;
            uint256 feeAmount2;
            // since we do not have the tokenId here, we need to emit event later
            (_params.amount0, _params.amount1, _params.amount2, feeAmount0, feeAmount1, feeAmount2) = _deductFees(DeductFeesParams(params.amount0, params.amount1, params.amount2, params.protocolFeeX64, FeeType.PROTOCOL_FEE, address(params.nfpm), 0, params.recipient, address(params.token0), address(params.token1), address(params.swapSourceToken)), false);

            eventData = DeductFeesEventData({
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
        emit DeductFees(address(params.nfpm), result.tokenId, params.recipient, eventData);
    }
```

Now see this excerpt from the readMe https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/README.md#L120-L121

```markdown
| Question                                                                                                          | Answer |
| ----------------------------------------------------------------------------------------------------------------- | ------ |
| [Multiple token addresses](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-transfers) | Yes    |
```

This would then mean that the check below can easily be sidestepped even if the same token is passed, considering it has more than one address.

```markdown
        if (params.token0 == params.token1) {
            revert SameToken();
        }
```

### Impact

QA

### Recommended Mitigation Steps

Do not support these tokens.

## QA-03 Setters don't have equality checkers

### Proof of Concept

Multiple instances in scope, for example take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L704-L708

```solidity

    function setMaxFeeX64(FeeType feeType, uint64 feex64) external onlyRole(ADMIN_ROLE) {
        _maxFeeX64[feeType] = feex64;
    }

```

We can see that there is no ` _maxFeeX64[feeType] != feex64` check before setting the new value.

### Impact

Unnecessary code execution in the case where ` _maxFeeX64[feeType] == feex64`

### Recommended Mitigation Steps

Consider including equality checkers in the setter functions.

## QA-04 Functionalities in Krystal Defi could be DOS'd when some tokens are integrated

### Proof of Concept

First, take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L260-L269

```solidity
    function withdrawERC20(IERC20[] calldata tokens, address to) external onlyRole(WITHDRAWER_ROLE) {
        uint count = tokens.length;
        for(uint i = 0; i < count; ++i) {
            uint256 balance = tokens[i].balanceOf(address(this));
            if (balance > 0) {
                SafeERC20.safeTransfer(tokens[i], to, balance);
            }
        }
    }

```

This function is used to withdraw ERC20s and one of the instances where the `.balanceOf()` functionality is being used, issue however is that this query would always revert for some tokens, see [source](https://github.com/sherlock-audit/2023-06-tokemak-judging/issues/585).

### Impact

QA, considering the very low likelihood.

### Recommended Mitigation Steps

Do not support these tokens or query `.balanceOf()` on a low-level.

## QA-05 Redundant check in `Common#_deductFees()`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L657-L695

```solidity
    function _deductFees(DeductFeesParams memory params, bool emitEvent) internal returns(uint256 amount0Left, uint256 amount1Left, uint256 amount2Left, uint256 feeAmount0, uint256 feeAmount1, uint256 feeAmount2) {
        uint256 Q64 = 2 ** 64;
        if (params.feeX64 > _maxFeeX64[params.feeType]) {
            revert TooMuchFee();
        }

        // to save gas, we always need to check if fee exists before deductFees
        if (params.feeX64 == 0) {
            revert NoFees();
        }

        if (params.amount0 > 0) {
            feeAmount0 = FullMath.mulDiv(params.amount0, params.feeX64, Q64);
            amount0Left = params.amount0 - feeAmount0;
            SafeERC20.safeTransfer(IERC20(params.token0), FEE_TAKER, feeAmount0);
        }
        if (params.amount1 > 0) {
            feeAmount1 = FullMath.mulDiv(params.amount1, params.feeX64, Q64);
            amount1Left = params.amount1 - feeAmount1;
            SafeERC20.safeTransfer(IERC20(params.token1), FEE_TAKER, feeAmount1);
        }
        if (params.amount2 > 0) {
            feeAmount2 = FullMath.mulDiv(params.amount2, params.feeX64, Q64);
            amount2Left = params.amount2 - feeAmount2;
            SafeERC20.safeTransfer(IERC20(params.token2), FEE_TAKER, feeAmount2);
        }


        if (emitEvent) {
            emit DeductFees(address(params.nfpm), params.tokenId, params.userAddress, DeductFeesEventData(
                params.token0, params.token1, params.token2,
                params.amount0, params.amount1, params.amount2,
                feeAmount0, feeAmount1, feeAmount2,
                params.feeX64,
                params.feeType
            ));
        }

    }
```

Evidently there is a `if (params.feeX64 == 0) { revert NoFees(); }` check, issue however is that this check is redundant, considering all instances where deductFees would be queried already ensures that the fee value is not `0`, here are all the instances where there is an attempt to deduct fees,

```solidity


src/V3Automation.sol:
  105              if (params.gasFeeX64 > 0) {
  106:                 (,,, gasFeeAmount0, gasFeeAmount1,) = _deductFees(DeductFeesParams(state.amount0, state.amount1, 0, params.gasFeeX64, FeeType.GAS_FEE, address(params.nfpm), params.tokenId, positionOwner, state.token0, state.token1, address(0)), true);
  107              }

  110              if (params.protocolFeeX64 > 0) {
  111:                 (,,, protocolFeeAmount0, protocolFeeAmount1,) = _deductFees(DeductFeesParams(state.amount0, state.amount1, 0, params.protocolFeeX64, FeeType.PROTOCOL_FEE, address(params.nfpm), params.tokenId, positionOwner, state.token0, state.token1, address(0)), true);
  112              }

src/V3Utils.sol:
  105          if (instructions.protocolFeeX64 > 0) {
  106:             (amount0, amount1,,,,) = _deductFees(DeductFeesParams(amount0, amount1, 0, instructions.protocolFeeX64, FeeType.PROTOCOL_FEE, address(nfpm), tokenId, instructions.recipient, token0, token1, address(0)), true);
  107          }

  213              // since we do not have the tokenId here, we need to emit event later
  214:             (_params.amount0, _params.amount1, _params.amount2, feeAmount0, feeAmount1, feeAmount2) = _deductFees(DeductFeesParams(params.amount0, params.amount1, params.amount2, params.protocolFeeX64, FeeType.PROTOCOL_FEE, address(params.nfpm), 0, params.recipient, address(params.token0), address(params.token1), address(params.swapSourceToken)), false);
  215

  254          if (params.protocolFeeX64 > 0) {
  255:             (_params.amount0, _params.amount1, _params.amount2,,,) = _deductFees(DeductFeesParams(params.amount0, params.amount1, params.amount2, params.protocolFeeX64, FeeType.PROTOCOL_FEE, address(params.nfpm), params.tokenId, params.recipient, token0, token1, address(params.swapSourceToken)), true);
  256          }

```

Evidently `deductFees()` only gets called if the fee != 0 making the check in the internal` _deductFees()` redundant

### Impact

Redundant code execution, unnecessary gas consumption.

### Recommended Mitigation Steps

Consider removing the check in the internal `_deductFees()`.

## QA-06 Make `V3Utils#execute()` more effective

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Utils.sol#L76-L85

```solidity
    function execute(INonfungiblePositionManager _nfpm, uint256 tokenId, Instructions calldata instructions)  whenNotPaused() external
    {
        // must be approved beforehand
        _nfpm.safeTransferFrom(
            msg.sender,
            address(this),
            tokenId,
            abi.encode(instructions)
        );
    }
```

This function is used to execute any instruction by pulling in the approved NFT instead of direct safeTransferFrom call from owner, issue however is that this is not done in the most effective way, considering if any user wants to use this functionality, they must divide the tx into two, i.e first approve snd then query this.

This logic can be made more effective by just requesting the approval in the function logic itself.

### Impact

Code implementation is not the most effective

### Recommended Mitigation Steps

Consider integrating an approval method in the function itself.

## QA-07 Comments should lead to valid links

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/EIP712.sol#L34-L52

```solidity
    /**
     * @dev Returns the keccak256 digest of an EIP-712 typed data (EIP-191 version `0x01`).
     *
     * The digest is calculated from a `domainSeparator` and a `structHash`, by prefixing them with
     * `\x19\x01` and hashing the result. It corresponds to the hash signed by the
     * https://eips.ethereum.org/EIPS/eip-712[`eth_signTypedData`] JSON-RPC method as part of EIP-712.//@audit invalid link format
     *
     * See {ECDSA-recover}.
     */
    function toTypedDataHash(bytes32 domainSeparator, bytes32 structHash) internal pure returns (bytes32 digest) {
        /// @solidity memory-safe-assembly
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, hex"19_01")
            mstore(add(ptr, 0x02), domainSeparator)
            mstore(add(ptr, 0x22), structHash)
            digest := keccak256(ptr, 0x42)
        }
    }
```

Command clicking on vsCode the link attached here " \* https://eips.ethereum.org/EIPS/eip-712[`eth_signTypedData`] JSON-RPC method as part of EIP-712." however we can see that there is a 404 error page, because no spacing after the end of the link and the next character i.e : ![](https://cdn.discordapp.com/attachments/1235966308542185483/1254244085892644985/Screenshot_2024-06-23_at_05.18.03.png?ex=6678c954&is=667777d4&hm=9430adfde885bd47dccc7cc38dbf44b7020a4cd549a38934f738349a99cf2dd3&)

### Impact

Confusing code/documentation.

### Recommended Mitigation Steps

Consider attaching a space between the link and the next character, i.e :

```diff
    /**
     * @dev Returns the keccak256 digest of an EIP-712 typed data (EIP-191 version `0x01`).
     *
     * The digest is calculated from a `domainSeparator` and a `structHash`, by prefixing them with
     * `\x19\x01` and hashing the result. It corresponds to the hash signed by the
-     * https://eips.ethereum.org/EIPS/eip-712[`eth_signTypedData`] JSON-RPC method as part of EIP-712.//@audit invalid link format
+     * https://eips.ethereum.org/EIPS/eip-712 [`eth_signTypedData`] JSON-RPC method as part of EIP-712.//@audit invalid link format
     *
     * See {ECDSA-recover}.
     */
    function toTypedDataHash(bytes32 domainSeparator, bytes32 structHash) internal pure returns (bytes32 digest) {
        /// @solidity memory-safe-assembly
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, hex"19_01")
            mstore(add(ptr, 0x02), domainSeparator)
            mstore(add(ptr, 0x22), structHash)
            digest := keccak256(ptr, 0x42)
        }
    }
```

## QA-08 Common.sol might never get initialised

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L102-L128

```solidity
    constructor() {
        _maxFeeX64[FeeType.GAS_FEE] = 1844674407370955264; // 10%
        _maxFeeX64[FeeType.PROTOCOL_FEE] = 1844674407370955264; // 10%
        _initializer = tx.origin;
    }

    bool private _initialized = false;
    function initialize(address router, address admin, address withdrawer, address feeTaker, address[] calldata whitelistedNfpms) public virtual {
        require(!_initialized);
        if (withdrawer == address(0)) {
            revert();
        }
        require(msg.sender == _initializer);

        _grantRole(ADMIN_ROLE, admin);
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(WITHDRAWER_ROLE, withdrawer);
        _grantRole(DEFAULT_ADMIN_ROLE, withdrawer);
        swapRouter = router;
        FEE_TAKER = feeTaker;
        for (uint256 i = 0; i < whitelistedNfpms.length; i++) {
            EnumerableSet.add(_whitelistedNfpm, whitelistedNfpms[i]);
        }

        _initialized = true;
    }

```

This is the implementation of both the constructor and extensively the initializer, problem here however is the fsct thst in the constructor the initializer is being set to `tx.origin` however in the initialiser there is a check that the caller (msg.sender) must be the initializer address, now considering not all cases/tx have `msg.sender == tx.origin` this could then leave the contract in an unwanted state.

### Impact

QA, imo as this can be considered admin error.

### Recommended Mitigation Steps

Consider checking against one address instead

## QA-08 Common.sol might never get initialised
## QA-09 Open todos should be sorted before live deployment

### Proof of Concept

Multiple instances in scope, for example, take a look at https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L135-L139

```solidity
    enum FeeType {
        GAS_FEE,
        PROTOCOL_FEE
        // todo: PERFORMANCE_FEE @audit open toDos
    }
```

Evidently this is a `todo` that's left in production code

As hinted this is just one of the multiple instances in scope, use this search command to pinpoint to other instances: [https://github.com/search?q=repo%3Acode-423n4%2F2024-06-krystal-defi%20todo&type=code](https://github.com/search?q=repo%3Acode-423n4%2F2024-06-krystal-defi%20todo&type=code)

### Impact

QA... Un-ready code is being deployed.

### Recommended Mitigation Steps

Consider sorting out all _todos_ before final deployment.

