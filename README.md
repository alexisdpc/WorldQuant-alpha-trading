# WorldQuant alpha trading

WorldQuant BRAIN is an online simulation platform to build alpha ($\alpha$) signals and perform backtesting using market data. However, it is hard to find resources and documentation on the use of this platform
We present some trading strategies with high Sharpe ratio [(see Report)](https://github.com/alexisdpc/WorldQuant-alpha-trading/blob/main/Worldquant_Report.pdf)

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



