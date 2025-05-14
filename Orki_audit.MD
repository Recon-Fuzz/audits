![Orki Security Review](https://alextheentreprenerd.publit.io/file/Audit-Review-13-P.png)

# Recon Security Review

## Introduction

Alex The Entreprenerd performed a 1 week Security Review of Orki

This was a Differential Review, comparing the changes performed by the Orki team against Liquity V2's Bold

Repos:
https://github.com/subvisual/orki/

This review uses [Code4rena Severity Classification](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization)

The Manual Review is done as a best effort service, while a lot of time and attention was dedicated to the security review, it cannot guarantee that no bug is left

As a general rule we always recommend doing one additional security review until no bugs are found, this in conjunction with a Guarded Launch and a Bug Bounty can help further reduce the likelihood that any specific bug was missed

Given the short duration of the review, and the vast amount of code

But also considering the small amount of changes that were implemented from Orki, I recommend setting up a Bug Bounty and performing a guarded launch with limited TVL

It's important you always seek additional security reviews as to avoid having a single point of failure in your process


## About Recon

Recon offers boutique security reviews, invariant testing development and is pioneering Cloud Fuzzing as a best practice by offering Recon Pro, the most complete tool to run tools such as Echidna, Medusa, Foundry, Kontrol and Halmos in the cloud with just a few clicks

## About Alex

Alex is a well known Security Researcher that has collaborated with multiple contest firms such as:
- Code4rena - One of the most prolific and respected judges, won the Tapioca contest, at the time the 3rd highest contest pot ever
- Spearbit - Have done reviews for Tapioca, Threshold USD, Velodrome and more
- Recon - Centrifuge Invariant Testing Suite, Corn and Badger invariants as well as live monitoring

## Additional Services by Recon

Recon offers:
- Invariant Testing Audits - We'll write your invariant tests then perform and audit on the code
- Cloud Fuzzing as a Service - The easiest way to run invariant tests in the cloud - Ask about Recon Pro

## Table of Contents

- **Med**
  - M-01 `priceToSqrtPriceX96` seems to behave in unintended ways when the price goes above `uint64`
  - M-02 Redaction: Undo the extra gas check
  - M-03 `swapFromBold` is using `ExactOutput` which is prone to sandwhiching
  - M-04 Fundamental Price Feeds carry non negligible risk in depeg scenarios
- **Economic**
  - E-01 CCR could be homogenized for WETH and LSTs
  - E-02 Economic and Risk Considerations
  - E-03 Mainnet LSTs Slippage Assement
  - E-04 Commented Price Feed Data
- **Analysis**
  - A-01 Euler Logic Looks Fine
  - A-02 Collateral Tokens Review
  - A-03 SuperChain Code Spec
  - A-04 Sequencer Downtime Feed Changes are Ok but the tail risks are increased
  - A-05 Using Collateral with 8 decimals
  - A-06 Swell Chain OFT Due Diligence
- **QA**
  - Q-01 Best to reset allowance
  - Q-02 Deployment Script Comments
  - Q-03 Review, monitor and deploy on all Balancer V2 like (beets, berachain, etc..) pools
  - Q-04 Old Font in SVG
  - Q-05 Absolute import can cause issues with foundry repos that want to import yours
  - Q-06 `Exchanges` Technically `approve` also requires a `safeApprove`
  - Q-07 Borrow past Borrow caps and then trigger CCR to trigger possible shutdown
  - Q-08 SCR & MCR relation can be abused to shutdown specific branches
  - Q-09 `MultiTroveGetter` in Bold was changed
  - Q-10 Event may pass the wrong failing aggregator
  - Q-11 Quill File
- **Gas**
  - G-01 `EulerFlashLoan` `makeFlashLoan` doesn't need to SSTORE `amountRequested`
  - G-02 `mint` can save gas by doing `borrowerOperationsToTroveManager` lookup only when necessary


# M-01 `priceToSqrtPriceX96` seems to behave in unintended ways when the price goes above `uint64`

I've adapted code that I wrote from here:
https://github.com/GalloDaSballo/Lp-Script

I've ran into similar issues when setting up an LP for UniV3

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {LiquidityProvider, UniV3Translator} from "../src/LiquidityProvider.sol";

import {IUniV3Factory, IV3NFTManager, IUnIV3Pool} from "../src/interfaces/IUni.sol";
import {ICurveFactory, ICurvePool} from "../src/interfaces/ICurve.sol";
import {ERC20} from "../src/mocks/ERC20.sol";

import {UniV3Translator} from "ebtc-amm-comparer/UniV3Translator.sol";

import {Math} from "openzeppelin-contracts/contracts/utils/math/Math.sol";
contract TranslatorTests is Test {
  

    uint256 constant DECIMAL_PRECISION = 1e18;
    function priceToSqrtPriceX96(uint256 _price) public pure returns (uint160 sqrtPriceX96) {
        if (_price > (1 << 64)) {
            sqrtPriceX96 = uint160(Math.sqrt(_price / DECIMAL_PRECISION) << 96);
        } else {
            /// @audit this assumes the token 1 is Bold / ETH and that it's price is below that
            sqrtPriceX96 = uint160(Math.sqrt((_price << 192) / DECIMAL_PRECISION));
        }
    }

    function test_equivalence(uint256 token1, uint256 token0) public {
        deployer = new LiquidityProvider();
        UniV3Translator translator = deployer.translator();

        // This ensures we always hit the "below u64 on getSqrtPriceX96GivenRatio"
        // But once we add more precision (1e18), this can change, so the other test should compare values properly
        require(token1 < type(uint64).max);
        require(token0 < type(uint64).max);
        require(token0 < 10e18);
        require(token1 < 200_000e18);

        uint256 price = token1 * 1e18 / token0;

        

        uint256 referenceVal = translator.getSqrtPriceX96GivenRatio(token1, token0);
        uint256 other = priceToSqrtPriceX96(price);

        if(referenceVal > other) {
            optimize_test_equivalence_reference = int256(referenceVal - other);
        } else {
            optimize_test_equivalence_other = int256(other - referenceVal);
        }
    }

    int256 public optimize_test_equivalence_reference;
    int256 public optimize_test_equivalence_other;
   
}

```

## Results

```python
⇾ [PASSED] Optimization Test: TranslatorTests.optimize_test_equivalence_reference()
Test for method "TranslatorTests.optimize_test_equivalence_reference()" resulted in the maximum value: 79228162511981885646111530437
[Call Sequence]
1) TranslatorTests.test_equivalence(uint256,uint256)(2249999999999999999, 589824) (block=2, time=2, gas=12500000, gasprice=1, value=0, sender=0x10000)
[Execution Trace]
 => [call] TranslatorTests.test_equivalence(uint256,uint256)(2249999999999999999, 589824) (addr=0xA647ff3c36cFab592509E13860ab8c4F28781a66, value=0, sender=0x10000)
         => [creation] LiquidityProvider.constructor() (addr=0x54919A19522Ce7c842E25735a9cFEcef1c0a06dA, value=0, sender=0xA647ff3c36cFab592509E13860ab8c4F28781a66)
                 => [creation] UniV3Translator.constructor() (addr=0x8085Fe5dF5e137F474201f8E79b65FcB9D4334ED, value=0, sender=0x54919A19522Ce7c842E25735a9cFEcef1c0a06dA)
                         => [return ()]
                 => [return ()]
         => [call] LiquidityProvider.translator()() (addr=0x54919A19522Ce7c842E25735a9cFEcef1c0a06dA, value=<nil>, sender=0xA647ff3c36cFab592509E13860ab8c4F28781a66)
                 => [return (0x8085Fe5dF5e137F474201f8E79b65FcB9D4334ED)]
         => [call] UniV3Translator.getSqrtPriceX96GivenRatio(uint256,uint256)(2249999999999999999, 589824) (addr=0x8085Fe5dF5e137F474201f8E79b65FcB9D4334ED, value=<nil>, sender=0xA647ff3c36cFab592509E13860ab8c4F28781a66)
                 => [return (154742504910672534328003304686517214)]
         => [return ()]

