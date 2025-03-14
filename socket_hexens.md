# YieldToken share rate can be manipulated using hook limits

## Description
Path: `contracts/hooks/Controller_YieldLimitExecHook.sol#L116-L130`

In the function `Controller::receiveInbound()`, the contract is required to mint an amount of shares corresponding to the `lockAmount` of assets. However, the amount of assets that could be used to mint may be less than `lockAmount` due to the execution limit `_receivingLimitParams` of the connector.

In function `Controller_YieldLimitExecHook::dstPreHookCall()`, the amount of assets to be minted is defined as `consumedUnderlying <= lockAmount` in line 124. The corresponding shares to be minted will be:

* `x = yieldToken__.calculateMintAmount(consumedUnderlying)`

After minting `x` shares, the `_totalSupply` of the yield token will also increase by `x`.

The problem arises when the yield token's `_totalSupply` is updated with a part of `lockAmount` (`consumedAmount` in this case), while the yield token's `totalUnderlyingAssets` is updated with the full amount of lockAmount, causing the share's value to temporarily inflate and potentially be utilized by the attacker to steal other users' funds.

Consider the following example:

1. Assume that the vault doesn't use any strategy to generate yield. We have some initial state:
	* `yieldToken._totalSupply = 100`
	* `yieldToken.totalUnderlyingAssets = 100`
	* `yieldToken._balanceOf[Bob] = 50`

2. Alice wants to bridge `100` tokens from Arbitrum to Aavo. Due to the limit of the `_receivingLimitParams` of the connector, when the function `Controller_YieldLimitExecHook::dstPreHookCall()` is triggered, the `consumedUnderlying` is just `50`. We get the following state:

	* `yieldToken.totalUnderlyingAssets = 100 + 100 = 200`

	* `yieldToken._totalSupply = 100 + 50 = 150`

3. At this point, the value of the share increases from `100 / 100 = 1` to `200 / 150 = 1.3` without using any strategy to generate yield.

4. Bob notices this change and bridges his `50` share tokens from Aavo to Arbitrum. By doing so, Bob will receive `50 * 200 / 150 = 66` tokens.

	* yieldToken.totalUnderlyingAsset = 200 - 66 = 134
	* yieldToken._totalSupply = 150 - 50 = 100
5. Alice calls `Controller::retry()` to mint her remaining shares. The amount of shares she receives is:
	* `sharesToMint = ceil(100 * 50 / 134) = 38`

In the case above:

* Bob generates a profit of `66 - 50 = 16` tokens.
* Alice loses `100 - (50 + 38) = 12` shares.

The above scenario can be amplified to steal the majority of underlying assets.

## Code Snippet
```solidity
    totalUnderlyingAssets =
        totalUnderlyingAssets +
        newTotalUnderlyingAssets -
        oldTotalUnderlyingAssets;


    if (params_.transferInfo.amount == 0)
        return (abi.encode(0, 0), transferInfo);


    (uint256 consumedUnderlying, uint256 pendingUnderlying) = _limitDstHook(
        params_.connector,
        params_.transferInfo.amount
    );
    uint256 sharesToMint = yieldToken__.calculateMintAmount(
        consumedUnderlying
    );
```

## Remediation
Consider minting an amount of shares corresponding to the `pendingUnderlying` to an address owned by the protocol in the function `Controller::receiveInbound()`. Then, transfer the shares from that address to the receiver in the `retry()` function instead of minting new ones.
