# 交易系统如何确保账簿100%准确

交易系统中，对账是一个大问题。对账处理不好，不但需要花费大量的人力去处理账簿，还要承担很大的线上修改账簿的风险。

如果系统能自动化保证账簿每时每刻100%准确，不能说一劳永逸地解决了所有问题，至少解决了绝大部分问题。

如何对账，能时刻确保账簿100%准确？

交易系统中，用户余额存储在账户表中。例如，一个比特币交易系统，假设用户有BTC和USD两种资产，要确保账簿余额是对的，一个设计的关键点就是：时刻保证整个系统的资产负债表为零！

设想账户表的初始状态，用户A、B、C、D分别存入了数量不等的BTC和USD：

| user | currency | balance |
|------|----------|--------:|
| A | BTC | 1.2  |
| B | USD | 4000 |
| C | BTC | 2.8  |
| D | USD | 6000 |

无论这些用户如何交易，以什么价位交易，最终的结果只是BTC和USD在账户之间移动，其总额并不会增减：

| user | currency | balance |
|------|----------|--------:|
| A | BTC | 0.2  |
| A | USD | 3000 |
| B | BTC | 1.0  |
| B | USD | 1000 |
| C | BTC | 0.8  |
| C | USD | 6000 |
| D | BTC | 2.0  |

把交易手续费考虑进来，其实也是一样的，只是用户的资产有很少一部分被移动到了系统手续费账户中：

| user | currency | balance |
|------|----------|--------:|
| A | BTC | 0.2  |
| A | USD | 2997 |
| B | BTC | 1.0  |
| B | USD | 1000 |
| C | BTC | 0.8  |
| C | USD | 5994 |
| D | BTC | 2.0  |
| FEE | USD | 9  |

上面的账户表不好对账，是因为根据“资产负债表总额始终为零”这一基本原理，缺少一个关键的“负债”账户。

如果把初始状态加入负债账户：

| user | currency | balance |
|------|----------|--------:|
| DEBT | BTC | -4    |
| DEBT | USD | -10000 |
| A | BTC | 1.2  |
| B | USD | 4000 |
| C | BTC | 2.8  |
| D | USD | 6000 |

整个账户表的余额加起来就一定为零。

随着交易的进行，无论资产如何转移，最终的账户余额也一定为零：

| user | currency | balance |
|------|----------|--------:|
| DEBT | BTC | -4    |
| DEBT | USD | -10000 |
| A | BTC | 0.2  |
| A | USD | 2997 |
| B | BTC | 1.0  |
| B | USD | 1000 |
| C | BTC | 0.8  |
| C | USD | 5994 |
| D | BTC | 2.0  |
| FEE | USD | 9  |

因此，可以设计出对账的基本逻辑：

```sql
SELECT SUM(balance) FROM ACCOUNTS GROUP BY currency;
```

在每笔交易后执行该SQL语句，如果账簿无误，结果集必定每一行均为`0`。

考虑到对账程序的执行效率，可以把它放到只读从库执行，不影响交易本身。

更进一步，考虑系统初始状态的账户表：

| user | currency | balance |
|------|----------|--------|
| (无记录) | (无记录) | (无记录) |

当用户X存入USD 1500后，要保证资产负载表为`0`，该存款操作本质上是一个资产转移过程：USD 1500从DEBT账户转移到用户X的账户：

| user | currency | balance |
|------|----------|--------:|
| DEBT | USD | -1500 |
| X    | USD | 1500  |

取款过程则恰好相反，它是用户资产转移到DEBT账户的过程，体现为DEBT账户余额增加（从负更多到负更少）。

对于财务人员来讲，用户总资产其实就是`ABS(DEBT) - FEE`，计算极其简单。

可见，一个清晰简洁而可靠的设计，不但有力地保证了系统的安全性，而且大大降低了财务对账成本。