[Optimization Test Execution Trace]
[Execution Trace]
 => [call] TranslatorTests.optimize_test_equivalence_reference()() (addr=0xA647ff3c36cFab592509E13860ab8c4F28781a66, value=0, sender=0x10000)
         => [return (79228162514229950370230467550)]
```

## Mitigation

Ensure that every price you pass to the function is below `uint64.max`

# M-02 Redaction: Undo the extra gas check

## Impact

This check:

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/PriceFeeds/MainnetPriceFeedBase.sol#L109

```solidity
 if (uint256(gasleft()) + uint256(2000) <= gasBefore / 64) revert InsufficientGasForExternalCall();
```

Is meant to ensure that sufficient gas was provided before the revert

It's original version 

```solidity
if (uint256(gasleft()) <= gasBefore / 64) revert InsufficientGasForExternalCall();
```

Is incorrect as it's ignoring the gas that is necessary

In the case of an oracle reverting due to consuming too much gas this will cause a revert

The check:
```solidity
 if (uint256(gasleft()) + uint256(2000) <= gasBefore / 64) revert InsufficientGasForExternalCall();
```

Will not, but this seems to open up to being able to actually triggering a shutdown

This requires very specific gas requirements I didn't fully test

But given the fact that the other code is being reviewed by hundreds of auditors, and the POC below, I believe the change should be undone

## POC
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";


contract MockCLFeed {
    enum Behaviour {
        NORMAL,
        REVERT,
        GASGRIEF
    }

    Behaviour behaviour = Behaviour.NORMAL;

    function setBehaviour(Behaviour b) external {
        behaviour = b;
    }

    function latestRoundData() external view returns (
            uint80 roundId, int256 answer, uint256, /* startedAt */ uint256 updatedAt, uint80 /* answeredInRound */
        ) {
            
            if(behaviour == Behaviour.REVERT) {
                revert("No out of gas");
            }

            if(behaviour == Behaviour.GASGRIEF) {
                // Grief them, burn all gas
                uint256 i;
                while (true) {
                    i++;
                }
            }

            if(behaviour == Behaviour.NORMAL) {
                // Grief them, burn all gas
                uint256 i;
                while (i < 10_000) {
                    /// About 300k gas
                    i+=1;
                }
            }


            answer = 123;   
        }
    

}

contract PriceLibTester is Test {

    bool public hasShutdown;

    function getCurrentChainlinkResponse(MockCLFeed _aggregator)
        external
        returns (int256 price)
    {
        uint256 gasBefore = gasleft();

        // Try to get latest price data:
        try _aggregator.latestRoundData() returns (
            uint80 roundId, int256 answer, uint256, /* startedAt */ uint256 updatedAt, uint80 /* answeredInRound */
        ) {


            return answer;
        } catch {
            // NOTE: The check is ignoring additional costs that come from processing the error + the call
            // So even thought the check is directionally right
            // You would need to give a few thousands gas of leniency to the check to actually be safe
            
            // Require that enough gas was provided to prevent an OOG revert in the call to Chainlink
            // causing a shutdown. Instead, just revert. Slightly conservative, as it includes gas used
            // in the check itself.
            if (gasleft() + 2000 <= gasBefore / 64) revert("InsufficientGasForExternalCall()");

            // This adds about 20k gas so prob not feasible
            hasShutdown = true;

            // If call to Chainlink aggregator reverts, return a zero response with success = false
            return -1;
        }
    }

    // forge test --match-test test_normal_and_revert_case -vv
    function test_normal_and_revert_case() public {
        MockCLFeed feed = new MockCLFeed();

        
        feed.setBehaviour(MockCLFeed.Behaviour.NORMAL);
        this.getCurrentChainlinkResponse(feed);
        console.log("Base case ok");

        
        feed.setBehaviour(MockCLFeed.Behaviour.REVERT);
        this.getCurrentChainlinkResponse(feed);
        console.log("Revert case ok");

        // NOTE: This fails without adding 2k gas (done)
        feed.setBehaviour(MockCLFeed.Behaviour.GASGRIEF);
        this.getCurrentChainlinkResponse(feed);
        console.log("Gas Grief Case ok");




    }

    function test_we_may_trigger_shutdown_branch_on_low_gas() public {
        MockCLFeed feed = new MockCLFeed();

        // After adding 2k gas, let's make sure we cannot trigger a revert
        feed.setBehaviour(MockCLFeed.Behaviour.NORMAL);
        this.getCurrentChainlinkResponse{gas: 1_000_000}(feed);
        console.log("Base Case Low Gas");
    }


    function test_fuzz_can_trigger_shutdown_on_low_gas(uint64 gas) public {
        MockCLFeed feed = new MockCLFeed();

        // After adding 2k gas, let's make sure we cannot trigger a revert
        feed.setBehaviour(MockCLFeed.Behaviour.NORMAL);
        try this.getCurrentChainlinkResponse{gas: gas}(feed) {
            console.log("Base Case Low Gas");

            assertTrue(!this.hasShutdown(), "Never true");
        } catch {

        }

    }

    function test_fuzz_repro() public {
        MockCLFeed feed = new MockCLFeed();

        // After adding 2k gas, let's make sure we cannot trigger a revert
        feed.setBehaviour(MockCLFeed.Behaviour.NORMAL);
        this.getCurrentChainlinkResponse{gas: 1239413}(feed);
        console.log("Base Case Low Gas");

        assertTrue(!this.hasShutdown(), "Never true");
    }

    
}
```

## Mitigation

Revert back to 
```solidity
if (uint256(gasleft()) <= gasBefore / 64) revert InsufficientGasForExternalCall();
```

I believe the only safe way to check that sufficient gas was provided is to check before the calls, with an hardcoded value

# M-03 `swapFromBold` is using `ExactOutput` which is prone to sandwhiching

## Impact

The function `swapFromBold` is used to swap from bold borrowed back to Collateral (flashloaned in some cases)


https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/Zappers/Modules/Exchanges/VelodromeSlipstreamExchange.sol#L29-L56

```solidity

    function swapFromBold(uint256 _boldAmount, uint256 _minCollAmount) external {
        ISlipstreamSwapRouter slipstreamRouterCached = slipstreamRouter;

        // Set initial balances to make sure there are not lefovers
        InitialBalances memory initialBalances;
        _setInitialTokensAndBalances(collToken, boldToken, initialBalances);

        boldToken.transferFrom(msg.sender, address(this), _boldAmount);
        boldToken.approve(address(slipstreamRouterCached), _boldAmount);

        ISlipstreamSwapRouter.ExactOutputSingleParams memory params = ISlipstreamSwapRouter.ExactOutputSingleParams({
            tokenIn: address(boldToken),
            tokenOut: address(collToken),
            tickSpacing: tickspacing,
            recipient: msg.sender,
            deadline: block.timestamp,
            amountOut: _minCollAmount,
            amountInMaximum: _boldAmount,
            sqrtPriceLimitX96: 0 // See: https://ethereum.stackexchange.com/a/156018/9205
        });

        slipstreamRouterCached.exactOutputSingle(params);

        // return leftovers to user
        _returnLeftovers(initialBalances);
    }

```

This uses the `UniV3.ExactOutputSingleParams` which asserts the following:
- Given this `max` amount in
- Give me this `exact` amount out

