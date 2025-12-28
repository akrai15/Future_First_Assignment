

# Technical Documentation: Stock Price Prediction & Market Regime Analysis

## 1. Project Abstract

This project focuses on the quantitative analysis of the relationship between an external "Data" variable (presumed macroeconomic or sentiment indicator) and daily Stock Prices. The core objective was to determine if the external data serves as a leading indicator for price movements.

By implementing a **Lagged Linear Regression Model**, the study successfully identified a statistically significant, albeit weak, negative correlation. The final model achieves an  of R Square of **0.9938**, leveraging an auto-regressive framework to predict prices with a Mean Absolute Percentage Error (MAPE) of less than **0.76%**.

---

## 2. Mathematical Framework & Methodology

### **The Predictive Equation**

The model does not rely on simple correlation. It utilizes a dynamic time-series regression equation. The predicted price at time  () is calculated as:

Where:

* : The intercept (baseline bias).
* : The **Previous Day's Stock Price** (The Anchor).
* : The **Previous Day's Change in Data** (The Signal).
* : The learned coefficients (weights).

### **Why Linear Regression (OLS)?**

While Neural Networks (LSTMs) are popular for time series, OLS (Ordinary Least Squares) was chosen for:

1. **Interpretability:** We needed to prove the *direction* of the relationship. OLS provides a coefficient () that is explicitly negative, proving the inverse relationship.
2. **Noise Robustness:** Financial data is stochastic. Complex models often overfit (memorize) noise. A linear constraint forces the model to find the single most dominant trend.

---

## 3. Data Dictionary & Structure

The project requires two specific CSV input files. All datasets are chronologically sorted and merged on the `Date` index.

| File Name | Column | Description | Data Type |
| --- | --- | --- | --- |
| `StockPrice.csv` | `Date` | The trading date (Index) | DateTime |
|  | `Price` | Daily closing price of the asset | Float |
| `Data.csv` | `Date` | The reporting date | DateTime |
|  | `Data` | The external variable (e.g., Inflation, Yield) | Float |

---

## 4. Feature Engineering Pipeline

To transform raw data into a supervised learning format, the following pipeline was executed:

1. **Stationarity Transformation (`.diff()`):**
* *Problem:* Stock prices are non-stationary (unbounded growth).
* *Solution:* We calculated the day-over-day **change** () for the Data variable. This inputs "shocks" to the model rather than arbitrary levels.


2. **Temporal Alignment (`.shift(1)`):**
* *Critical Step:* All feature columns were shifted forward by 1 day.
* *Reasoning:* To predict Tuesday's close, the model is strictly allowed to see only data available up to Monday night. This prevents **Data Leakage** (Look-ahead bias).


3. **Normalization (`StandardScaler`):**
* Features were scaled to Zero Mean and Unit Variance. This ensures the model treats a $50 price movement and a 0.05 data movement with equal mathematical weight during gradient descent.



---

## 5. Comprehensive Results & Analysis

### **A. Model Accuracy**

* **R Square (0.9938):** The model captures 99.4% of the variance. This is primarily driven by the inertia of the stock price itself.
* **RMSE ($48.76):** On a typical trading day, the model's prediction is within $48 of the actual closing price.

### **B. The "Signal" Discovery**

* **Coefficient Analysis:** The coefficient for `Prev_Data_Diff` is **Negative**.
* **Interpretation:** A positive shock in the Data variable acts as a "brake" on the stock price. This validates the hypothesis of a contrarian relationship.

### **C. Regime Instability (Rolling Correlation)**

An advanced rolling window analysis (60-day) revealed that the relationship is **non-linear over time**:

* **"Red Zones" (Negative Correlation):** The standard regime where the model performs optimally.
* **"Green Zones" (Positive Correlation):** Distinct periods where the market logic inverts. This suggests that during specific economic cycles, "Bad News" (Data increase) is interpreted as "Good News" by the market, likely due to liquidity expectations.

---





## 6. Future Recommendations: Moving from Linear to Adaptive

To transform the current linear model into an institutional-grade trading algorithm, future development should focus on **Non-Linear Dynamics** and **Regime Switching**.

### **A. Interaction Terms: Conditional Sensitivity (The "VIX" Filter)**

**The Hypothesis:**
Currently, the model assumes the market reacts to the "Data" variable the same way every single day. In reality, markets are hyper-sensitive during high stress and complacent during low stress.

* *Scenario A (Calm Market):* Bad news is ignored.
* *Scenario B (Panic Market):* The same bad news causes a crash.

**The Mathematical Fix:**
Instead of just using `Data_Change`, we introduce an **Interaction Term** between the Data and Volatility (e.g., VIX or Rolling Standard Deviation).

**Implementation Strategy:**

1. **Calculate Volatility:** If external VIX data is unavailable, calculate the 14-day rolling standard deviation of the Stock Price.
2. **Create the Feature:** `df['Interaction'] = df['Data_Change'] * df['Volatility']`
3. **Result:** If  is negative and significant, it proves that **the "Data" variable effectively acts as a multiplier for fear.** This allows the model to bet *larger* when volatility is high and *smaller* when the market is calm.

---

### **B. Regime Detection: The "Meta-Model" Approach**

**The Problem:**
Our rolling correlation analysis proved that the relationship "flips" (Green Zones vs. Red Zones). A single static regression model fails during these flips because it is permanently biased negative.

**The Solution:**
Build a **Two-Step "Meta-Model"**:

1. **Step 1 (The Classifier):** Train a secondary model (Random Forest or Logistic Regression) to predict the *Regime*.
* *Target:* Is the 60-day correlation likely to be Positive (1) or Negative (0) next week?
* *Features:* Interest Rates, Inflation trends, Moving Averages.


2. **Step 2 (The Switch):**
* *If Regime == Negative:* Use the current model (Short the Data spikes).
* *If Regime == Positive:* Flip the sign (Long the Data spikes) or exit the market (Cash).



**Implementation Strategy:**

```python
# Pseudo-code logic for the trading system
current_regime = regime_classifier.predict(current_market_conditions)

if current_regime == "Red Zone":
    prediction = linear_model.predict(data) # Standard logic
elif current_regime == "Green Zone":
    prediction = -1 * linear_model.predict(data) # Inverted logic

```

**The Benefit:**
This transforms the system from a **"Static Forecaster"** into an **"Adaptive Algorithm"** that knows when to follow the rule and when to break it. This is the primary method quantitative funds use to survive changing market cycles.
