# Order Flow Imbalance & Market Depth — SPY

Implementation of the price impact model from Cont, Kukanov & Stoikov (2014) on S&P 500 tick-by-tick data.

---

## The Model

The central question : does the imbalance between buying and selling pressure at the best quote predict short-term price changes ?

$$\Delta p_t = \alpha + \beta \cdot OFI_t + \epsilon_t$$

**OFI** (Order Flow Imbalance) measures the net variation of volume at the best bid and ask between two timestamps :

$$OFI_t = \Delta B_t - \Delta A_t$$

$$\Delta B_t = q^b_t \cdot \mathbf{1}_{p^b_t \geq p^b_{t-1}} - q^b_{t-1} \cdot \mathbf{1}_{p^b_t \leq p^b_{t-1}}$$

$$\Delta A_t = q^a_t \cdot \mathbf{1}_{p^a_t \leq p^a_{t-1}} - q^a_{t-1} \cdot \mathbf{1}_{p^a_t \geq p^a_{t-1}}$$

| Parameter | Interpretation |
|-----------|---------------|
| $\beta$ | price impact per unit of OFI — Kyle's lambda estimated from the LOB |
| $1/\beta$ | market depth — net volume required to move the mid-price by one tick |
| $R^2$ | fraction of price variance explained by order flow imbalance alone |

---

## Data

- **Asset** : SPY (S&P 500 ETF — large-tick regime, spread = 1 tick)
- **Source** : Alpaca Markets API, SIP feed (consolidated across all US venues)
- **Granularity** : tick-by-tick quotes resampled to 1-minute bars
- **Sample** : one trading session, 09:30–16:00 ET

SPY is used as a liquid proxy for the S&P 500. The large-tick regime concentrates liquidity at the best price, which makes OFI a stronger signal than in fragmented or small-tick markets.

---

## Results

### Global regression (full session)

| Metric | Value |
|--------|-------|
| $\beta$ (Kyle lambda) | 0.0036 USD / share |
| Market depth $1/\beta$ | ~278 shares per tick |
| $R^2$ | 0.372 |
| t-stat (OFI) | 15.1 |
| Kurtosis (residuals) | 10.17 |
| Durbin-Watson | 2.305 |

A R² of 0.37 at 1-minute frequency is consistent with the empirical findings of Cont et al. (2014). The intercept is not statistically significant (p = 0.73), confirming the absence of systematic drift within a single session.

### OFI vs Price Change

The linear relationship is well-identified. Observations are concentrated in OFI ∈ [-20, 20]. A small number of outliers at OFI > 40 deviate slightly from the fit line, consistent with the concavity of price impact at large order imbalances documented in the literature.

The residuals are stationary but leptokurtic (kurtosis = 10.17). This is the empirical fat-tail signature of financial returns — the OLS confidence intervals are underestimated as a result. Robust standard errors (HC3) should be preferred for inference.

### Intraday beta and R²

Rolling estimates on 30-minute windows reveal three distinct intraday regimes :

**Opening (09:30–11:00)** — beta rises sharply, reaching its first peak near 11:00 (~0.0075). The book is thin, adverse selection is high, and a small order imbalance moves the price significantly.

**Midday trough (13:00–13:30)** — beta collapses to its daily minimum (~0.0017). This is the lunch effect : professional market makers reduce activity, liquidity thins out, but order flow becomes less directional and less informative simultaneously. R² reaches its minimum (~0.28) at the same time, confirming that OFI loses its predictive power precisely when market depth is lowest.

**Afternoon and close (14:00–16:00)** — beta and R² recover progressively. A second peak appears around 15:00–15:30, driven by end-of-day inventory unwinding and pre-close positioning.

This pattern deviates from the simple inverted-J profile described in Bouchaud et al. (2018). The midday collapse of both beta and R² suggests that during the lunch window, price movements are driven by factors other than order book pressure — consistent with a period dominated by noise trading and low-participation flow.

---

## Notebook Structure

```
ofi_sp500.ipynb
├── 0. Dependencies
├── 1. Data — tick-by-tick quotes via Alpaca API
├── 2. OFI computation — Cont et al. formula
├── 3. OLS regression — global price impact estimate
├── 4. Visualisation — scatter + residuals
├── 5. Rolling beta — intraday market depth
└── 6. Summary
```

---

## Setup

```bash
pip install alpaca-py statsmodels pandas numpy matplotlib
```

API keys are loaded from environment variables :

```bash
export ALPACA_API_KEY="your_key"
export ALPACA_SECRET_KEY="your_secret"
```

---

## Limitations

**Single session.** Results cover one trading day. Beta and R² are sensitive to macro announcements (FOMC, NFP) — sessions around these events show structurally higher adverse selection and thinner books.

**Linearity assumption.** The model assumes a linear price impact. Empirical evidence (Taranto et al., 2018) shows concavity at large imbalances — the linear model underestimates impact in the tails.

**SPY vs ES futures.** SPY trades on equity venues with full price-time priority. ES futures (CME) operate under a partial pro-rata allocation rule with a different participant mix. The estimated beta is not directly transferable to futures markets.

**Residual fat tails.** Kurtosis = 10.17 violates the normality assumption of OLS. Use `cov_type="HC3"` for robust inference.

---

## References

- Cont, Kukanov & Stoikov — *The Price Impact of Order Book Events*, Review of Finance (2014)
- Bouchaud, Bonart, Donier & Gould — *Trades, Quotes and Prices*, Cambridge University Press (2018)
- Kyle — *Continuous Auctions and Insider Trading*, Econometrica (1985)
- Taranto, Bormetti, Bouchaud, Lillo & Toth — *Linear models for the impact of order flow on prices*, Quantitative Finance (2018)