This has 2 possible impacts
1) Non Sandwiched case: We don't swap the bold fully, we get some bold and w/e coll was passed as "minCollAmount"
This is the intended case afaict, but this relies ENTIRELY on using a protected RPC and not being attacked
(This is not checking the implied ratio between bold and collateral, and you're hoping the ratio is a non manipulated one)

2) Sandwiched Case: The boldAmount is consumed in it's entirety to return exactly the collAmount
This converts the "benign case" to a literal case, where 100% of the bold is consumed for exactly that collAmount
This is the risk that comes from using `ExactOutputSingleParams` without any validation to a bold / coll ratio

## Mitigation

I believe you should use `ExactInputSingleParams` at all times as it does a better job at protecting you against slippage

Or you should check the price and use a ratio to determine if the price you received was acceptable

# M-04 Fundamental Price Feeds carry non negligible risk in depeg scenarios

## Executive Summary

Fundamental rates tend to over-estimate prices, which is good to prevent against redemption arbitrage

They tend to carry additional risks for the borrowing side

## Considerations

All "fundamental" prices are generally good to price redemptions as they will take:
ETH/USD * RATE which we expect (generally, but not always) to be the MAX(ETH/USD*RATE vs LSTPRICE / USD)

The risk I see in using these is that you can open up to overborrowing

- See that the LST market has depegged on Mainnet or Swell Chain
- Bridge / Buy the token (10 mins delay at most)
- Borrow at maximum (most likely the token would be liquidatable, likelyhood of bad debt is pretty low until the token massively depegs)

There's 2 sides to this:
- Some protocols prefer not to liquidate on short price changes, meaning in some cases this choice is acceptable
- The protocol will be unable to react to actual emergency scenarios and will instead rely completely on governance to trigger a shutdown


Overborrowing if you use these feeds should mostly cause LPs to be less willing to LP, but since the system will not acknowledge a depeg it won't trigger liquidations

There's still a non negligible risk in having the system becoming effectively insolvent, which would require governance to act rapidly

## Mitigation

You could reduce LTV to a point that ensures that the mismatch between the fundamental feed and the market feed is negligible | TODO: Investigate relative volatility

You should also put in place extensive monitoring to ensure that no massive price change can be used agains the system

Given the fact that all tokens are OFTs you should expect that if a price mismatch were to happen, it would be weaponized very rapidly

## Relative Volatility of LSTs over ETH

Using Redstone Mainnet Price Feeds I have computed the relative price changes on LSTs over ETH

Some LSTs have had period of decoupled price changes, however even in-spite of it, the changes were fairly contained

I believe it will be necessary for you to setup monitoring that will trigger shutdown if any LST depegs too aggressively

However, given the available historical data I don't believe the LSTs have shown aggressive decoupling from ETH so far, meaning you'd need the monitoring for a black swan event more so than to combat arbitrage

# E-01 CCR could be homogenized for WETH and LSTs

Given the relative stability of all these assets
As well as the mandatory need for monitoring the oracle update (due to the oracles not being to warn around depeg scenarios)

I believe that all LSTs could have their CCR lowered to that of WETH, which is already fairly conservative

# E-02 Economic and Risk Considerations

## Executive Summary

- SWELL can benefit by raising the MCR to 150 / 170 due to extreme volatility and low float
- SWELL should have a very conservative borrow limit due to low float and high concentration
- LSTs use Fundamental prices which requires monitoring to trigger a emergency shutdown if the LST trades below parity by a wide margin which could cause unrealized bad debt to the protocol

## Considerations

All "fundamental" prices are generally good to price redemptions as they will take:
ETH/USD * RATE which we expect (generally, but not always) to be the MAX(ETH/USD*RATE vs LSTPRICE / USD)

The risk I see in using these is that you can open up to overborrowing

- See that the LST market has depegged on Mainnet or Swell Chain
- Bridge / Buy the token (10 mins delay at most)
- Borrow at maximum (most likely the token would be liquidatable, likelyhood of bad debt is pretty low until the token massively depegs)

There's 2 sides to this:
- Some protocols prefer not to liquidate on short price changes, meaning in some cases this choice is acceptable
- The protocol will be unable to react to actual emergency scenarios and will instead rely completely on governance to trigger a shutdown


Overborrowing if you use these feeds should mostly cause LPs to be less willing to LP, but since the system will not acknowledge a depeg it won't trigger liquidations

There's still a non negligible risk in having the system becoming effectively insolvent, which would require governance to act rapidly

## Deviation Threshold

Having a 50 BPS dev threshold and a consistent 6 hours heartbeat seems to indicate that no partic

## Compounded Deviation (fundamental * ETH/USD)

```ts
double drift {
  spot: 1000000000000000000,
  max: 1010025000000000000,
  min: 990025000000000100
}
max up 101.0025
max down 99.00250000000001
```

This ends up not being an issue because all fundamental feeds have repricing that is within just a few BPS in the worst case

Due to this, we can expect some theoretical possible small arbitrages, however the size of said arbitrages is so small they should in practice never happen

If we change the "advertised BPS deviation" to an upper estimate of the realized deviation of 5 BPS, then we get:
```ts
double drift {
  spot: 1000000000000000000,
  max: 1005502500000000100,
  min: 994502499999999900
}
max up 100.55025000000002
max down 99.45024999999998
```

Meaning we can eliminate the vast majority of arbitrages with 50 BPS base fee, and should have close to zero at 55

Arbitrages that attack the delay between the oracle triggering an update round and actually updating it's value are still possible (and are always possible) and should be monitored

## Slippage Analysis

I'm unable to perform any meaningful slippage analysis on Swell Chain as no aggregator is present

## Mainnet Swell Liquidity

Liquidity for the Swell token on mainnet is abysmal, with both Cowswap and 1Inch throwing a 30% slippage after a 15k swap

Coingeko: https://www.coingecko.com/en/coins/swell-network , shows that on average a 20k sell / buy will cause a 2% price change

This, mixed with the fairly high volatility, extremely low liquid supply, and concentrated supply in multiple 4/6 wallets makes the asset extremely risky

Overal:
75%+ of supply is held by 4, 4/6 wallets, no timelock
Liquidity is extremely low

Historical volatility has been very high, but low supply makes most price action "abnormal"

130 MCR  is 76% LTV

In the historical data we can see that a 30% price swing within 4 hours has happened, a 70% price change has happened within a day

Given the low float, low liquidity it may be best to change the CR to a more conservative 150 / 170

Additionally, given the very low liquidity, I believe that the borrow limits should be reduced to ensure that at most $25k of SWELL would have to be liquidated at any given time

As such I'd recommend starting with a 25k borrow limit, and then raise is slowly by estimating the average ICR

## Mainnet Swell Pricing

The relative price changes between ETH and SWELL seem to indicate that because of it's low float, the price is trading as beta of ETH

I don't believe this indicates a wide distribution, and ultimately I have to warn that the SWELL token is currently not liquid enough to be used as collateral in any significant portion of the system

# E-03 Mainnet LSTs Slippage Assement

## Executive Summary

Below are listed amounts after which a swap on Cowswap on Mainnet will incur around 10% Slippage

This is a very basic and superficial check that can help determine >> at this time << whether a liquidation could happen in a way that will be profitable to a liquidator (and stakers to the SP)

It's important you monitor these parameters and ensure borrow limits are set in a way that would allow liquidations to happen profitably

Note that liquidators will contend this liquidity amongst multiple lending markets, meaning the effective amount liquidatable will be lower

Also note that because of the StabilityPool, the first round of liquidations (until the SP is emptied) will happen safely to the system, but that may be unprofitable to users

### rswETH / ETH | 300e18

<img width="540" alt="Image" src="https://github.com/user-attachments/assets/465ffb0e-89a9-4b0d-b718-30babb629915" />

### weETH / ETH | 17_500e18

<img width="494" alt="Image" src="https://github.com/user-attachments/assets/2ced349d-a268-48ee-81e0-7048f7897a8a" />

### swETH / ETH | 260e18

<img width="539" alt="Image" src="https://github.com/user-attachments/assets/80610105-d436-4653-b38a-6cb9ee477b81" />


