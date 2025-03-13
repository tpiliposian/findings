# Wrong price calculation in tBTCChainlinkAdapter leads constraint to work wrongly

## Summary
The `tBTCChainlinkAdapter.sol` contract claims to provide a `tBTC/BTC` price feed but the calculation formula actually returns `BTC/tBTC`, which is the inverse of the expected value. This discrepancy leads to incorrect price constraints in the `OraclePriceConstraint.sol` contract, potentially allowing minting operations when they should be restricted and vice versa.

## Finding Description
The `OraclePriceConstraint.sol` contract clearly indicates that the expected feed should provide the tBTC/BTC ratio:

```solidity
    /// @notice Precision of the asset price from the feed
    uint256 public immutable ASSET_FEED_PRECISION;
```
so does the adapter's self-description:

```solidity
    function description() external view returns (string memory) {
        return "tBTC/BTC Chainlink Adapter";
    }
```

However, the actual calculation in `tBTCChainlinkAdapter.sol::_convertAnswer()` is:

```solidity
    function _convertAnswer(int256 btcUsdPrice, int256 tBtcUsdPrice) private view returns (int256) {
        return
            (btcUsdPrice * TBTC_USD_PRECISION * ADAPTER_PRECISION) / 
            (BTC_USD_PRECISION * tBtcUsdPrice);
    }
```

## Impact Explanation
The `OraclePriceConstraint` contract relies on the price feed to enforce minting constraints. With the inverted price ratio, price threshold checks will be applied incorrectly, so minimum price constraints will be evaluated against an inverted ratio. This means minting operations may be allowed when they should be restricted.

The specific impact is demonstrated in the proof of concept below, where the incorrect formula results in the `canMint()` function returning true when it should return false, compromising the protocol's economic model.

## Likelihood Explanation
The likelihood is highest, as the formula is wrong, so it will always calculate wrongly inverted.

## Proof of Concept
Import the `AggregatorV3Interface` in the `tBTCChainlinkAdapterTests` contract:

```solidity
import {AggregatorV3Interface} from "../src/Dependencies/AggregatorV3Interface.sol";
```

and add the following test/poc in the mentioned contract and run `forge test --match-test testOraclePoc -vv`:

```solidity
    function testOraclePoc() public {
        uint256 fork = vm.createFork("https://eth-mainnet.g.alchemy.com/v2/JJfBwJUqcn4uKNtb_GJdNxd1V8VvGep3");
        vm.selectFork(fork);

        address tBtcUsdFeed = 0x8350b7De6a6a2C1368E7D4Bd968190e13E354297;
        address btcUsdFeed = 0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c;
        //Deploy the tBTCChainlinkAdapter
        tBTCChainlinkAdapter adapter = new tBTCChainlinkAdapter(AggregatorV3Interface(tBtcUsdFeed), AggregatorV3Interface(btcUsdFeed));
        
        //Calculating as it is in the tBTCChainlinkAdapter and this test
        ( , int256 answer, , ,) = adapter.latestRoundData();
        console.log("\nThe answer calculated with the wrong formula from tBTCChainlinkAdapter");
        console.log("answer = _convertAnswer(btcUsdPrice, tBtcUsdPrice) =", answer);
        
        //Calcultaing with both formulas
        AggregatorV3Interface TBTC_USD_CL_FEED;
        AggregatorV3Interface BTC_USD_CL_FEED;
        TBTC_USD_CL_FEED = AggregatorV3Interface(tBtcUsdFeed);
        BTC_USD_CL_FEED = AggregatorV3Interface(btcUsdFeed);
        (, int256 tBtcUsdPrice, , , ) = TBTC_USD_CL_FEED.latestRoundData();
        (, int256 btcUsdPrice, , , ) = BTC_USD_CL_FEED.latestRoundData();

        int256 TBTC_USD_PRECISION = int256(10 ** TBTC_USD_CL_FEED.decimals());
        int256 BTC_USD_PRECISION = int256(10 ** BTC_USD_CL_FEED.decimals());
        int256 ADAPTER_PRECISION = int256(10 ** 18);

        int256 answerOld = (btcUsdPrice * TBTC_USD_PRECISION * ADAPTER_PRECISION) / (BTC_USD_PRECISION * tBtcUsdPrice);

        int256 answerNew = (BTC_USD_PRECISION * tBtcUsdPrice * ADAPTER_PRECISION) / (btcUsdPrice * TBTC_USD_PRECISION);

        console.log("\nThe answer calculated with both formulas and without decimals for simplicity");

        console.log("TBTC_USD_CL_FEED", tBtcUsdPrice / TBTC_USD_PRECISION);
        console.log("BTC_USD_CL_FEED", btcUsdPrice / BTC_USD_PRECISION);

        console.log("answerOLD, i.e. BTC / tBTC", answerOld / ADAPTER_PRECISION);
        console.log("with precision for comparing", answerOld);
        console.log("answerNEW, i.e. tBTC / BTC", answerNew / ADAPTER_PRECISION);
        console.log("with precision for comparing", answerNew);

        //check the revert of the OraclePriceConstraint::canMint
        // function canMint(
        //     uint256 _amount,
        //     address _minter
        //     ) external view returns (bool, bytes memory) {
        //     uint256 assetPrice = _getAssetPrice();
        //     /// @dev peg price is 1e18
        //     uint256 minAcceptablePrice = (1e18 * minPriceBPS) / BPS;

        //     if (minAcceptablePrice <= assetPrice) {
        //         return (true, "");
        //     } else {
        //         return (
        //             false,
        //             abi.encodeWithSelector(
        //                 BelowMinPrice.selector,
        //                 assetPrice,
        //                 minAcceptablePrice
        //             )
        //         );
        //     }
        // }
        uint256 BPS = 10000;
        uint256 minPriceBPS = BPS;
        uint256 assetPriceOld = uint256(answerOld); //uint256 assetPrice = _getAssetPrice();
        uint256 assetPriceNew = uint256(answerNew); //uint256 assetPrice = _getAssetPrice();
        /// @dev peg price is 1e18
        uint256 minAcceptablePrice = (1e18 * minPriceBPS) / BPS;

        //the OraclePriceConstraint::canMint as it is now, is not reverting with the OLD price because
        //OLD price is > 1, but in reality it should revert as the NEW correct proce is 0,98 so < 1

        //     if (minAcceptablePrice <= assetPrice) {
        //         return (true, "");
        vm.assumeNoRevert();
        if (minAcceptablePrice <= assetPriceNew) revert();

        vm.expectRevert();
        if (minAcceptablePrice > assetPriceOld) revert();

    }
```

Output:

  ![Screenshot 2025-03-10 at 10.09.03.png](https://imagedelivery.net/wtv4_V7VzVsxpAFaxzmpbw/dee89a61-513a-4c84-581a-45f7d1f10800/public)  

## Recommendation
Change the price calculation in the `_convertAnswer()` function as mentioned in the poc:

```diff
    function _convertAnswer(int256 btcUsdPrice, int256 tBtcUsdPrice) private view returns (int256) {
        return
  -         (btcUsdPrice * TBTC_USD_PRECISION * ADAPTER_PRECISION) / 
  -         (BTC_USD_PRECISION * tBtcUsdPrice);
  +         (BTC_USD_PRECISION * tBtcUsdPrice * ADAPTER_PRECISION) / 
  +         (btcUsdPrice * TBTC_USD_PRECISION); 
    }
```
