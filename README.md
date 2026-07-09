# yand-mvsk-tailrisk
YAND-MVSK: Beyond standard YAND—explicit MVSK optimization for tail-risk hedging. Designed for extreme market scenarios: maximize upside asymmetry, suppress heavy tails. No tensor storage. 800+ assets in ~0.05s.
YAND-MVSK-Behavioral

Portfolio Optimization with Tail Risk Control & Behavioral Finance Calibration

---

📖 Introduction

Markowitz gave us mean-variance optimization. But financial returns don't follow Gaussian distributions—they have fat tails and asymmetry. The 2008 crash, the COVID drop, the meme stock spikes—these are tail events that traditional optimizers simply ignore.

YAND-MVSK-Behavioral builds on the YAND (Yau's Affine-Normal Descent) algorithm developed by Fields Medalist Shing-Tung Yau's team. It optimizes across all four statistical moments in a single convex optimization: maximize return, minimize variance, maximize skewness (prefer upside), minimize kurtosis (avoid tail events).

But this project goes one step further: it integrates behavioral finance into the risk calibration process. Because what investors say about their risk tolerance and what they actually do when markets crash are two very different things.

---

🎯 Key Features

· Four-Moment Optimization: Optimizes mean, variance, skewness, and kurtosis simultaneously—not just the first two moments.
· Lightning-Fast Solver: Never builds O(n³) coskewness tensors or O(n⁴) cokurtosis tensors. Stores only the T×n return matrix and computes everything through matrix-vector products. 800+ assets solve in ~0.05s on a laptop in 5-10 iterations.
· Behavioral Gamma Calibration: Maps client risk profiles to gamma values using a 5-question Visual Behavioral Questionnaire (VBQ-5) that uses relative anchors (instead of absolute numbers) to detect loss aversion, disposition effect, and overconfidence intuitively.
· Dynamic Risk Monitoring: Rolling-window skewness/kurtosis alerts for early warning of tail risk accumulation.
· Client-Specific Portfolios: Different gamma values for conservative (γ=12-15), moderate (γ=6-8), and aggressive (γ=2-3) investors.

---

🚀 Quick Start

Installation

```bash
pip install yand-mvsk
```

Basic Usage

```python
import yfinance as yf
from yand_mvsk import EfficientMVSK

# Download price data
prices = yf.download(["SPY", "QQQ", "TLT", "GLD"], start="2020-01-01")["Close"]

# Initialize optimizer with gamma=6 (moderate risk aversion)
ef = EfficientMVSK.from_prices(prices, gamma=6)

# Optimize and get weights
weights = ef.optimize()
cleaned = ef.clean_weights()

# View performance
ef.portfolio_performance(verbose=True)
```

With Behavioral Gamma Calibration

```python
from yand_mvsk import EfficientMVSK
from behavioral_gamma import BehavioralGammaOptimizer

# Load client's behavioral questionnaire responses (A, B, C, D)
client_answers = {
    'Q1': 'B',   # Loss aversion
    'Q2': 'A',   # Disposition effect
    'Q3': 'C',   # Overconfidence
    'Q4': 'D',   # Time horizon
    'Q5': 'C'    # Realized panic response
}

# Calculate behaviorally-calibrated gamma
bgo = BehavioralGammaOptimizer()
gamma = bgo.calculate_gamma(client_answers)  # Returns e.g., 9.6

# Run optimization with calibrated gamma
ef = EfficientMVSK.from_prices(prices, gamma=gamma)
weights = ef.optimize()
```

---

🧠 Behavioral Finance Integration (VBQ-5)

Traditional risk questionnaires ask "How much loss can you tolerate?"—but people are terrible at predicting their own behavior under stress. Worse, asking about absolute percentages (e.g., "15% drop") is flawed because 15% for a bank stock is a crash, while 15% for a tech stock is a Tuesday.

YAND-MVSK-Behavioral uses a Visual Behavioral Questionnaire (VBQ-5) with relative anchors and scenario contrasts to detect three key cognitive biases intuitively:

The VBQ-5 Questionnaire

Question Bias Detected Gamma Adjustment
Q1. Loss Aversion Market crashes, but your stock drops 10% more than the market (e.g., market -5%, your stock -15%). What do you do? 🟢 A. Buy more ("It's a discount!") 🟡 B. Hold and wait 🔴 C. Sell/Cut losses Loss Aversion C → γ × 1.4 B → γ × 1.1
Q2. Disposition Effect You need cash urgently and must sell one stock: A. Stock A: +20% (winner) B. Stock B: -20% (loser) Which do you sell? Disposition Effect A (Sell winner) → γ + 2.0 (hard adjustment)
Q3. Overconfidence Predict the S&P 500 / CSI 300 index level for next year. You are 80% confident it will fall within which range? 🟢 A. Very wide (±30%+) 🟡 B. Medium (±15%) 🔴 C. Very narrow (±5%) Overconfidence C → γ × 1.5
Q4. Time Horizon How long until you need this money? 🟢 A. 10+ years 🟡 B. 3–10 years 🔴 C. 1–3 years ⚫ D. < 1 year Time Horizon D → γ × 1.6 C → γ × 1.3 B → γ × 1.1
Q5. Realized Panic Recall a time your portfolio dropped a significant amount (e.g., 10-20% of total assets). What did you actually do? 🟢 A. Held or bought more 🟡 B. Anxious but did nothing 🔴 C. Panic-sold near the bottom Realized Risk Response C → γ × 1.6

Gamma Mapping Code

```python
def calculate_behavioral_gamma(answers):
    """
    answers: dict with keys 'Q1'~'Q5', values are 'A', 'B', 'C', or 'D'
    """
    # 1. Base gamma (from traditional questionnaire, default to 6 here)
    base_gamma = 6  
    
    # 2. Behavioral adjustment multiplier
    adjust = 1.0
    
    # Q1: Loss aversion - Sell/Cut losses means extreme fear
    if answers.get('Q1') == 'C':  
        adjust *= 1.4  
    elif answers.get('Q1') == 'B':  
        adjust *= 1.1  
    
    # Q2: Disposition effect - Selling winners while holding losers
    if answers.get('Q2') == 'A':  
        adjust += 2.0  # Hard addition, not multiplication
    
    # Q3: Overconfidence - Narrow prediction range
    if answers.get('Q3') == 'C':  
        adjust *= 1.5  
    
    # Q4: Time horizon - Shorter horizon = higher gamma
    if answers.get('Q4') == 'D':  
        adjust *= 1.6
    elif answers.get('Q4') == 'C':  
        adjust *= 1.3
    elif answers.get('Q4') == 'B':  
        adjust *= 1.1
    
    # Q5: Realized panic - Past panic selling behavior
    if answers.get('Q5') == 'C':  
        adjust *= 1.6  
    
    final_gamma = base_gamma * adjust
    return min(final_gamma, 20)  # Cap at 20
```

Client Profile Gamma Range Description
Conservative 12 – 15 Retirees, capital preservation focus
Moderate 6 – 8 Most retail investors
Aggressive 2 – 3 Young investors with long horizons

---

📊 Risk Management Workflow

1. Pre-Trade: Set Risk Budget

Set gamma and position limits before constructing the portfolio.

2. During-Trade: Rolling Monitoring

Calculate rolling 60-day skewness and kurtosis weekly:

```python
port_returns = returns_60d @ current_weights
skew = port_returns.skew()
kurt = port_returns.kurtosis()

if skew < -0.5 and kurt > 5:
    send_alert("Warning: Negative skew + fat tail detected!")
```

3. Stress Testing

Test portfolio against historical crash periods (e.g., March 2020, 2022 bear market).

4. Rebalancing Trigger

When rolling kurtosis exceeds 2× historical mean, re-optimize with higher gamma.

---

🏗️ Project Structure

```
YAND-MVSK-Behavioral/
├── src/
│   ├── optimizer/
│   │   └── mvsk_optimizer.py      # YAND-MVSK wrapper
│   ├── behavioral/
│   │   ├── gamma_calibrator.py    # VBQ-5 gamma calculator
│   │   └── questionnaire.py       # VBQ-5 implementation
│   └── risk/
│       ├── monitor.py             # Rolling skewness/kurtosis
│       └── stress_test.py         # Historical scenario testing
├── examples/
│   ├── basic_usage.ipynb
│   ├── behavioral_calibration.ipynb
│   └── dynamic_rebalancing.ipynb
├── tests/
├── README.md
└── requirements.txt
```

---

⚠️ Limitations & Caveats

· Noise Amplification: Higher moments (especially kurtosis) are extremely sensitive to outliers. What you compute from historical data may be 90% random noise.
· Frequency Mismatch: Using high-frequency data to capture tail risk, then holding positions for 18 months, requires careful window selection.
· Not a Silver Bullet: Use this tool as a risk-aware complement to, not a replacement for, fundamental investment judgment.

---

📚 References

· Wang, Zhou, Ying, and Palomar (2022). High-Order Portfolio Optimization
· Yau's Affine-Normal Descent (YAND) algorithm
· Boudt, Cornilly, Van Holle and Willems (2020). MVSK Portfolio Tilting

---

📄 License

MIT

---

🎉 Contributing

Issues and pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

---

⭐ Star History

If you find this project useful, please consider giving it a star!