### WETH / USDC | 40_000e18

At 50k Cowswap glitches out

# E-04 Commented Price Feed Data

## 0x3fd49f2146FE0e10c4AE7E3fE04b3d5126385Ac4 - weeth_eth_oracle - weeth_fundamental
https://app.redstone.finance/app/feeds/swellchain/weeth_fundamental/

### Comment

No particular concern from a technical POV

It's pricing the underlying exchange rate

NOTE: This doesn't include Market Data, meaning this is probably good for Redemptions but may over-price for borrowing

### Prices

pricesWithinTimePeriod 438
mean 105904885.33561644
STANDARD DEVIATION 236837.00627180975
as percent of mean 0.223631804634191
getHighestAndLowestPrice 106319262 105496105


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739204257, price: 105983544 },
  { date: 1738601217, price: 105913598 }
]
deviation 0.06604062303690221


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1737643211, price: 105830924 },
  { date: 1737578379, price: 105811262 }
]
deviation 0.018582142985876116


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738000331, price: 105864277 },
  { date: 1737987071, price: 105860605 }
]
deviation 0.0034687124639047736


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739234177, price: 105987608 },
  { date: 1739232519, price: 105986493 }
]
deviation 0.001052020845712859

## 0x4BAD96DD1C7D541270a0C92e1D4e5f12EEEA7a57 - rsweth_eth_oracle - fundamental

https://app.redstone.finance/app/feeds/swellchain/rsweth_fundamental/

### Comment

Also doesn't express market risk

### Prices

pricesWithinTimePeriod 439
mean 103231098.85649203
STANDARD DEVIATION 303334.9156110117
as percent of mean 0.29384063423823126
getHighestAndLowestPrice 103725555 102702541


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1741675809, price: 103671863 },
  { date: 1741092337, price: 103517932 }
]
deviation 0.14869984071938377


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1741675809, price: 103671863 },
  { date: 1741610977, price: 103566476 }
]
deviation 0.10175783136620387


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738032331, price: 103219212 },
  { date: 1738025829, price: 103188150 }
]
deviation 0.030102293722680366

interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing []

## 0x5C4c8d6f6Bf79B718F3e8399AaBdFEd01cB7e48f - swell_usd_oracle - Market

https://app.redstone.finance/app/feeds/swellchain/swell/


pricesWithinTimePeriod 2853
mean 1627895.710129688
STANDARD DEVIATION 550027.2732305255
as percent of mean 33.78762348275413
getHighestAndLowestPrice 2970275 983000


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738550807, price: 1171430 },
  { date: 1737950941, price: 2284919 }
]
deviation 48.73209947486104
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739226937, price: 1959196 },
  { date: 1739138847, price: 1092664 }
]
deviation 79.30452545338731


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738550807, price: 1171430 },
  { date: 1738504667, price: 1691408 }
]
deviation 30.7423164606056
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739226937, price: 1959196 },
  { date: 1739140899, price: 1131591 }
]
deviation 73.13640705873412


interval 14400 - ⚠️ - 30% in 4 hours!

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1737332919, price: 2353884 },
  { date: 1737318907, price: 2748890 }
]
deviation 14.369654660608463
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739226937, price: 1959196 },
  { date: 1739214867, price: 1505140 }
]
deviation 30.167027651912782


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1739228937, price: 1768389 },
  { date: 1739226937, price: 1959196 }
]
deviation 9.739046016835477
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1737950941, price: 2284919 },
  { date: 1737948349, price: 2034732 }
]
deviation 12.295820776397088

## 0x3587a73AA02519335A8a6053a97657BECe0bC2Cc - sweth_eth_oracle -  sweth_fundamental 
https://app.redstone.finance/app/feeds/swellchain/sweth_fundamental/

### Comment

No particular concern from a technical POV

It's pricing the underlying exchange rate

NOTE: This doesn't include Market Data, meaning this is probably good for Redemptions but may over-price for borrowing

### Prices

pricesWithinTimePeriod 439
mean 108309596.97494306
STANDARD DEVIATION 260259.50232533683
as percent of mean 0.24029218979140574
getHighestAndLowestPrice 108745988 107841858


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1740833017, price: 108575013 },
  { date: 1740244217, price: 108464092 }
]
deviation 0.10226518099649054


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1741675809, price: 108679286 },
  { date: 1741610977, price: 108621010 }
]
deviation 0.053650762407751504


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738032639, price: 108293974 },
  { date: 1738025829, price: 108254000 }
]
deviation 0.0369261182034844


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing []


## 0x197225B3B017eb9b72Ac356D6B3c267d0c04c57c - rseth_eth_oracle - fundamental

https://app.redstone.finance/app/feeds/swellchain/rseth_fundamental/

### Comment

Also fundamental, meaning it may not express market risks fully

### Prices

pricesWithinTimePeriod 441
mean 103587074.14739229
STANDARD DEVIATION 249129.73395476383
as percent of mean 0.24050272295584035
getHighestAndLowestPrice 104026650 103164427


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739160727, price: 103674882 },
  { date: 1738557997, price: 103590966 }
]
deviation 0.08100706387852392


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738644437, price: 103620020 },
  { date: 1738579607, price: 103590966 }
]
deviation 0.02804684725113964


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1739160727, price: 103674882 },
  { date: 1739156887, price: 103664948 }
]
deviation 0.00958279552698951


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing []

## 0xbbF121624c3b85C929Ac83872bf6c86b0976A55e - ezeth_eth_oracle - Fundamental

https://app.redstone.finance/app/feeds/swellchain/ezeth_fundamental/

Also doesn't express market risk

pricesWithinTimePeriod 438
mean 103547628.75799087
STANDARD DEVIATION 340928.88222529273
as percent of mean 0.3292483722848969
getHighestAndLowestPrice 104114362 102973279


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1737362279, price: 103386920 },
  { date: 1737254227, price: 103417514 }
]
deviation 0.02958299693802348
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738709267, price: 103646387 },
  { date: 1738123629, price: 103492595 }
]
deviation 0.14860193620616047


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1734413499, price: 103007043 },
  { date: 1734391887, price: 103033045 }
]
deviation 0.025236563667510748
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1740330657, price: 103835273 },
  { date: 1740265827, price: 103776842 }
]
deviation 0.05630446916085575


interval 14400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738871865, price: 103650190 },
  { date: 1738862441, price: 103651563 }
]
deviation 0.0013246302904279408
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1735907397, price: 103228599 },
  { date: 1735893997, price: 103199634 }
]
deviation 0.028066960004916295


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing []
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738169019, price: 103501332 },
  { date: 1738166849, price: 103501239 }
]
deviation 0.00008985399682026996

## 0xe7f71d6a24EBc391f5ee57B867ED429EB7Bd74f4 - eth_usd_oracle - eth usd market

https://app.redstone.finance/app/feeds/swellchain/eth/

pricesWithinTimePeriod 2576
mean 280677621479.16614
STANDARD DEVIATION 52486729602.112785
as percent of mean 18.700005125278118
getHighestAndLowestPrice 373229856000 178935908530


interval 604800

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738548467, price: 225544000000 },
  { date: 1738340167, price: 340800000000 }
]
deviation 33.81924882629108
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738623847, price: 288278000000 },
  { date: 1738548467, price: 225544000000 }
]
deviation 27.814528429042667


interval 86400

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738548467, price: 225544000000 },
  { date: 1738462127, price: 314350129170 }
]
deviation 28.250705480694684
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738623847, price: 288278000000 },
  { date: 1738548467, price: 225544000000 }
]
deviation 27.814528429042667


interval 14400 - ⚠️ - 24% in 4 hours!

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738548467, price: 225544000000 },
  { date: 1738534389, price: 293350420170 }
]
deviation 23.11447862617868
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1740938389, price: 247991939750 },
  { date: 1740925517, price: 219133289530 }
]
deviation 13.169450557647547


