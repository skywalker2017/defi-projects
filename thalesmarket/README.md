# Thales Market 
## 网址
https://thalesmarket.io/
## 介绍
Thales 是一个以太坊协议，允许创建任何人都可以加入的点对点平局市场。这个构建模块是新颖的链上倡议的基础，从基于 AMM 的定价市场到沉浸式游戏化体验，以及更多。
## 概念
### 彩金市场
### 代币化头寸市场
### 代币化范围市场
### 快速市场
## 核心代码
### priceImpact
priceImpact和slippage的区别：priceImpact是处理头寸买入前后赔率的偏差，slippage是USDT, DAI, sUSD这些抵押物价格的偏差
```javascript
    /// @notice 获取在买入时应用于市场该方向的偏斜影响 
    /// @param market 已知的头寸市场 
    /// @param position UP或DOWN 
    /// @param amount 要购买的头寸数量，小数点后18位 
    /// @return _
    function buyPriceImpact(
        address market,
        IThalesAMM.Position position,
        uint amount
    ) public view returns (int _priceImpact) {
        IThalesAMM.Position positionOtherSide = position == IThalesAMM.Position.Up
            ? IThalesAMM.Position.Down
            : IThalesAMM.Position.Up;
        // 获取赔率
        uint basePrice = price(market, position);

        if (basePrice >= minSupportedPrice && basePrice <= maxSupportedPrice) {
            uint basePriceOtherSide = ONE - basePrice;
            // 可购买的头寸
            uint _availableToBuyFromAMM = _availableToBuyFromAMMWithBasePrice(market, position, basePrice, true);
            uint _availableToBuyFromAMMOtherSide = _availableToBuyFromAMMWithBasePrice(
                market,
                positionOtherSide,
                basePriceOtherSide,
                true
            );
            if (amount > 0 && amount <= _availableToBuyFromAMM) {
                _priceImpact = _buyPriceImpact(
                    BuyPriceImpactParams(
                        market,
                        position,
                        amount,
                        _availableToBuyFromAMM,
                        _availableToBuyFromAMMOtherSide,
                        liquidityPool,
                        basePrice
                    )
                );
            }
        }
    }

    struct PriceImpactParams {
        uint amount;
        uint balanceOtherSide;
        uint balancePosition;
        uint balanceOtherSideAfter;
        uint balancePositionAfter;
        uint availableToBuyFromAMM;
        uint max_spread; // 使用 Thales AMM 进行交易时，对衍生价格可能产生的最大价格影响(如果交易耗尽 AMM 某一侧的所有流动性)
    }

    function buyPriceImpactImbalancedSkew(PriceImpactParams memory params) public view returns (uint) {
        // 最大可能的偏离
        uint maxPossibleSkew = params.balanceOtherSide + params.availableToBuyFromAMM - params.balancePosition;
        // 偏离：买入后balance之差
        uint skew = params.balanceOtherSideAfter - (params.balancePositionAfter);
        uint newImpact = (params.max_spread * ((skew * ONE) / (maxPossibleSkew))) / ONE;
        if (params.balancePosition > 0 && params.amount > params.balancePosition) { // 需要mint 超买
            uint newPriceForMintedOnes = newImpact / (2);
            uint tempMultiplier = (params.amount - params.balancePosition) * (newPriceForMintedOnes);
            return (tempMultiplier * ONE) / (params.amount) / ONE;
        } else {  // 不需要mint
            // newImpact和previousImpact的计算与maxPossibleSkew的计算相似
            // balanceOtherSideAfter + balanceOtherSide - balancePositionAfter
            uint previousSkew = params.balanceOtherSide;
            uint previousImpact = (params.max_spread * ((previousSkew * ONE) / (maxPossibleSkew))) / ONE;
            return (newImpact + previousImpact) / (2);
        }
    }

    function _buyPriceImpact(BuyPriceImpactParams memory params) internal view returns (int priceImpact) {
        if (params.basePrice >= minSupportedPrice && params.basePrice <= maxSupportedPrice) {
            // Up/Down token balance
            (uint balancePosition, uint balanceOtherSide) = ammUtils.balanceOfPositionsOnMarket(
                params.market,
                params.position,
                params.liquidityPool.getMarketPool(params.market)
            );
            // 买入后的balance
            uint balancePositionAfter = balancePosition > params.amount ? balancePosition - params.amount : 0;
            uint balanceOtherSideAfter = balanceOtherSide +
                (balancePosition > params.amount ? 0 : (params.amount - balancePosition));
            if (balancePositionAfter >= balanceOtherSideAfter) {
                // 里面也调了buyPriceImpactImbalancedSkew
                priceImpact = ammUtils.calculateDiscount(
                    IThalesAMMUtils.DiscountParams(
                        balancePosition,
                        balanceOtherSide,
                        params.amount,
                        params._availableToBuyFromAMMOtherSide,
                        max_spread
                    )
                );
            } else {
                if (params.amount > balancePosition && balancePosition > 0) {
                    uint amountToBeMinted = params.amount - balancePosition;
                    uint positiveSkew = ammUtils.buyPriceImpactImbalancedSkew(
                        IThalesAMMUtils.PriceImpactParams(
                            amountToBeMinted,
                            balanceOtherSide + balancePosition,
                            0,
                            balanceOtherSide + amountToBeMinted,
                            0,
                            params._availableToBuyFromAMM - balancePosition,
                            max_spread
                        )
                    );

                    uint pricePosition = price(params.market, params.position);
                    uint priceOtherPosition = price(
                        params.market,
                        params.position == IThalesAMM.Position.Up ? IThalesAMM.Position.Down : IThalesAMM.Position.Up
                    );
                    uint skew = (priceOtherPosition * positiveSkew) / pricePosition;

                    int discount = ammUtils.calculateDiscount(
                        IThalesAMMUtils.DiscountParams(
                            balancePosition,
                            balanceOtherSide,
                            balancePosition,
                            params._availableToBuyFromAMMOtherSide,
                            max_spread
                        )
                    );

                    int discountBalance = int(balancePosition) * discount;
                    int discountMinted = int(amountToBeMinted * skew);
                    int amountInt = int(balancePosition + amountToBeMinted);

                    priceImpact = (discountBalance + discountMinted) / amountInt;

                    if (priceImpact > 0) {
                        int numerator = int(pricePosition) * priceImpact;
                        priceImpact = numerator / int(priceOtherPosition);
                    }
                } else {
                    priceImpact = int(
                        ammUtils.buyPriceImpactImbalancedSkew(
                            IThalesAMMUtils.PriceImpactParams(
                                params.amount,
                                balanceOtherSide,
                                balancePosition,
                                balanceOtherSideAfter,
                                balancePositionAfter,
                                params._availableToBuyFromAMM,
                                max_spread
                            )
                        )
                    );
                }
            }
        }
    }

    function _sellPriceImpact(
        address market,
        IThalesAMM.Position position,
        uint amount,
        uint available
    ) internal view returns (uint _sellImpact) {
        (uint _balancePosition, uint balanceOtherSide) = ammUtils.balanceOfPositionsOnMarket(
            market,
            position,
            liquidityPool.getMarketPool(market)
        );
        uint balancePositionAfter = _balancePosition > 0 ? _balancePosition + (amount) : balanceOtherSide > amount
            ? 0
            : amount - (balanceOtherSide);
        uint balanceOtherSideAfter = balanceOtherSide > amount ? balanceOtherSide - (amount) : 0;
        if (!(balancePositionAfter < balanceOtherSideAfter)) {
            _sellImpact = ammUtils.sellPriceImpactImbalancedSkew(
                amount,
                balanceOtherSide,
                _balancePosition,
                balanceOtherSideAfter,
                balancePositionAfter,
                available,
                max_spread
            );
        }
    }
```
### buyFromAMM
```javascript
    /// @notice 通过USDC、USDT或DAI从AMM购买给定市场的指定类型头寸
    /// @param market 已知的头寸市场
    /// @param position UP or DOWN
    /// @param amount 头寸数量
    /// @param expectedPayout 买家期望支付的金额（通过报价获取）
    /// @param collateral USDC, USDT or DAI
    /// @param additionalSlippage 买家愿意接受的sUSD期望支付的滑点
    /// @param _referrer 谁向Thales推荐了买家
    function buyFromAMMWithDifferentCollateralAndReferrer(
        address market,
        IThalesAMM.Position position,
        uint amount,
        uint expectedPayout,
        uint additionalSlippage,
        address collateral,
        address _referrer
    ) public nonReentrant notPaused returns (uint) {
        //更新推荐人
        if (_referrer != address(0)) {
            IReferrals(referrals).setReferrer(_referrer, msg.sender);
        }
        //抵押物索引
        int128 curveIndex = _mapCollateralToCurveIndex(collateral);
        require(curveIndex > 0 && curveOnrampEnabled, "unsupported collateral");
        //查询报价，返回抵押物报价以及sUSD报价
        (uint collateralQuote, uint susdQuote) = buyFromAmmQuoteWithDifferentCollateral(
            market,
            position,
            amount,
            collateral
        );

        uint transformedCollateralForPegCheck = collateral == usdc || collateral == usdt
            ? collateralQuote * (1e12)
            : collateralQuote;
        //防止稳抵押品报价与sUSD报价差异过大
        require(
            maxAllowedPegSlippagePercentage > 0 &&
                transformedCollateralForPegCheck >= (susdQuote * (ONE - (maxAllowedPegSlippagePercentage))) / ONE,
            "Amount below max allowed peg slippage"
        );
        //滑点检查
        require((collateralQuote * ONE) / (expectedPayout) <= (ONE + additionalSlippage), "Slippage too high!");
        //抵押品转账
        IERC20Upgradeable collateralToken = IERC20Upgradeable(collateral);
        collateralToken.safeTransferFrom(msg.sender, address(this), collateralQuote);
        curveSUSD.exchange_underlying(curveIndex, 0, collateralQuote, susdQuote);
        // 是否send sUSD为false，上面已经转过了
        return _buyFromAMM(market, position, amount, susdQuote, additionalSlippage, false, susdQuote);
    }

    /// @notice 获取选择的抵押品（USDC、USDT或DAI）的报价，以了解交易者需要支付多少钱才能购买指定数量的UP或DOWN头寸
    /// @param market 已知的头寸市场
    /// @param position UP or DOWN
    /// @param amount 要购买的头寸数量，小数点后18位
    /// @param collateral USDT、USDC或DAI地址
    /// @return collateralQuote 抵押品报价，交易者需要支付多少钱才能购买指定数量的UP或DOWN头寸
    /// @return sUSDToPay sUSD报价，交易者需要支付多少钱才能购买指定数量的UP或DOWN头寸
    function buyFromAmmQuoteWithDifferentCollateral(
        address market,
        IThalesAMM.Position position,
        uint amount,
        address collateral
    ) public view returns (uint collateralQuote, uint sUSDToPay) {
        int128 curveIndex = _mapCollateralToCurveIndex(collateral);
        if (curveIndex > 0 && curveOnrampEnabled) {
            sUSDToPay = buyFromAmmQuote(market, position, amount);
            //无法获取sUSD需要多少抵押品的信息，因此，最好获取sUSD报价的抵押品数量，并在此基础上加上0.2%
            //get_dy_underlying是查询稳定币抵押物的价格，再乘以系数防止价格波动
            collateralQuote = (curveSUSD.get_dy_underlying(0, curveIndex, sUSDToPay) * (ONE + (ONE_PERCENT / 5))) / ONE;
        }
    }

    function _mapCollateralToCurveIndex(address collateral) internal view returns (int128) {
        if (collateral == dai) {
            return 1;
        }
        if (collateral == usdc) {
            return 2;
        }
        if (collateral == usdt) {
            return 3;
        }
        return 0;
    }

    function get_dy_underlying(
        int128 i,
        int128 j,
        uint256 _dx
    ) external view returns (uint256) {
        if (j == 1) {
            return (_dx * (ONE - BREAKING_PERCENT)) / ONE;
        } else return ((_dx / 1e12) * (ONE - BREAKING_PERCENT)) / ONE;
    }

function _buyFromAMM(
        address market, // 标的市场
        IThalesAMM.Position position, //头寸 Up/Down
        uint amount, // 头寸
        uint expectedPayout, //预期的支付金额
        uint additionalSlippage, //滑点
        bool sendSUSD,
        uint sUSDPaidCarried //支付的金额，与expectedPayout一致
    ) internal returns (uint sUSDPaid) {
        require(isMarketInAMMTrading(market), "Market is not in Trading phase");
        // 需要支付的sUSD
        sUSDPaid = sUSDPaidCarried;
        // 计算赔率
        uint basePrice = price(market, position);
        uint basePriceOtherSide = ONE - basePrice;

        basePrice = basePrice < minSupportedPrice ? minSupportedPrice : basePrice;
        // 获取在给定的持仓市场中可以购买多少个特定类型（UP或DOWN）的头寸。
        uint availableToBuyFromAMMatm = _availableToBuyFromAMMWithBasePrice(market, position, basePrice, false);
        require(amount > 0 && amount <= availableToBuyFromAMMatm, "Not enough liquidity.");
        // buyFromAMMWithDifferentCollateralAndReferrer 中抵押品转账已经完成，sendSUSD为false
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
        // position代币不足时需要mint
        uint toMint = _getMintableAmount(market, position, amount);
        if (toMint > 0) {
            liquidityPool.commitTrade(market, toMint);
            IPositionalMarket(market).mint(toMint);
            spentOnMarket[market] = spentOnMarket[market] + toMint;
        }
        liquidityPool.getOptionsForBuy(market, amount - toMint, position);
        // market中的Up/Down合约
        address target = _getTarget(market, position);
        // Up/Down代币支付给用户
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
        //下注到资金池
        _sendMintedPositionsAndUSDToLiquidityPool(market);

        emit BoughtFromAmm(msg.sender, market, position, amount, sUSDPaid, address(sUSD), target);
        //处理相关事件
        _handleInTheMoneyEvent(market, position, sUSDPaid, msg.sender);
    }

    function _updateSpentOnMarketOnBuy(
        address market,
        uint sUSDPaid,
        address buyer
    ) internal {
        uint safeBoxShare;
        if (safeBoxImpact > 0) {
            safeBoxShare = (sUSDPaid -
                (sUSDPaid * ONE) /
                (ONE + (safeBoxFeePerAddress[msg.sender] > 0 ? safeBoxFeePerAddress[msg.sender] : safeBoxImpact)));
            sUSD.safeTransfer(safeBox, safeBoxShare);
        }

        if (
            spentOnMarket[market] <= IPositionalMarketManager(manager).reverseTransformCollateral(sUSDPaid - (safeBoxShare))
        ) {
            spentOnMarket[market] = 0;
        } else {
            spentOnMarket[market] =
                spentOnMarket[market] -
                (IPositionalMarketManager(manager).reverseTransformCollateral(sUSDPaid - (safeBoxShare)));
        }

        if (referrerFee > 0 && referrals != address(0)) {
            uint referrerShare = sUSDPaid - ((sUSDPaid * ONE) / (ONE + (referrerFee)));
            _handleReferrer(buyer, referrerShare, sUSDPaid);
        }
    }

    function _buyFromAmmQuoteWithBasePrice(
        address market,
        IThalesAMM.Position position,
        uint amount,
        uint basePrice,
        uint basePriceOtherSide,
        uint safeBoxImpactForCaller
    ) internal view returns (uint returnQuote) {
        uint _available = _availableToBuyFromAMMWithBasePrice(market, position, basePrice, false);
        uint _availableOtherSide = _availableToBuyFromAMMWithBasePrice(
            market,
            position == IThalesAMM.Position.Up ? IThalesAMM.Position.Down : IThalesAMM.Position.Up,
            basePriceOtherSide,
            true
        );

        if (amount <= _available) {
            int tempQuote;
            int skewImpact = _buyPriceImpact(
                BuyPriceImpactParams(market, position, amount, _available, _availableOtherSide, liquidityPool, basePrice)
            );
            basePrice = basePrice + (min_spreadPerAddress[msg.sender] > 0 ? min_spreadPerAddress[msg.sender] : min_spread);
            if (skewImpact >= 0) {
                int impactPrice = ((ONE_INT - int(basePrice)) * skewImpact) / ONE_INT;
                // add 2% to the price increase to avoid edge cases on the extremes
                impactPrice = (impactPrice * (ONE_INT + (ONE_PERCENT_INT * 2))) / ONE_INT;
                tempQuote = (int(amount) * (int(basePrice) + impactPrice)) / ONE_INT;
            } else {
                tempQuote = ((int(amount)) * ((int(basePrice) * (ONE_INT + skewImpact)) / ONE_INT)) / ONE_INT;
            }
            tempQuote = (tempQuote * (ONE_INT + (int(safeBoxImpactForCaller)))) / ONE_INT;
            returnQuote = IPositionalMarketManager(manager).transformCollateral(uint(tempQuote));
        }
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


    /// @notice 获取市场给定一侧的基本价格（赔率）
    /// @param market 已知的头寸市场
    /// @param position UP or DOWN
    /// @return priceToReturn 市场给定一侧的基本价格（赔率)
    function price(address market, IThalesAMM.Position position) public view returns (uint priceToReturn) {
        if (isMarketInAMMTrading(market)) {
            // add price calculation
            IPositionalMarket marketContract = IPositionalMarket(market);
            // 到期日
            (uint maturity, ) = marketContract.times();
            // 剩余时间
            uint timeLeftToMaturity = maturity - block.timestamp;
            // 剩余天数
            uint timeLeftToMaturityInDays = (timeLeftToMaturity * ONE) / 86400;
            uint oraclePrice = marketContract.oraclePrice();

            (bytes32 key, uint strikePrice, ) = marketContract.getOracleDetails();
            // 赔率计算
            priceToReturn =
                ammUtils.calculateOdds(oraclePrice, strikePrice, timeLeftToMaturityInDays, impliedVolatilityPerAsset[key]) /
                1e2;
            // 赔率和为1
            if (position == IThalesAMM.Position.Down) {
                priceToReturn = ONE - priceToReturn;
            }
        }
    }

    /// @notice 获取市场在金钱中的算法赔率，取自JS代码https://gist.github.com/aasmith/524788/208694a9c74bb7dfcb3295d7b5fa1ecd1d662311 
    /// @param _price 资产的当前价格 
    /// @param 资产的执行价格 
    /// @param timeLeftInDays 市场何时成熟 
    /// @param volatility 资产的隐含年波动率 
    function calculateOdds(
        uint _price,
        uint strike,
        uint timeLeftInDays,
        uint volatility
    ) public view returns (uint result) {
        uint vt = ((volatility / (100)) * (sqrt(timeLeftInDays / (365)))) / (1e9);
        bool direction = strike >= _price;
        uint lnBase = strike >= _price ? (strike * (ONE)) / (_price) : (_price * (ONE)) / (strike);
        uint d1 = (PRBMathUD60x18.ln(lnBase) * (ONE)) / (vt);
        uint y = (ONE * (ONE)) / (ONE + ((d1 * (2316419)) / (1e7)));
        uint d2 = (d1 * (d1)) / (2) / (ONE);
        if (d2 < 130 * ONE) {
            uint z = (_expneg(d2) * (3989423)) / (1e7);

            uint y5 = (powerInt(y, 5) * (1330274)) / (1e6);
            uint y4 = (powerInt(y, 4) * (1821256)) / (1e6);
            uint y3 = (powerInt(y, 3) * (1781478)) / (1e6);
            uint y2 = (powerInt(y, 2) * (356538)) / (1e6);
            uint y1 = (y * (3193815)) / (1e7);
            uint x1 = y5 + (y3) + (y1) - (y4) - (y2);
            uint x = ONE - ((z * (x1)) / (ONE));
            result = ONE * (1e2) - (x * (1e2));
            if (direction) {
                return result;
            } else {
                return ONE * (1e2) - result;
            }
        } else {
            result = direction ? 0 : ONE * 1e2;
        }
    }

    function _getMintableAmount(
        address market,
        IThalesAMM.Position position,
        uint amount
    ) internal view returns (uint mintable) {
        uint availableInContract = ammUtils.balanceOfPositionOnMarket(market, position, liquidityPool.getMarketPool(market));
        if (availableInContract < amount) {
            mintable = amount - availableInContract;
        }
    }
```

