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