interval 3600

eth_usd_swings.pointsOfBiggestNegativeSwing [
  { date: 1738548467, price: 225544000000 },
  { date: 1738546297, price: 279154166056 }
]
deviation 19.204501517360654
eth_usd_swings.pointsOfBiggestPositiveSwing [
  { date: 1738552059, price: 251152619312 },
  { date: 1738548467, price: 225544000000 }
]
deviation 11.354156755222927

# A-01 Euler Logic Looks Fine

## Summary

Imo the Euler logic is safe

The EVK code flows as follows:
- It will not use the EVC (since it's called directly)
- It will trigger the reentrancy guard
- The guard prevents from using the collateral to alter EVC state
- Your code never reads EVC state
- Your code returns the exact amount requested (see QA finding)



# A-02 Collateral Tokens Review

## WeETH - 0xA6cB988942610f6731e664379D15fFcfBf282b44

OFT Upgradeable with custom functionality

Owner:
https://explorer.swellnetwork.io/address/0x6D685276540271076e2c07eD6ed6Ce351549bA11

a 3/6 gnosis safe (unverified, but seems to be a gnosis safe)
Can use this to debug:
https://callthis.link/

The owner can, at any time add a MINTER_ROLE and mint unbacked assets

## RSWETH - 0x18d33689AE5d02649a859A1CF16c9f0563975258

Default OFT

## RSETH - 0xc3eACf0612346366Db554C991D7858716db09f58

Default OFT

## SWETH - 0x09341022ea237a4DB1644DE7CCf8FA0e489D85B7

Default OFT

## SWBTC - 0x1cf7b5f266A0F39d6f9408B90340E3E71dF8BF7B

Default OFT (but has 8 decimals)

# A-03 SuperChain Code Spec

## Deterministic Deployment Risks

The multichain deployment is deterministic and works in this way:
- Use the CREATE2 deployer with a specific nonce

https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainTokenBridge.sol
https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/OptimismSuperchainERC20Factory.sol


## Real OP Spec

https://specs.optimism.io/interop/token-bridging.html#superchainerc20-standard

## Alternatives:

- Do not use the spec and use mint and burn
- Wait for the spec to improve and I believe they will allow to use the [OptimismSuperchainERC20Factory](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/OptimismSuperchainERC20Factory.sol) to turn ERC20s into super ERC20s (Not working as of now)
- Setup for Deterministic addresses and deal with extra risks

The key challenge is in setting up a deterministic deployment that is not manipulatable in any way, specifically if you deploy the `BoldToken` logic which could be used to mint unbacked USDK

From looking at `DeployOrkiSwellchainMainnet.s` I can tell the following:
- `quillAccessManager`
- `boldToken`
- `collateralRegistry`
- `hintHelpers`
- `multiTroveGetter`

Are all deployed deterministically, via `Upgrades.sol` because once you pass `upgradeOptions.salt` it will use Create2

These are then verified to be valid

The reason why I believe the deployments to be safe is:
- The `quillAccessManager` gives the permission via `initialize` which is part of the constructor for the UUPS meaning it is deterministically enforced
- The `boldToken` will use the `quillAccessManager` in its `initialize` meaning these 2 are tied in a deterministic way

The deployment on the main chain should be straightforward

The deployment on all other chains will always set the owner / admin to the `eoa` `deployer`, this means that every future deployment could be subject to a PK leak

This is because an `authority` can setup a fake `collRegistry` and then steal fiunds via `mint`

So this operation would have to be bricked

It may be possible to brick the operations by enforcing that `mint` is only done via the `SuperChainERC20` perhaps by hardcoding a constant for chainID (meaning every deployment of BoldToken that is not on swell chain won't be able to use the `BoldToken` functions)

Overall I think that the setup seems sufficient , however, I believe that wrapping USDK into a superUSDK and then deploying those in a simpler way is probably going to also work in the future and carry less overall risk to all parts of the stack

From my research it seems clear that the SuperERC20 standard is nascent and not fully developed, the fact that most documents "imply" that you need to deploy deterministically without an easy way to do so leads me to believe you'd be amongst the first to use the standard, and as such will bear a ton of the risk

## Limitations on the SuperchainERC20 Spec

The standard doesn't clarify how to canonically determine the chain from which the caller is performing the request

I believe it to be the [crossDomainMessageSource](https://github.com/ethereum-optimism/optimism/blob/9f480717e640591768939ef4cd97bc010f4e2ff0/packages/contracts-bedrock/src/L2/L2ToL2CrossDomainMessenger.sol#L102-L103)

This can help verify from which chain the caller is bridging from and to

Ensuring that each mint and burn comes from a "safe chain" seems like a theoretically impossible task

In practice, deploying each token with a "disabled interop" first mode and then enabling each chain separately could work

However this fails in the case of a malicious initial deployer (which we have to threat model for), this leads me to believe that you will have to brick native functionality (mint from a malicious BO) for every deployment that is not the canonical one

On the theoretical attacks side, if an attacker were to deploy on all chains at ones they would most likely be able to find a chain of chains for which the deployer is compromised or for which the chain enables minting, overall these attacks are poorly researched and can be critical, I highly recommend you let someone else explore these instead of your project

## Notes on the SuperchainERC20 standard

This seems to be the very light set of requirements for the token to become a SuperchainERC20

https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainERC20.sol

Here's the full EIP: https://github.com/defi-wonderland/ERCs/blob/bf1ebe7d946d69c90c9f443df658fd37521b69f7/ERCS/erc-7802.md

This seems to be the current version of the SuperBridge: https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SuperchainTokenBridge.sol

NOTE: OP is currently going a contest on Cantina, some changes may happen due to findings found during said contest

### Thoughts

The safest change would be to wrap `USDK` into a `superUSDK` wrapper, this has zero risks, allows to easily specify solvency invariants that can be fairly simply monitored and carries no risk to the system

Upgrading seems to be completely feasible because the spec simply requires granting additional permissions to the `SUPERCHAIN_TOKEN_BRIDGE`

From a superficial review the token bridge looks innocent and the code removes the need for allowances

The only gotchas would be interacting with system contracts

### Broken Invariants

If we allowed the bridge to mint to system contracts (e.g. SP, BO, TM), then we may break some superficial accounting properties

Similarly a simple check like `_requireValidRecipient` could be added

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/BoldToken.sol#L145-L151

```solidity

    function _requireValidRecipient(address _recipient) internal view {
        require(
            _recipient != address(0) && _recipient != address(this),
            "Bold: Cannot transfer tokens directly to the Bold token contract or the zero address"
        );
    }
```

Overall I would not expect any particular risk, provided you perform the upgrade with a SuperERC20Wrapper contract that will look like the following:

```solidity
contract SuperERC20Wrapper is BoldToken, IERC7802 {
  // Expand code by using `BoldToken` logic, without any additional storage slot
}
```

### Comment

Nowhere is stated that addresses must match, and yet it seems obvious, and implied

I'm not a fan of this aspect

For example Balancer having predictable non deployed contracts can be weaponized, meaning Balancer V2 should never be used with a SuperERC20 token

### Long tail risks

A predictable address can:
- Be griefed by farming a nonce that clashes (2^88 odds, cost: Billions)
- Cause an exploit similar to: https://x.com/kankodu/status/1893206310105080058 for all contracts that do not check for contract existence
- Be subject to risks if any of the chains performs an irregular state transition (which is afaik completely possible given the single sequencer and the optimistic nature of the chain)


## TL;DR

- Don't use the standard, wait for other projects to trailblaze
- If you want to use it anyway, change BoldToken to not function on every chain except canonical (hardcode a simple require(chainID == MAIN_CHAIN_ID) as to ensure the deterministic deployments will not mint unbacked tokens
- Private key compromise of the Deployer is still a critical risk, this is why I believe you should not proceed at this time
- I highly recommend waiting for a standardized wrapper with no private key leak risks, I would be surprised if OP isn't working on this


# A-04 Sequencer Downtime Feed Changes are Ok but the tail risks are increased

## Impact

`requireUpAndOverGracePeriod` was changed as follows:

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/Quill/SequencerSentinel/Swellchain.sol#L39-L53

```solidity
    /// @inheritdoc ISequencerSentinel
    function requireUpAndOverGracePeriod() public view {
        if(address(sequencerFeed) == address(0)) {
            return ;
        }

        (, int256 answer, uint256 startAt,,) = sequencerFeed.latestRoundData();
        if (answer != 0) {
            revert SequencerDown();
        }
        if (block.timestamp - startAt < gracePeriod) {
            revert SystemUnderGracePeriod();
        }
    }

```

This means that when no feed is set it will never revert

I believe that until a sequencer feed is up, this is fine

## Key risks wrt not having a sequencer uptime feed

Forced inclusion on OP stack (default config) means that a tx can be denied for up to 12 hours
After 12 hours the tx can be manually sequenced (NOTE: I'm not 100% confident in this as I did not audit this behaviour)
https://docs.optimism.io/stack/rollup/outages#bypassing-the-sequencer
https://specs.optimism.io/protocol/derivation.html#l2-chain-derivation-pipeline

This leads me to believe that in the worst case scenario, oracle updates could be delayed by at least 12 hours
I would also expect oracle providers to require a bit more time to coordinate and be able to push updates during this window

This leads us to consider 2 options:
- A shorter Shutdown Staleness check (e.g. 6 hours), would cause a ton of redemptions while the L2 is inaccessible
- A longer Shutdown Staleness check (e.g. 24 hours), would prevent the shutdown but would possible allow performing operations on very old oracle prices

A longer shutdown delay increases the likelihood that borrowers will borrow with a old (lower) price that would possibly allow locking-in bad debt

In this scenario, the self-liquidation BCR may also be bypassed and used to cause more bad debt

These are very low likelihood scenarios but generally speaking these long-tail events are why having a sequencer check increases the safety of the protocol

# A-05 Using Collateral with 8 decimals


## The system can handle 8 decimals by upscaling the oracle return value by 1e10

I believe that the system can safely handle different decimals by upscaling the oracle, instead of requiring you to wrap the collateral

Below are a few key functions that use price, tested to demonstrate that (excluding the precision loss) the results are comparable

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";


contract ScaledMathTests is Test {

    function setUp() public {
    }

    uint256 constant DECIMAL_PRECISION = 1e18;

    function _computeCR(uint256 _coll, uint256 _debt, uint256 _price) internal pure returns (uint256) {
        // if (_debt > 0) {
        uint256 newCollRatio = _coll * _price / _debt; /// @audit CR needs to be in 18 decimals

        return newCollRatio;
    }

    function _getCollPenaltyAndSurplus(
        uint256 _collToLiquidate,
        uint256 _debtToLiquidate,
        uint256 _penaltyRatio,
        uint256 _price
    ) internal pure returns (uint256) {
        uint256 maxSeizedColl = _debtToLiquidate * (DECIMAL_PRECISION + _penaltyRatio) / _price;

        return maxSeizedColl;
    }

    

    function _redemptionMath(uint256 boldLot, uint256 price) public returns (uint256) {
        uint256 correspondingColl = boldLot * DECIMAL_PRECISION / price;
        return correspondingColl;
    }


    uint256 scaling = 1e10;

    function test_math_cr(uint128 debt, uint128 coll, uint128 price) public {
        vm.assume(debt > 0 && coll > scaling && price > 0);

        // Need truncation cause we lose precision, else we cannot simulate a proper check
        uint256 normal = _computeCR(coll / scaling * scaling, debt, price);
        uint256 scaled = _computeCR(coll / scaling, debt, price * scaling);

        vm.assume(normal > 0);

        assertEq(normal, scaled, "same res");
    }

    function test_math_penalty(uint128 debt, uint128 coll, uint128 price) public {
        vm.assume(debt > 0 && coll > scaling && price > 0);

        // Need truncation cause we lose precision, else we cannot simulate a proper check
        uint256 normal = _getCollPenaltyAndSurplus(coll / scaling * scaling, debt, 20e16, price);
        uint256 scaled = _getCollPenaltyAndSurplus(coll / scaling, debt, 20e16, price * scaling);

        vm.assume(scaled > 0);

        assertEq(normal / scaling, scaled, "same res");
    }



    function test_math_redemption(uint128 debt, uint128 price) public {
        vm.assume(debt > 0 && price > 0 && price < 1e15);

        /// @audit Redemption Precision DOES change
        /// Ultimately there's precision loss this is within the bounds of the scaling amount
        /// TODO: Fuzz maximum delta loss of precision
        // TODO: HALMOS

        // Need truncation cause we lose precision, else we cannot simulate a proper check
        uint256 normal = _redemptionMath(debt, price);
        uint256 scaled = _redemptionMath(debt, price * scaling);

        vm.assume(scaled > 0);

        console2.log("normal", normal);
        console2.log("scaled", scaled);

        assertEq(normal / scaling, scaled, "same res");
    }
}

```

## Recommendation

Extend all of these tests to ensure that the code will work

I have shown a pretty decent way of testing this, however I would not consider my work complete


## Zappers will break

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/Zappers/Modules/Exchanges/VelodromeSlipstreamExchange.sol#L85-L90

```solidity
    /// @audit this only works for 18 decimals prices
    function priceToSqrtPrice(IBoldToken _boldToken, IERC20 _collToken, uint256 _price) public pure returns (uint160) {
        // inverse price if Bold goes first
        uint256 price = _zeroForOne(_boldToken, _collToken) ? DECIMAL_PRECISION * DECIMAL_PRECISION / _price : _price;
        return priceToSqrtPriceX96(price);
    }
```

# A-06 Swell Chain OFT Due Diligence

## Executive Summary

The following is a set of notes derived from reviewing the current configuration of the collaterals in-scope deployed on Swell Chain


## Economic

### SWELL

87 MLN dollars of SWELL can be dropped in 12 seconds by those 4 signers

(on a 114 MCap)

All 4/6 with the same owners

https://etherscan.io/address/0x2019fe73c426e74fed2a140d7e5b577117a6dc1a#readProxyContract
https://etherscan.io/address/0x3b3472e3db5b3a2026ac33cac12ace8eb229a1fa#readProxyContract

https://etherscan.io/address/0x7815ba83da2e47b3d4386586216e2b1d57c36a6d#readProxyContract
https://etherscan.io/address/0x4b9c95fdd52f170bb38cd7ca959524f1b46a2bef#readProxyContract

Liquity is going to be very low
-------

## Integration Risk

### WETH

Literally the OP Predeploy


### swETH

https://explorer.swellnetwork.io/address/0x09341022ea237a4DB1644DE7CCf8FA0e489D85B7?tab=contract

lzEndpoint (not verified)
https://explorer.swellnetwork.io/address/0xcb566e3B6934Fa77258d68ea18E931fa75e1aaAa?tab=txs

Delegate: Can cause losses (EOA)
https://explorer.swellnetwork.io/address/0xaf994551f4f940224825F54F810ed5439651E5f9

#### L0 Config (Sender)


### rsETH OFT (kelp dao)

https://explorer.swellnetwork.io/address/0xc3eACf0612346366Db554C991D7858716db09f58?tab=contract

Same lzEndpoint

Different Delegate: (EOA)
https://explorer.swellnetwork.io/address/0xCF7bF9D3a485b9c4d4A38d65e4d68e081a585662


### rswETH OFT

https://explorer.swellnetwork.io/address/0x18d33689AE5d02649a859A1CF16c9f0563975258?tab=contract

Same lzEndpoint

Different Delegate: (EOA)
https://explorer.swellnetwork.io/address/0xbeaf552D6B169e33c99f466bc9B2b11453453D42

### weETH (ether fi) | TODO: Prob a l0 with extra steps (https://layerzeroscan.com/address/0xa6cb988942610f6731e664379d15ffcfbf282b44)

https://explorer.swellnetwork.io/address/0xA6cB988942610f6731e664379D15fFcfBf282b44?tab=contract

### Swell OFT

https://explorer.swellnetwork.io/address/0x2826D136F5630adA89C1678b64A61620Aab77Aea?tab=contract

Same lzEndpoint
2
Different Delegate: (EOA)
https://explorer.swellnetwork.io/address/0xaf994551f4f940224825F54F810ed5439651E5f9

DVN = EOA


## Observations

Delegates have the ability to change the endpoint (TODO: Confirm)

And the validation library

They can be a source of DOS and can (TODO) also mint tokens

The DVN instead, in lack of any customization (not present afaict) can MINT and BURN AT WILL, this is a critical single point of failure

### Delegate Risks

Clear

https://github.com/LayerZero-Labs/LayerZero-v2/blob/9a4049ae3a374e1c0ef01ac9fb53dd83f4257a68/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol#L211

Skip, Nillify, Burn

https://github.com/LayerZero-Labs/LayerZero-v2/blob/9a4049ae3a374e1c0ef01ac9fb53dd83f4257a68/packages/layerzero-v2/evm/protocol/contracts/MessagingChannel.sol#L83

They can DOS and cause Losses to all users, however they cannot mint unbacked assets


### DVN Risks

Can mint and burn and forge all messages

## Mitigations

- All Delegates MUST be upgraded to Multisigs with Timelock, example and instructions are available here:
https://x.com/getreconxyz/status/1885249716386226572

- DVN config

I am not a fan of using DVNs without Storage Proofs in general

That said, afaik the default number of DVNs for LayerZero should be 2 (L0 + nethermind), that would be a start

# Q-01 Best to reset allowance

## Impact

`setProviders` looks as follows:

https://github.com/subvisual/orki/blob/9f6bfba7735940992aa7fb276bc1f37a44264424/contracts/src/Zappers/WETHZapper.sol#L23-L34

```solidity
    function setProviders(IFlashLoanProvider _flashLoanProvider, IExchange _exchange)
        public
        virtual
        override(BaseZapper)
        restricted
    {
        super.setProviders(_flashLoanProvider, _exchange);

        // Approve Coll to exchange module (for closeTroveFromCollateral)
        WETH.approve(address(_exchange), type(uint256).max);
    }

```

There's no code to reset the allowance of WETH against the previous `_exchange`

This most likely won't cause issues, however revoking allowances is strictly safer

# Q-02 Deployment Script Comments

## QA - Use `_1pct` everywhere

https://github.com/subvisual/orki/blob/9f6bfba7735940992aa7fb276bc1f37a44264424/contracts/script/DeployOrkiSwellchainMainnet.s.sol#L120-L131

```solidity
        troveManagerParamsArray[0] = TroveManagerParams({
            CCR: 140e16,
            MCR: 110e16,
            SCR: 110e16,
            BCR: 10e16,
            LIQUIDATION_PENALTY_SP: 5e16,
            LIQUIDATION_PENALTY_REDISTRIBUTION: 5 * _1pct, /// @audit QA: Standardize
            MIN_DEBT: 500e18,
            SP_YIELD_SPLIT: 75e16,
            minAnnualInterestRate: 5 * _1pct, /// @audit pretty high
            branchCap: WETH_BORROW_CAP
        });
```


## QA / MED: Are you sure you want to use the 1% Fee Tier?

FROM: https://medium.com/@VelodromeFi/velodrome-slipstream-88cd4229ad05


> Suggested tick spacings are:

> Stable pairs use 0.05% fee tier, with tick spacing of 10
> Established pairs (e.g. OP/ETH), .3% fee tier, with tick spacing of 60
> Volatile pairs, using 1% fee tier, with tick spacing of 200

This means you're setting up a ETH/USDK 1% pool (prob ok)
But also a USD/USDK 1% pool, which is not great, for stable pairs you should  set a lower fee pool (50 BPS due to redemptions and lower if there's demand)

https://github.com/subvisual/orki/blob/9f6bfba7735940992aa7fb276bc1f37a44264424/contracts/script/DeployOrkiSwellchainMainnet.s.sol#L404-L416

```solidity
        // USDK/USDC
        InitSlipstreamLiquidityPoolArgs memory usdk_usdc_args;
        usdk_usdc_args.deployer = deployer;
        usdk_usdc_args.poolFactory = slipstreamPoolFactory;
        usdk_usdc_args.positionManager = slipstreamPositionManager;
        usdk_usdc_args.tokenA = r.boldToken;
        usdk_usdc_args.tokenB = usdc;
        usdk_usdc_args.amountADesired = 1e18;
        usdk_usdc_args.amountBDesired = 1e6; 
        usdk_usdc_args.tickSpacing = 200; // TODO: should the spacing be minimal? | /// @audit this implies the fee so yes
        usdk_usdc_args.upscaledPrice = DECIMAL_PRECISION / USDC_PRECISION * DECIMAL_PRECISION;

        initSlipstreamLiquidityPool(usdk_usdc_args); /// @audit can get griefed, but prob OK
```

### Recommendation

Use 1% Fee Tier for ETH/USDK and 30 BPS for USDC/USDK


## Slipstream

https://github.com/subvisual/orki/blob/9f6bfba7735940992aa7fb276bc1f37a44264424/contracts/script/VelodromeLiquidityPools.s.sol#L93-L96

```solidity
    vars.TICK_SPACING = ICLPool(pool).tickSpacing();
    (, vars.tick,,,,) = ICLPool(pool).slot0();
    vars.tickLower = (vars.tick - 6000) / vars.TICK_SPACING * vars.TICK_SPACING; /// @audit MASSIVE swings | 400X LP | 2500 is 12 x prob ok as well
    vars.tickUpper = (vars.tick + 6000) / vars.TICK_SPACING * vars.TICK_SPACING;
```

https://github.com/subvisual/orki/blob/9f6bfba7735940992aa7fb276bc1f37a44264424/contracts/script/VelodromeLiquidityPools.s.sol#L97-L111

```solidity

    ISlipstreamNonfungiblePositionManager.MintParams memory params = ISlipstreamNonfungiblePositionManager.MintParams({
        token0: vars.tokens[0],
        token1: vars.tokens[1],
        tickSpacing: vars.TICK_SPACING,
        tickLower: vars.tickLower,
        tickUpper: vars.tickUpper,
        amount0Desired: vars.amounts[0],
        amount1Desired: vars.amounts[1],
        amount0Min: 0, /// @audit 0 looks off, but maybe it's OK
        amount1Min: 0,
        recipient: args.deployer,
        deadline: block.timestamp + 10 minutes,
        sqrtPriceX96: 0 // vars.sqrtPriceX96 // if != 0, tries to create a new pool
    });
```


## Recommendations

- Use _1pct everywhere
- Ensure the Create2 Script works as intended
- Set USDC/USDK to use 30 BPS or lower
- Ensure all prices you pass are below uint64 to avoid any gotcha
- 

# Q-03 Review, monitor and deploy on all Balancer V2 like (beets, berachain, etc..) pools

If you want to maintain a cross chain strategy, you should not only deploy the token on all chains but also ensure that the LP venues have not been attacked

Balancer V2 recently released a disclosure: https://forum.balancer.fi/t/balancer-v2-token-frontrun-vulnerability-disclosure/6309

This to means that you should NOT use Balancer V2, or alternatively should ensure that all chains that contain a Balancer V2 deployment didn't have the attack setup on them

# Q-04 Old Font in SVG

## Impact

The function `_styles` is using `geist`

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/NFTMetadata/utils/baseSVG.sol#L39-L49

```solidity
    function _styles(FixedAssetReader _assetReader) private view returns (string memory) {
        return svg.el(
            "style",
            utils.NULL,
            string.concat(
                '@font-face { font-family: "Geist"; src: url("data:font/woff2;utf-8;base64,',
                _assetReader.readAsset(bytes4(keccak256("geist"))),
                '"); }'
            )
        );
    }
```

But in the rest of the code you replaced it with `denim`


# Q-05 Absolute import can cause issues with foundry repos that want to import yours

## Impact

This line is an absolute import

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/NFTMetadata/MetadataNFT.sol#L4-L5

```solidity
import "lib/Solady/src/utils/SSTORE2.sol";

```

From my experience working with Liquity this can break compilation for project that want to import your code

## Mitigation

Use a relative import or use `import "Solady/utils/SSTORE2.sol";`

# Q-06 `Exchanges` Technically `approve` also requires a `safeApprove`

## Impact

The function `swapToBold` looks as follows:

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/Zappers/Modules/Exchanges/VelodromeSlipstreamExchange.sol#L57-L65

```solidity
    function swapToBold(uint256 _collAmount, uint256 _minBoldAmount) external returns (uint256) {
        ISlipstreamSwapRouter slipstreamRouterCached = slipstreamRouter;

        // Set initial balances to make sure there are not lefovers
        InitialBalances memory initialBalances;
        _setInitialTokensAndBalances(collToken, boldToken, initialBalances);

        collToken.safeTransferFrom(msg.sender, address(this), _collAmount);
        collToken.approve(address(slipstreamRouterCached), _collAmount);
```

It's using `safeTransferFrom` but a non-safe `approve`

For tokens that misbehave with `transferFrom` this will not be sufficient

## Mitigation

Use `forceApprove` / `safeApprove` from OZ

# Q-07 Borrow past Borrow caps and then trigger CCR to trigger possible shutdown

## Impact

Because Orki introduced Borrow Caps as well as CCR borrow shutdown the following theoretical attack is possible:

An attack can borrow until the borrow caps to ensure nobody else can borrow
They can then wait for their position to go below CCR (which also prevents borrowing) and slowly wait for it to reach the SCR, which will permanently disable the branch

This is very low likelihood as it requires the branch having only one borrower

It's also worth noting that Orki can add back shutdown collaterals by deploying a new branch

## Mitigation

Monitor for abuse, or ensure you have a super healthy position to prevent griefing

# Q-08 SCR & MCR relation can be abused to shutdown specific branches

## Impact

In scope config:

```
| Collateral | CCR | MCR | SCR | Pen_SP | Pen_RD | MIN_DEBT | SP_Yield | MIN_AIR | Borrow CAP |
| ---------- | :-: | :-: | :-: | -----: | -----: | -------: | -------: | ------: | ---------: |
| WETH       | 140 | 110 | 110 |      5 |      5 |    $ 500 |       75 |       5 |     $ 1.5M |
| SWETH      | 150 | 115 | 115 |      5 |    7.5 |    $ 500 |       75 |       5 |       $ 1M |
| RSWETH     | 160 | 120 | 120 |      5 |    7.5 |    $ 500 |       75 |       5 |       $ 1M |
| WEETH      | 160 | 120 | 120 |      5 |    7.5 |    $ 500 |       75 |       5 |       $ 1M |
| SWELL      | 170 | 130 | 130 |      5 |     10 |    $ 500 |       75 |       5 |     $ 500K |
```

-> SCR needs to be a bit below MCR else you can shutdown the branch with one trove via
-> Open to CCR -> Let it go down
-> Shutdown
-> Close position


## Mitigation 

Either ensure you will open a position on each branch, which seems to be the case

Or change the SCR to be below the MCR to prevent this edge case

# Q-09 `MultiTroveGetter` in Bold was changed

## Impact

The bold repo was changed to use `currId = _startId == 0 ? sortedTroves.getLast() : _startId;`

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/MultiTroveGetter.sol#L134-L136

```solidity
        data = new DebtPerInterestRate[](_maxIterations);
        currId = sortedTroves.getPrev(_startId);
```


https://github.com/liquity/bold/blob/8c797232fc285260f988e0943c34cf53cb6c6c3b/contracts/src/MultiTroveGetter.sol#L139-L140

```solidity
        data = new DebtPerInterestRate[](_maxIterations);
        currId = _startId == 0 ? sortedTroves.getLast() : _startId;
```

The change seems more intuitive, so I think you should implement it

# Q-10 Event may pass the wrong failing aggregator

## Impact

`_shutDownAndSwitchToLastGoodPrice` will emit the failing aggregator address

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/PriceFeeds/MainnetPriceFeedBase.sol#L83

```solidity
ShutDownFromOracleFailure(_failedOracleAddr)
```

The check in `QuillCompositePriceFeed` will always pass `ethUsdOracle.aggregator` even when `lstEthOracleDown` could be true

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/Quill/PriceFeeds/QuillCompositePriceFeed.sol#L66-L68

```solidity
        if (ethUsdOracleDown || lstEthOracleDown) {
            return (_shutDownAndSwitchToLastGoodPrice(address(ethUsdOracle.aggregator)), true); /// @audit QA: Could be the lstEthOracleDown
        }
```

## Mitigation

Acknowledge or fix it by passing the correct address by performing separate checks

# Q-11 Quill File

AuditDelta is probably stale

# G-01 `EulerFlashLoan` `makeFlashLoan` doesn't need to SSTORE `amountRequested`

## Impact

`_amount` is stored into `amountRequested`

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/Zappers/Modules/FlashLoans/EulerFlashLoan.sol#L49-L54

```solidity
        receiver = IFlashLoanReceiver(msg.sender);
        amountRequested = _amount; /// @audit this is == flashLoanAmount
        /// @audit I think some params in userData and _amount overlap
        eVault.flashLoan(_amount, userData); /// @audit TODO: Trace Euler FL and check invariants
    }

```

From looking at all calls `flashLoanProvider.makeFlashLoan` we can see that `_amount == _params.flashLoanAmount`

Meaning you don't need to store the `amountRequested` in storage

# G-02 `mint` can save gas by doing `borrowerOperationsToTroveManager` lookup only when necessary

## Impact

The function `mint` looks as follows:

https://github.com/subvisual/orki/blob/0e5f372d6bd37e794700e46b7be4974d7787319a/contracts/src/BoldToken.sol#L94-L111

```solidity
    function mint(address _account, uint256 _amount) external override {
        bool isBO = _requireCallerIsBOorAP();
        address _cachedTMAddress = borrowerOperationsToTroveManager[msg.sender];

        // if is AP, allow to mint interest, otherwise check for max minting cap
        if (isBO) {
            uint256 _maxBranchCap =
                ICollateralRegistry(collateralRegistryAddress).getTokenCapByAddress(_cachedTMAddress);
            uint256 _currentBranchCap = ITroveManager(_cachedTMAddress).getEntireBranchDebt();

            // current branch cap already accounts for _amount value
            if (_currentBranchCap > _maxBranchCap) {
                revert OverBranchCapLimit(_cachedTMAddress);
            }
        }

        _mint(_account, _amount);
    }
```

It will cache `address _cachedTMAddress = borrowerOperationsToTroveManager[msg.sender];`

However, this is sued only if `isBO` is true

## Mitigation

Place `address _cachedTMAddress = borrowerOperationsToTroveManager[msg.sender];` inside the `if` clause

## Additional Services by Recon

Recon offers:
- Invariant Testing Audits - We'll write your invariant tests then perform and audit on the code
- Cloud Fuzzing as a Service - The easiest way to run invariant tests in the cloud - Ask about Recon Pro
