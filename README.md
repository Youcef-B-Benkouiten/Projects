Forecast Error Modeling for Prediction Markets

Tools:Python, pandas, NumPy, Matplotlib  

What it is
This project looks at how accurate weather forecasts are and how their probabilities compare with what the market expects.  
It’s a hands-on data pipeline and analysis project that avoids common mistakes like look-ahead bias.

Why I did it
Prediction markets and trading rely on good probabilities.  
I wanted to see where forecasts are biased or uncertain and how that changes over time.

How it works
1. Grab real-time forecasts from NOAA/NWS.  
2. Match each forecast to what actually happened.  
3. Measure bias and uncertainty at different lead times.  
4. Compare model probabilities with market-implied probabilities.  
5. Make sure the data is causal — no cheating by looking ahead.

What it produces
- Bias and error metrics over time  
- Calibration charts comparing forecast vs market probabilities  
- Modular Python scripts for:
  - Data collection (`collect_forecasts.py`)  
  - Data matching (`match_outcomes.py`)  
  - Analysis and plotting (`analysis.py`)

How to run
1. Install packages:
```bash
pip install pandas numpy matplotlib
