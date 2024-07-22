# 合约交易系统设计与开发

上次在“[证券交易系统设计与开发](../2018-01-05-exchange-design/index.html)”这篇文章中，详细介绍了一个证券交易所的设计和开发思路，受到了广大人民群众的广泛肯定。

不过有很多群众反馈，说文章太简单，连炒股的大爷大妈都看得懂，能不能再深入一点？

考虑到现货交易系统确实很简单，今天我们就来实现一个合约交易系统的设计与开发。

合约交易，通常指期货合约。现货合约我们以后再讨论。这里我们仍然以数字货币的期货合约为例，实现一个基于BTC/USD价格指数的期货合约。

所谓期货交易，就是指以约定的价格在未来进行交割。

期货交易的目的原本是以当前约定的价格锁定未来某个时间段的价格，这样企业生产就可以合理地锁定采购成本，避免了价格涨跌带来的经营风险。

但是市场的风险并不会平白消失。风险其实并没有消失，而是转移到了市场的其他参与方。企业通过期货市场锁定现货价格，现货价格的波动风险实际上转移到了合约的对手方。因此，期货市场是一个高度投机的市场。期货市场的高收益源于交易方承担了高风险，而高风险也会带来巨额亏损。

期货合约与现货交易的区别是，期货合约需要交易双方以约定的“份数”进行买卖，例如黄金期货，外盘通常以“盎司”为单位进行买卖，只能买卖整数倍的合约。

在期货交易中，通常不需要支付全额费用，而只需要支付一定比例的保证金。根据保证金比例的不同，期货交易的杠杆也不同。例如，10%的保证金率就是10倍杠杆，5%的保证金率就是20倍杠杆。杠杆既能放大收益，也能放大亏损。

此外，如果不计算交易手续费，期货合约的买卖双方就是一个零和博弈，即所有交易方的盈利和亏损相加为零。

根据期货合约的特点，我们就可以设计一个期货合约交易系统的所有模块：

```ascii
                                                                           ┌───────────┐
                                                                           │    ADL    │
                                 ┌───────────────────────────────┐         └───────────┘
                                 │                               │               │
                                 ▼                               │               ▼
           ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
Request ──▶│   User    │──▶│  Account  │──▶│   Order   │   │ Clearing  │──▶│ Position  │
           └───────────┘   └───────────┘   └───────────┘   └───────────┘   └───────────┘
                                                 │               ▲               ▲
                                                 ▼               │               │
                                           ┌───────────┐   ┌───────────┐   ┌───────────┐
                                           │ Sequence  │──▶│   Match   │──▶│Liquidation│
                                           └───────────┘   └───────────┘   └───────────┘
                                                                 │
                                                                 ▼
                                                           ┌───────────┐
 Market ◀──────────────────────────────────────────────────│ Quotation │
                                                           └───────────┘
```

当系统收到一个用户的订单请求时，首先，仍然由用户系统（User）进行身份认证，然后，确定当前交易的合约是否在有效期内，紧接着，由账户系统（Account）确定是否有足够的余额作为保证金进行交易。如果由足够的保证金，该订单请求就由订单系统（Order）创建成功，并进入定序系统（Sequence）。定序后进入撮合引擎（Match），如果成交，订单由清算系统（Clearing）进行清算。

和现货交易不同的是，用户订单成交后，同时创建（或者平掉）了一个仓位（Position），所以，期货合约系统需要一个仓位管理模块（Position）来管理所有用户的仓位。

由于市场价格在实时变动，对于普通的期货交易所，通常是每日结算，收盘后要求亏损的用户补足保证金。而对于24小时交易的数字货币交易所，无法每日结算，因此需要一个爆仓引擎（Liquidation）。

爆仓引擎根据当前市场价格实时计算所有用户的仓位权益是否降至零。如果用户的仓位保证金不足，爆仓引擎就启动爆仓流程：

1. 检测用户是否设置了“自动转入保证金”，并且账户有足够余额；
2. 如果能自动转入保证金，则自动转入，权益提高，不需要爆仓；
3. 如果不能自动转入，则爆仓引擎首先接管仓位，用户对此仓位的权益归零；
4. 爆仓引擎计算用户权益为零的价格，并按照该价格平仓。平仓是否成功取决于市场波动性和流动性。

如果市场缺乏足够的流动性，导致爆仓引擎所持有仓位无法及时平仓，交易所本身就会承受损失。在某个期货合约周期内，交易所可以选择所有盈利用户共同分担损失，也可以优先选择高风险盈利用户强制减仓来提高流动性。

如果选择强制减仓，系统还需要一个自动减仓系统（ADL：Auto-Deleveraging），自动减仓系统根据用户风险和盈利高低对盈利仓位排序，排在靠前的仓位更有可能被强制减仓。

合约到期后，由清算系统（Clearing）对所有仓位进行交割，按照盈亏更新账户（Account），然后平掉所有仓位，合约终止。

下一步是编写代码来实现：

```java
/**
 * A crypto futures exchange.
 * 
 * @author liaoxuefeng
 */
public class CryptoFuturesExchangeApplication {
    public static void main(String[] args) {
        // TODO:
    }
}
```

完善代码后，一个期货合约交易系统就设计实现成功！