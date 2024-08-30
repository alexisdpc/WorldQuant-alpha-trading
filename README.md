# WorldQuant alpha trading

WorldQuant BRAIN is an online simulation platform to build alpha ($\alpha$) signals and perform backtesting using market data. However, it is hard to find resources and documentation on the use of this platform.
We present some trading strategies with a high Sharpe ratio
.
## American market

▶ A simple Price Reversion alpha:

```
SMA_30 = ts_mean(close,30);
rank(SMA_30 - close)
```

with the following properties:
```
Region: USA 
Universe: Top3000
Delay: 1
Neutralization: Subindustry
Decay: 4
Truncation: 0.08
Pasteurization: Off
Unit Handling: Verify
NaN Handling: Off
```


▶ Trade based on volume and price difference. If the volume is larger than the 20-day average, then trade based on the price difference of the last 5 days. 

```
event = volume>adv20;
alpha = (-ts_delta(close,5));
trade_when(event,alpha,-1)
```
with the following properties:
```
Region: USA 
Universe: Top3000
Delay: 1
Neutralization: Subindustry
Decay: 2
Truncation: 0.01
Pasteurization: Off
Unit Handling: Verify
NaN Handling: Off
```

Results from the price weighted average alpha:
![results_1](https://github.com/alexisdpc/WorldQuant-alpha-trading/assets/124795834/a733fc99-f811-4c38-b5ab-de0166676607)

▶ Price weighted average. vwap is the daily volume weighted average price. The following alpha uses vwap to implement Price Reversion:
```
(vwap-close)/vwap
```
```
Region: USA 
Universe: Top3000
Delay: 1
Neutralization: Subindustry
Decay: 15
Truncation: 0.08
Pasteurization:  On
Unit Handling: Verify
NaN Handling: Off
```

▶  Operating income 

Usually the most important income is the one from operation, since this means the operating efficiency is good. In this alpha,  we take long positions on companies with higher operating income and we take short positions on companies with low operating income. 

We divide by market cap to account for the size of the companies in order to make a fair comparison
$$\text{Operating yield} = \frac{\text{Operating income}}{\text{Market cap}}$$
Moreover, the cashflow from operations is better since it takes into account expenses and operating costs. We rank the operational cashflow divided by capitalization over the last 63 days.
```
alpha = ts_rank(cashflow_op/cap,60);
group_rank(alpha, subindustry)
```
```
Region: USA 
Universe: Top3000
Delay: 1
Neutralization: Subindustry
Decay: 4
Truncation: 0.08
Pasteurization:  On
Unit Handling: Verify
NaN Handling: On
```

## Chinese market

▶  Volatility of daily turnover in last 20 days.\
• `mdl175_volatility`: Volatility of daily turnover during the last 20 days.

```
rank(-mdl175_volatility*log(volume))
    *(1+group_rank(mdl175_revenuettm, sector))
```
with the following properties:
```
Region: CHN 
Universe: Top3000
Delay: 0
Neutralization: Sector
Decay: 3
Truncation: 0.08
Pasteurization:  On
Unit Handling: Verify
NaN Handling: Off
```

## Improving alphas

▶  **Turnover**: Turnover of an alpha is a metric that measures the daily trading activity
$${\rm Turnover} = \frac{\text{Dollar Trading Value}}{\text{ Booksize }} = \frac{\text{Value Traded}}{\text{Value Held}} $$

• Increase Decay $\implies$ Reduces the turnover\
• Reducing turnover. In order to reduce turnover we can add a condition on the trading.
Instead of trading at each market open, it will trade only when event is True (where adv20
corresponds to the average daily volume in past 20 days):
```
event = volume>adv20;
alpha = news_cap;
trade_when(event, alpha, -1)
```

▶  **Fitness**: Fitness of an alpha is a function of Returns, Turnover \& Sharpe. Fitness is defined as:
$$\text{Fitness} = \text{Sharpe} \sqrt{\frac{|{\rm returns}|  }{ {\rm max}[{\rm turnover},0.125] }}$$
Good alphas generally have high fitness. You can seek to improve the performance of your alphas by increasing Sharpe (or returns) and reducing turnover. The passing requirement for fitness on the BRAIN platform is to be greater than 1.0.





