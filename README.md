# WorldQuant alpha trading

This is a guide to use the WorldQuant BRAIN paltform. We present some trading strategies with high Sharpe ratio [(see Report)](https://github.com/alexisdpc/WorldQuant-alpha-trading/blob/main/Worldquant_Report.pdf)

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

