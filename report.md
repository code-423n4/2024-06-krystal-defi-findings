---
sponsor: "Krystal DeFi"
slug: "2024-06-krystal-defi"
date: "2024-07-12"
title: "Krystal DeFi Invitational"
findings: "https://github.com/code-423n4/2024-06-krystal-defi-findings/issues"
contest: 399
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Krystal DeFi smart contract system written in Solidity. The audit took place between June 21—July 1 2024.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 6 wardens contributed reports:

  1. [Dup1337](https://code4rena.com/@Dup1337) ([sorrynotsorry](https://code4rena.com/@sorrynotsorry) and [deliriusz](https://code4rena.com/@deliriusz))
  2. [d3e4](https://code4rena.com/@d3e4)
  3. [SpicyMeatball](https://code4rena.com/@SpicyMeatball)
  4. [Bauchibred](https://code4rena.com/@Bauchibred)
  5. [Ch\_301](https://code4rena.com/@Ch_301)

This audit was judged by [3docSec](https://code4rena.com/@3docsec).

Final report assembled by [itsmetechjay](https://twitter.com/itsmetechjay).

# Summary

The C4 analysis yielded an aggregated total of 5 unique vulnerabilities. Of these vulnerabilities, 0 received a risk rating in the category of HIGH severity and 5 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 5 reports detailing issues with a risk rating of LOW severity or non-critical. 

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Krystal DeFi repository](https://github.com/code-423n4/2024-06-krystal-defi), and is composed of 5 smart contracts written in the Solidity programming language and includes 1,137 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (5)
## [[M-01] Wrong logic in `AUTO_COMPOUND` doesn't allow for swap to token1 ](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/28)
*Submitted by [Dup1337](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/28)*

Protocol functionality broken

### Proof of Concept

`AUTO_COMPOUND` action allows for compounding your gains into liquidity. It additionally allows for swaps in the middle. There is a faulty condition though, which is never effective and does not allow to set `token1` as targetToken, namely `else if (state.token0 == state.token1)`:

```javascript
        } else if (params.action == Action.AUTO_COMPOUND) {
            if (params.targetToken == state.token0) {
                _swapAndIncrease(
//[...]

            // @audit it should be params.targetToken == state.token1 Currently it doesn't allow for a swap
@>          } else if (state.token0 == state.token1) {
                _swapAndIncrease(
//[...]

            } else {
                // compound without swap
                _swapAndIncrease(
//[...]
            }
```

Because pools cannot have the same token0 and token1, there is no possible position that will fulfill this condition. Looking at other parts of the codebase and params passed to `_swapAndIncrease()` in this code branch, what the protocol wants to achieve is to check if `params.targetToken == state.token1` and perform swap similarly to `params.targetToken == state.token0` branch.

### Recommended Mitigation Steps

```diff
        } else if (params.action == Action.AUTO_COMPOUND) {
            if (params.targetToken == state.token0) {
                _swapAndIncrease(SwapAndIncreaseLiquidityParams(params.protocol, params.nfpm, params.tokenId, state.amount0, state.amount1, 0, positionOwner, params.deadline, IERC20(state.token1), params.amountIn1, params.amountOut1Min, params.swapData1, 0, 0, bytes(""), params.amountAddMin0, params.amountAddMin1, 0), IERC20(state.token0), IERC20(state.token1), false);
-           } else if (state.token0 == state.token1) {
+           } else if (params.targetToken == state.token1) {
                _swapAndIncrease(SwapAndIncreaseLiquidityParams(params.protocol, params.nfpm, params.tokenId, state.amount0, state.amount1, 0, positionOwner, params.deadline, IERC20(state.token0), 0, 0, bytes(""), params.amountIn0, params.amountOut0Min, params.swapData0, params.amountAddMin0, params.amountAddMin1, 0), IERC20(state.token0), IERC20(state.token1), false);
```


**[3docSec (judge) decreased severity to Medium and commented](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/28#issuecomment-2207035503):**
 > The finding completely misses a justification for High severity. A swap not happening as it should is better categorized as Medium.

**Haupc (Krystal DeFi) confirmed**

***

## [[M-02] The signatures are replayable](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/27)
*Submitted by [Dup1337](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/27), also found by [SpicyMeatball](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/35)*

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Automation.sol#L79-L81>

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/StructHash.sol#L274-L296>

### Impact

User signed orders can be replayed

### Proof of Concept

All orders managed by the Krystal team are signed by the user, executed by the operator and the signature is verified on-chain:

```solidity
src/V3Automation.sol
    function execute(ExecuteParams calldata params) public payable onlyRole(OPERATOR_ROLE) whenNotPaused() {
        require(_isWhitelistedNfpm(address(params.nfpm)));
        address positionOwner = params.nfpm.ownerOf(params.tokenId);
@>      _validateOrder(params.userOrder, params.orderSignature, positionOwner);
        _execute(params, positionOwner);
    }
//[...]
    function _validateOrder(StructHash.Order memory order, bytes memory orderSignature, address actor) internal view {
@>      address userAddress = recover(order, orderSignature);
@>      require(userAddress == actor);
        require(!_cancelledOrder[keccak256(orderSignature)]);
    }
```

```solidity
src/EIP712.sol
    function recover(StructHash.Order memory order, bytes memory signature) internal view returns (address) {
@>      bytes32 digest = _hashTypedDataV4(StructHash._hash(order));
@>      return ECDSA.recover(digest, signature);
    }
```

However, there is no nonce field in the [StructHash.Order](https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/StructHash.sol). That means that the same hashed order can be reused multiple times. 

What's also important is that the order is used only to verify if it was signed by the user, however it's not checked if the order that user signed is the same as the params passed by the operator, because [ExecuteParams](https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Automation.sol#L40-L82) sent to `V3Automation.execute()` use other set of params to indicate the action to perform:

```solidity
    struct ExecuteParams {
        Action action;
        Protocol protocol;
        INonfungiblePositionManager nfpm;

        uint256 tokenId;
        uint128 liquidity; // liquidity the calculations are based on

        // target token for swaps (if this is address(0) no swaps are executed)
        address targetToken;
    
        uint256 amountIn0;
        // if token0 needs to be swapped to targetToken - set values
        uint256 amountOut0Min;
        bytes swapData0;
        //[...]
        // user signed config
        StructHash.Order userOrder;
        bytes orderSignature;
    }
```

And later in the function, `params.userOrder` is not used, only the params prepared by the operator:

```solidity
src/V3Automation.sol
@>      if (params.action == Action.AUTO_ADJUST) {
            require(state.tickLower != params.newTickLower || state.tickUpper != params.newTickUpper);
            SwapAndMintResult memory result;
@>          if (params.targetToken == state.token0) {
@>              result = _swapAndMint(SwapAndMintParams(params.protocol, params.nfpm, IERC20(state.token0), IERC20(state.token1), state.fee, params.newTickLower, params.newTickUpper, 0, state.amount0, state.amount1, 0, positionOwner, params.deadline, IERC20(state.token1), params.amountIn1, params.amountOut1Min, params.swapData1, 0, 0, bytes(""), params.amountAddMin0, params.amountAddMin1), false);
            } else if (params.targetToken == state.token1) {
@>              result = _swapAndMint(SwapAndMintParams(params.protocol, params.nfpm, IERC20(state.token0), IERC20(state.token1), state.fee, params.newTickLower, params.newTickUpper, 0, state.amount0, state.amount1, 0, positionOwner, params.deadline, IERC20(state.token0), 0, 0, bytes(""), params.amountIn0, params.amountOut0Min, params.swapData0, params.amountAddMin0, params.amountAddMin1), false);
            } else {
                // Rebalance without swap
@>              result = _swapAndMint(SwapAndMintParams(params.protocol, params.nfpm, IERC20(state.token0), IERC20(state.token1), state.fee, params.newTickLower, params.newTickUpper, 0, state.amount0, state.amount1, 0, positionOwner, params.deadline, IERC20(address(0)), 0, 0, bytes(""), 0, 0, bytes(""), params.amountAddMin0, params.amountAddMin1), false);
            }
```

Actually, this behavior is shown in [test/integration/V3Automation.t.sol](https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/test/integration/V3Automation.t.sol#L13-L73):

```solidity
@>  StructHash.Order emptyUserConfig; // todo: remove this when we fill user configuration

    function setUp() external {
        _setupBase();
    }

    function testAutoAdjustRange() external {
        // add liquidity to existing (empty) position (add 1 DAI / 0 USDC)
        _increaseLiquidity();
        (address userAddress, uint256 privateKey) = makeAddrAndKey("positionOwnerAddress");

        vm.startPrank(TEST_NFT_ACCOUNT);
        NPM.safeTransferFrom(TEST_NFT_ACCOUNT, userAddress, TEST_NFT);
        vm.stopPrank();

@>      bytes memory signature = _signOrder(emptyUserConfig, privateKey);

        uint256 countBefore = NPM.balanceOf(userAddress);

        (, , , , , , , uint128 liquidityBefore, , , , ) = NPM.positions(
            TEST_NFT
        );

        V3Automation.ExecuteParams memory params = V3Automation.ExecuteParams(
            V3Automation.Action.AUTO_ADJUST,
            Common.Protocol.UNI_V3,
            NPM,
            TEST_NFT,
            liquidityBefore,
            address(USDC),
            500000000000000000,
            400000,
            _get05DAIToUSDCSwapData(),
            0,
            0,
            "",
            0,
            0,
            block.timestamp,
            184467440737095520, // 0.01 * 2^64
            0,
            MIN_TICK_500,
            -MIN_TICK_500,
            true,
            0,
            0,
            emptyUserConfig,
            signature
        );

        // using approve / execute pattern
        vm.prank(userAddress);
        NPM.setApprovalForAll(address(v3automation), true);

        vm.prank(TEST_OWNER_ACCOUNT);

        v3automation.execute(params);

        // now we have 2 NFTs (1 empty)
        uint256 countAfter = NPM.balanceOf(userAddress);
        assertGt(countAfter, countBefore);

        (, , , , , , , uint128 liquidityAfter, , , , ) = NPM.positions(
            TEST_NFT
        );
        assertEq(liquidityAfter, 0);
    }
```

In the test above, `emptyUserConfig` - being really empty, i.e. consisting of only 0s, because it's not being set - is being signed and the order successfully executes, even though the action adds liquidity. That means that the `deadline`, `liquidity` to get, `recipient` of the funds, `amount` , etc. - basically all parameters that the user signs are independent from what is being sent on-chain.

To summarize, there are multiple occasions, where no nonce and actually no verification of execute params conformity to under signed order can be exploited:

*   Purposeful action
    *   private operator key compromise
*   Accidental action
    *   Backend API breach
    *   software failure leading to sending other orders than users signed
    *   sending the same order multiple times due to local database failure

### Recommended Mitigation Steps

Introduce nonce and verification that operator parameters are the same that the user signed.

**[3docSec (judge) commented](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/27#issuecomment-2208694766):**
 > Confirming as Medium. There is the concrete possibility of fund loss, however, the `onlyRole(OPERATOR_ROLE)` privilege required to exploit it mitigates the risk.

**namnm1991 (Krystal DeFi) acknowledged**

***

## [[M-03] `_deductFees()` is incompatible with tokens that revert on zero value transfers](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/21)
*Submitted by [d3e4](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/21)*

All main functionality risks reverting with tokens that revert on zero value transfers, via a transfer in `Common._deductFees()`.

### Proof of Concept

In `Common._deductFees()`

```solidity
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
```

if `0 < params.feeX64 * params.amountX < Q64` then `feeAmountX = 0`. In this case the `SafeERC20.safeTransfer()` reverts on a token which reverts on zero transfers.

`_deductFees()` is used throughout the functionality of `V3Automation` and `V3Utils`.

### Recommended Mitigation Steps

```solidity
if (feeAmount0 > 0) {
    SafeERC20.safeTransfer(IERC20(params.token0), FEE_TAKER, feeAmount0);
}
```

etc.

### Assessed type

ERC20

**Haupc (Krystal DeFi) confirmed**

***

## [[M-04] The Protocol breaks the Allowance Mechanism of the NFTs](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/17)
*Submitted by [Dup1337](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/17)*

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L392> 

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Utils.sol#L76-L85>

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Utils.sol#L171> 

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/V3Automation.sol#L92>

### Impact

The user loses their approved entities

### Proof of Concept

The protocol transfers the Position NFT from the position owner to the protocol and back to the position owner to execute the actions.

<details>
<summary>List of occurrences</summary>

```solidity
Contract: Common.sol

402:         params.nfpm.transferFrom(address(this), params.recipient, result.tokenId);
```

```solidity
Contract: V3Utils.sol

171:         nfpm.transferFrom(address(this), from, tokenId);
```

```solidity
Contract: V3Utils.sol

76:     function execute(INonfungiblePositionManager _nfpm, uint256 tokenId, Instructions calldata instructions)  whenNotPaused() external
77:     {
78:         // must be approved beforehand
79:         _nfpm.safeTransferFrom(
80:             msg.sender,
81:             address(this),
82:             tokenId,
83:             abi.encode(instructions)
84:         );
85:     }
```

```solidity
Contract: V3Automation.sol

92:         params.nfpm.transferFrom(positionOwner, address(this), params.tokenId);
```

```solidity
Contract: V3Automation.sol

167:         params.nfpm.transferFrom(address(this), positionOwner, params.tokenId);
```

</details>

This breaks the allowance mechanism of ERC721 for every action taken on the platform and it omits the allowances given from the token owners to the approved entities.

For every NFT transfer, the allowances are zeroed out:

```solidity
Contract: ERC721.sol

333:     function _transfer(address from, address to, uint256 tokenId) internal virtual {
334:         require(ERC721.ownerOf(tokenId) == from, "ERC721: transfer from incorrect owner");
335:         require(to != address(0), "ERC721: transfer to the zero address");
336: 
337:         _beforeTokenTransfer(from, to, tokenId, 1);
338: 
339:         // Check that tokenId was not transferred by `_beforeTokenTransfer` hook
340:         require(ERC721.ownerOf(tokenId) == from, "ERC721: transfer from incorrect owner");
341: 
342:         // Clear approvals from the previous owner
343:   >>    delete _tokenApprovals[tokenId];
344: 
345:         unchecked {
346:             // `_balances[from]` cannot overflow for the same reason as described in `_burn`:
347:             // `from`'s balance is the number of token held, which is at least one before the current
348:             // transfer.
349:             // `_balances[to]` could overflow in the conditions described in `_mint`. That would require
350:             // all 2**256 token ids to be minted, which in practice is impossible.
351:             _balances[from] -= 1;
352:             _balances[to] += 1;
353:         }
354:         _owners[tokenId] = to;
355: 
356:         emit Transfer(from, to, tokenId);
357: 
358:         _afterTokenTransfer(from, to, tokenId, 1);
359:     }
```

E.g.:

1.  Alice is on her flight to attend DSS Bangkok
2.  She already approved Bob to take care of her positions until she arrives during the long flight
3.  Bob interacts with Krystal to collect the LP fees.
4.  Bob loses his approval
5.  Alice´s position remains unattended and one of the pool tokens takes a nose dive before Bob can withdraw the liquidity and sell it.

### Recommended Mitigation Steps

NFPM uses `isAuthorizedForToken` modifier as below:

```solidity
Contract: NonfungiblePositionManager.sol

183:     modifier isAuthorizedForToken(uint256 tokenId) {
184:         require(_isApprovedOrOwner(msg.sender, tokenId), 'Not approved');
185:         _;
186:     }
```

Take approval of token owners rather than using `onERC721Received` hook.

**[Haupc (Krystal DeFi) acknowledged and commented](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/17#issuecomment-2205369614):**
 > There are 2 ways to take approval from user
> 1. via `approve` function => this function also discards previous approval. So we can not keep other's approval.
> 2. via `setApprovalForAll` function => this function seems too risky for user to use


**[quanghuy219 (Krystal DeFi) commented](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/17#issuecomment-2210002589):**
 > ERC721 implementation allows only one spender on a token at a time, therefore taking user's approval for our contract will also clear other approval on the same token
> ```solidity
> mapping(uint256 tokenId => address) private _tokenApprovals;
> ```
> Another approach is to use `setApprovalForAll` function in ERC721, which poses another concern of allowing our contract to use all user's positions.
> 
> 
> With that in mind, we think that using `onERC721Received` hook is still the safest option for our users and Krystal is able to provide convenient user experience.
> 

***

## [[M-05] Swapping logic would be broken for some supported tokens](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/10)
*Submitted by [Bauchibred](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/10), also found by [SpicyMeatball](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/34) and [d3e4](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/20)*

First take a look at this excerpt from the README: 

<https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/README.md#L103-L106>

```markdown
| Question                                                                                                                                                   | Answer                                      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| Chains the protocol will be deployed on                                                                                                                    | Ethereum,Arbitrum,Base,BSC,Optimism,Polygon |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------                                      |
| [Revert on zero value approvals](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-approvals)                                    | Yes                                         |
```

From the above we can conclude that tokens like BNB that revert on zero value approvals are to be integrated within protocol.

Now take a look at <https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L537-L567>

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
            //@audit resetting the approval would never work for these tokens
            _safeApprove(tokenIn, swapRouter, 0);

            uint256 balanceInAfter = tokenIn.balanceOf(address(this));
            uint256 balanceOutAfter = tokenOut.balanceOf(address(this));

            amountInDelta = balanceInBefore - balanceInAfter;
            amountOutDelta = balanceOutAfter - balanceOutBefore;

            // amountMin slippage check
            if (amountOutDelta < amountOutMin) {
                revert SlippageError();
            }

            // event for any swap with exact swapped value
            emit Swap(address(tokenIn), address(tokenOut), amountInDelta, amountOutDelta);
        }
    }
```

This is the general swap function that eventually gets called which uses the external router with off-chain calculated swap instruction. The issue, however, is that after approving the initial amount needed to the router, there is a need to reset these approvals, however resetting these approvals whereas would work for most tokens would not work for a token like BNB that's to be supported, [see minimalistic coded POC here](https://github.com/d-xo/weird-erc20/blob/main/src/ApprovalWithZeroValue.sol), which then translates to a direct break in swapping for these tokens and other instances where `_swap()` gets called across protocol.

Additionally, it seems the `_safeResetAndApprove()` function was coded to help sort this issue, but it is not being used in `_swap()`, <https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L713-L724>

```solidity
    /// @dev some tokens require allowance == 0 to approve new amount
    /// but some tokens does not allow approve ammount = 0
    /// we try to set allowance = 0 before approve new amount. if it revert means that
    /// the token not allow to approve 0, which means the following line code will work properly
    function _safeResetAndApprove(IERC20 token, address _spender, uint256 _value) internal {
        /// @dev ommited approve(0) result because it might fail and does not break the flow
        address(token).call(abi.encodeWithSelector(token.approve.selector, _spender, 0));

        /// @dev value for approval after reset must greater than 0
        require(_value > 0);
        _safeApprove(token, _spender, _value);
    }
```

### Impact

As hinted above, swaps would be completely broken for tokens that revert on zero value approvals, due to a reversion that always occurs on this [line](https://github.com/code-423n4/2024-06-krystal-defi/blob/f65b381b258290653fa638019a5a134c4ef90ba8/src/Common.sol#L551-L552).

### Recommended Mitigation Steps

Consider try/catching the attempt to safeApprove in `_swap()` and in the case it reverts, query `_safeResetAndApprove()` instead. Alternatively, do not support tokens that revert on zero value approvals.

### Assessed type

Context

**Haupc (Krystal DeFi) confirmed**

***

# Low Risk and Non-Critical Issues

For this audit, 5 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/2) by **SpicyMeatball** received the top score from the judge.

*The following wardens also submitted reports: [Dup1337](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/29), [Ch\_301](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/26), [Bauchibred](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/7), and [d3e4](https://github.com/code-423n4/2024-06-krystal-defi-findings/issues/30).*

## [01] Withdrawer granted DEFAULT_ADMIN_ROLE

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

## [02] `execute` function is defined as payable

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Automation.sol

```solidity
    function execute(ExecuteParams calldata params) public payable onlyRole(OPERATOR_ROLE) whenNotPaused() {
```

The `execute` function includes the `payable` keyword but does not utilize ETH tokens within its logic.

## [03] `amountIn` validation should be after fees were deducted

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

## [04] No incentive for users to pay protocol fees in the `V3Utils.sol`

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

## [05] Contracts without receive/fallback function may not be able to receive leftovers from `swapAndMint`, `swapAndIncreaseLiquidity` functions

If a non-zero amount of ETH is sent during `swapAndMint` or `swapAndIncreaseLiquidity` calls, the protocol will unwrap leftover WETH tokens and send them to the recipient.

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L230

https://github.com/code-423n4/2024-06-krystal-defi/blob/main/src/V3Utils.sol#L258

```solidity
result = _swapAndIncrease(_params, IERC20(token0), IERC20(token1), msg.value != 0);
```

The last argument is a bool value, that will unwrap WETH if `true`. Consider allowing user to specify whether he wants to wrap or unwrap his leftover tokens.

## [06] Orders and params are separate

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

***





# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
