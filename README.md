# defi-projects
## Thales Market 
### 网址
https://thalesmarket.io/
### 介绍
Thales 是一个以太坊协议，允许创建任何人都可以加入的点对点平局市场。这个构建模块是新颖的链上倡议的基础，从基于 AMM 的定价市场到沉浸式游戏化体验，以及更多。
### 核心代码
```
function _buyFromAMM(
        address market,
        IThalesAMM.Position position,
        uint amount,
        uint expectedPayout,
        uint additionalSlippage,
        bool sendSUSD,
        uint sUSDPaidCarried
    ) internal returns (uint sUSDPaid) {
        require(isMarketInAMMTrading(market), "Market is not in Trading phase");

        sUSDPaid = sUSDPaidCarried;

        uint basePrice = price(market, position);
        uint basePriceOtherSide = ONE - basePrice;

        basePrice = basePrice < minSupportedPrice ? minSupportedPrice : basePrice;

        uint availableToBuyFromAMMatm = _availableToBuyFromAMMWithBasePrice(market, position, basePrice, false);
        require(amount > 0 && amount <= availableToBuyFromAMMatm, "Not enough liquidity.");

        if (sendSUSD) {
            sUSDPaid = _buyFromAmmQuoteWithBasePrice(
                market,
                position,
                amount,
                basePrice,
                basePriceOtherSide,
                safeBoxFeePerAddress[msg.sender] > 0 ? safeBoxFeePerAddress[msg.sender] : safeBoxImpact
            );
            require((sUSDPaid * ONE) / (expectedPayout) <= (ONE + additionalSlippage), "Slippage too high");

            sUSD.safeTransferFrom(msg.sender, address(this), sUSDPaid);
        }

        uint toMint = _getMintableAmount(market, position, amount);
        if (toMint > 0) {
            liquidityPool.commitTrade(market, toMint);
            IPositionalMarket(market).mint(toMint);
            spentOnMarket[market] = spentOnMarket[market] + toMint;
        }
        liquidityPool.getOptionsForBuy(market, amount - toMint, position);

        address target = _getTarget(market, position);
        IERC20Upgradeable(target).safeTransfer(msg.sender, amount);

        if (address(stakingThales) != address(0)) {
            stakingThales.updateVolume(msg.sender, sUSDPaid);
        }
        _updateSpentOnMarketOnBuy(market, sUSDPaid, msg.sender);

        if (amount > toMint) {
            uint discountedAmount = amount - toMint;
            uint paidForDiscountedAmount = (sUSDPaid * discountedAmount) / amount;
            emit BoughtWithDiscount(msg.sender, discountedAmount, paidForDiscountedAmount);
        }

        _sendMintedPositionsAndUSDToLiquidityPool(market);

        emit BoughtFromAmm(msg.sender, market, position, amount, sUSDPaid, address(sUSD), target);
        _handleInTheMoneyEvent(market, position, sUSDPaid, msg.sender);
    }

    function _handleInTheMoneyEvent(
        address market,
        IThalesAMM.Position position,
        uint sUSDPaid,
        address sender
    ) internal {
        (bytes32 key, uint strikePrice, ) = IPositionalMarket(market).getOracleDetails();
        uint currentAssetPrice = priceFeed.rateForCurrency(key);
        bool inTheMoney = position == IThalesAMM.Position.Up
            ? currentAssetPrice >= strikePrice
            : currentAssetPrice < strikePrice;
        emit BoughtOptionType(msg.sender, sUSDPaid, inTheMoney);
    }

    function _getTarget(address market, IThalesAMM.Position position) internal view returns (address target) {
        (IPosition up, IPosition down) = IPositionalMarket(market).getOptions();
        target = address(position == IThalesAMM.Position.Up ? up : down);
    }

    function _updateSpentOnMarketOnSell(
        address market,
        uint sUSDPaid,
        uint sUSDFromBurning,
        address seller,
        uint safeBoxShare
    ) internal {
        if (safeBoxImpact > 0) {
            sUSD.safeTransfer(safeBox, safeBoxShare);
        }

        spentOnMarket[market] =
            spentOnMarket[market] +
            (IPositionalMarketManager(manager).reverseTransformCollateral(sUSDPaid + (safeBoxShare)));
        if (spentOnMarket[market] <= IPositionalMarketManager(manager).reverseTransformCollateral(sUSDFromBurning)) {
            spentOnMarket[market] = 0;
        } else {
            spentOnMarket[market] =
                spentOnMarket[market] -
                (IPositionalMarketManager(manager).reverseTransformCollateral(sUSDFromBurning));
        }

        if (referrerFee > 0 && referrals != address(0)) {
            uint referrerShare = (sUSDPaid * ONE) / (ONE - (referrerFee)) - (sUSDPaid);
            _handleReferrer(seller, referrerShare, sUSDPaid);
        }
    }
```

## Pyth Network Benchmarks
### 网址
https://pyth.network/benchmarks
### 介绍
一个产品，它提供一组可追溯查询的历史标准化度量。基准将被需要访问历史价格的下游数据用户使用，最终，其他类型的历史数据。 
在这篇文章中，我们介绍了基准的概念，讨论了它们在传统金融世界的用例，并为像 Pyth 这样的高频链上预言机提供未来市场基准的案例。
### 和chainlink比较
Pyth 和 Chainlink 都是为区块链提供数据的预言机服务，但它们有一些不同。以下是两者之间的关键区别： 
核心使命：Pyth 的使命是向最终用户提供高保真、高频的金融市场数据，以保护他们的 DeFi 协议，而 Chainlink 的使命是通过使智能合约能够访问现实世界的数据和链下计算来扩展智能合约的功能 。 
数据提供商：Pyth 的数据来自传统金融和加密行业的金融机构，包括 CBOE、Jane Street、Susquehanna、Two Sigma 等，而 Chainlink 的数据来自中继器，通常称为节点运营商。 
数据源：Pyth 使用来自交易所、交易公司和金融机构的第一方数据，而 Chainlink 从 Kraken 和 Huobi 等交易所获取一些数据，但主要从 BraveNewCoin、CoinMarketCap 和 CoinGecko 等第三方数据聚合器获取数据来提供价格数据 。 
L1 可用性：Pyth 可在多个链上使用，它旨在提供实时数据馈送，刷新率为 300-400 毫秒间隔。相比之下，Chainlink 被构建为推送模型预言机，其刷新率可以从几分钟到几个小时不等。 
总结一下，虽然 Pyth 和 Chainlink 都是预言机服务，但 Pyth 专注于从第一方来源提供高保真、高频的金融市场数据，并且它旨在提供实时数据馈送，具有高刷新率，这使它与 Chainlink 区别开来。

