---
name: ali-actuarial
description: |
  Actuarial science fundamentals for insurance analytics and reserving. Use when:

  PLANNING: Designing reserve models, planning development factor methodology, architecting
  loss projection systems, choosing credibility approaches, evaluating aggregate loss models

  IMPLEMENTATION: Building loss development triangles, calculating IBNR, implementing
  Bornhuetter-Ferguson method, coding credibility models, fitting severity distributions,
  writing reserve automation, generating NAIC Annual Statement exhibits

  GUIDANCE: Asking about chain ladder vs. Cape Cod, when to use paid vs. incurred triangles,
  how to select development factors, credibility theory applications, loss distribution fitting,
  ASOP 43 requirements, tail factor estimation methods

  REVIEW: Validating reserve calculations, checking development factor selection, auditing
  IBNR projections, reviewing ultimate loss estimates, verifying regulatory compliance (SAP/GAAP)
---

# Actuarial Science Fundamentals

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing reserve estimation models or data pipelines
- Planning development factor calculation methodology
- Architecting aggregate loss projection systems
- Choosing between reserving methods (chain ladder, BF, Cape Cod)
- Evaluating credibility-weighted estimation approaches
- Planning regulatory reporting automation (NAIC Annual Statement)

**Implementation:**
- Building loss development triangles (paid and incurred)
- Calculating IBNR (Incurred But Not Reported) reserves
- Implementing chain ladder, Bornhuetter-Ferguson, or Cape Cod methods
- Coding credibility models (Buhlmann, Bayesian)
- Fitting severity distributions (Pareto, lognormal, Weibull)
- Writing frequency-severity aggregate models
- Generating Schedule P exhibits or other regulatory schedules
- Building reserve automation in Python or SQL

**Guidance/Best Practices:**
- Asking about development factor selection criteria
- Asking when to use paid vs. incurred triangles
- Asking how to estimate tail factors
- Credibility theory applications and Z-factor calculation
- Loss distribution fitting and goodness-of-fit tests
- Trend analysis for frequency and severity
- ASOP 43 (actuarial standards) compliance
- SAP vs. GAAP accounting differences

**Review/Validation:**
- Validating reserve calculations for accuracy
- Checking development factor selection reasonableness
- Auditing IBNR projections against historical patterns
- Reviewing ultimate loss estimates
- Verifying regulatory compliance (NAIC, SAP, GAAP)
- Checking for common actuarial mistakes (mixing periods, inappropriate tails)

---

## Key Principles

- **Triangle integrity**: Accident year, policy year, and calendar year must align consistently
- **Development patterns are key**: Past claims behavior predicts future development
- **Paid vs. incurred trade-off**: Paid is more stable, incurred is more responsive
- **Tail factors matter**: Long-tail lines (medical malpractice) need careful tail selection
- **Credibility is power**: Blend high-variance data with stable priors for better estimates
- **Distribution matters**: Severity modeling requires understanding fat tails (Pareto, not normal)
- **ASOP 43 compliance**: Document assumptions, methods, and sensitivity analysis
- **SAP vs. GAAP**: Statutory accounting is more conservative (no discounting, different admissibility)
- **Regulatory alignment**: NAIC Annual Statement Schedule P drives reserve disclosure
- **Validation always**: Actual-to-expected analysis catches model drift

---

## Core Reserving Methods

### Loss Development Triangles

Fundamental data structure for reserving. Organizes claims by accident year (rows) and development period (columns).

**Triangle Types:**
- **Paid triangle**: Cumulative paid losses by development period
- **Incurred triangle**: Cumulative incurred losses (paid + case reserves)
- **Case reserve triangle**: Case reserves only
- **Claim count triangle**: Number of claims reported

**Paid vs. Incurred:**
| Aspect | Paid Triangle | Incurred Triangle |
|--------|---------------|-------------------|
| Stability | More stable, less volatile | More responsive to changes |
| Speed | Slower recognition | Faster recognition |
| Reserve quality | Independent of case reserves | Depends on case reserve adequacy |
| Long-tail lines | Often preferred | May show artificial patterns |

**Accident Year vs. Policy Year vs. Calendar Year:**
```
Accident Year (AY): Year loss occurred
Policy Year (PY): Year policy was written
Calendar Year (CY): Year payment made

Medical malpractice: Often policy year for claims-made policies
Workers compensation: Always accident year
```

**Python Implementation:**
```python
import pandas as pd
import numpy as np

def build_triangle(df, origin_col='accident_year', dev_col='dev_period', value_col='paid'):
    """Build development triangle from transactional data."""
    triangle = df.pivot_table(
        values=value_col,
        index=origin_col,
        columns=dev_col,
        aggfunc='sum',
        fill_value=0
    )
    # Convert to cumulative
    triangle = triangle.cumsum(axis=1)
    return triangle

# Example usage
# df has columns: accident_year, dev_period, paid
paid_triangle = build_triangle(df, value_col='paid')
incurred_triangle = build_triangle(df, value_col='incurred')
```

### Chain Ladder Method

**Purpose:** Project ultimate losses from historical development patterns

**Steps:**
1. Calculate age-to-age factors (link ratios)
2. Select development factors
3. Calculate cumulative development factors (CDFs)
4. Project ultimate losses
5. Calculate IBNR

**Age-to-Age Factors (Link Ratios):**
```python
def calculate_age_to_age_factors(triangle):
    """Calculate development factors from triangle."""
    factors = triangle.iloc[:, 1:].values / triangle.iloc[:, :-1].values
    factors_df = pd.DataFrame(
        factors,
        index=triangle.index,
        columns=[f"{triangle.columns[i]}-{triangle.columns[i+1]}"
                 for i in range(len(triangle.columns)-1)]
    )
    return factors_df

# Volume-weighted average (typical)
def volume_weighted_average(triangle, factors_df):
    """Calculate volume-weighted average development factors."""
    selected_factors = []
    for col in factors_df.columns:
        # Get development period
        dev = int(col.split('-')[0])
        # Sum across accident years for denominator
        denominator = triangle[dev].sum()
        numerator = (triangle[dev] * factors_df[col]).sum()
        selected_factors.append(numerator / denominator)
    return np.array(selected_factors)
```

**Development Factor Selection Methods:**
| Method | Formula | Use Case |
|--------|---------|----------|
| Simple Average | mean(factors) | Stable patterns, similar volume |
| Volume-Weighted | Σ(prior×factor) / Σ(prior) | Varying volume by year |
| Medial Average | Exclude high/low, then average | Outlier years |
| Geometric Average | (∏ factors)^(1/n) | Exponential trends |
| Regression | Fit curve to historical | Systematic trend detection |

**Cumulative Development Factors (CDFs):**
```python
def calculate_cdfs(selected_factors):
    """Convert selected factors to cumulative development factors."""
    # Work backwards from tail to earliest
    cdfs = np.ones(len(selected_factors) + 1)
    for i in range(len(selected_factors) - 1, -1, -1):
        cdfs[i] = cdfs[i + 1] * selected_factors[i]
    return cdfs

def project_ultimate(triangle, cdfs):
    """Project ultimate losses using CDFs."""
    ultimate = triangle.copy()
    latest_diagonal = np.diag(np.fliplr(triangle.values))

    projected_ultimate = pd.Series(
        latest_diagonal * cdfs[:len(latest_diagonal)],
        index=triangle.index
    )
    return projected_ultimate

def calculate_ibnr(triangle, ultimate):
    """Calculate IBNR as ultimate minus latest."""
    latest_diagonal = np.diag(np.fliplr(triangle.values))
    ibnr = ultimate - latest_diagonal
    return pd.Series(ibnr, index=triangle.index)
```

**Using chainladder-python Library:**
```python
import chainladder as cl

# Load triangle
triangle = cl.Triangle(
    df,
    origin='accident_year',
    development='dev_period',
    columns='paid',
    cumulative=True
)

# Chain ladder method
cl_model = cl.Chainladder()
cl_model.fit(triangle)

# Get ultimate losses and IBNR
ultimate = cl_model.ultimate_
ibnr = cl_model.ibnr_

# Get development factors
ldfs = cl_model.ldf_
```

### Bornhuetter-Ferguson Method

**Purpose:** Blend expected losses with actual experience for immature years

**Formula:**
```
Ultimate = Paid + (Expected × % Unreported)
where % Unreported = (1 - CDF_inverse)
```

**When to Use:**
- Recent accident years with little development
- High variability in chain ladder estimates
- New lines of business with limited history
- Unstable development patterns

**Implementation:**
```python
def bornhuetter_ferguson(triangle, cdfs, expected_lr, premium):
    """Bornhuetter-Ferguson ultimate loss estimate."""
    ultimate = pd.Series(index=triangle.index, dtype=float)

    for ay in triangle.index:
        # Latest reported value
        age = len(triangle.loc[ay].dropna())
        latest = triangle.loc[ay, age - 1]

        # Percent unreported
        pct_unreported = 1 - (1 / cdfs[age - 1])

        # Expected ultimate
        expected_ult = premium[ay] * expected_lr

        # BF formula
        ultimate[ay] = latest + (expected_ult * pct_unreported)

    return ultimate

# Example
expected_loss_ratio = 0.65  # From pricing or experience
premium = pd.Series([1000000, 1100000, 1200000], index=[2022, 2023, 2024])
bf_ultimate = bornhuetter_ferguson(paid_triangle, cdfs, expected_loss_ratio, premium)
```

**Bornhuetter-Ferguson Strengths:**
- Stabilizes estimates for immature years
- Reduces development factor volatility impact
- Incorporates pricing assumptions

**Bornhuetter-Ferguson Weaknesses:**
- Requires accurate expected loss ratio
- May not respond quickly to emerging trends
- Heavy reliance on a priori assumptions

### Cape Cod Method (Stanard-Buhlmann)

**Purpose:** Iteratively estimate expected loss ratio from the data itself

**Advantage over BF:** No need to specify expected loss ratio a priori

**Formula:**
```
LR = Σ(Reported) / Σ(Premium × % Reported)
Ultimate_ay = Reported_ay + (Premium_ay × LR × % Unreported_ay)
```

**Implementation:**
```python
def cape_cod(triangle, cdfs, premium):
    """Cape Cod (Stanard-Buhlmann) ultimate loss estimate."""
    # Calculate reported percentage by age
    reported_pcts = 1 / cdfs

    # Estimate loss ratio
    total_reported = 0
    total_exposure = 0

    for ay in triangle.index:
        age = len(triangle.loc[ay].dropna())
        latest = triangle.loc[ay, age - 1]
        total_reported += latest
        total_exposure += premium[ay] * reported_pcts[age - 1]

    estimated_lr = total_reported / total_exposure

    # Calculate ultimate for each AY
    ultimate = pd.Series(index=triangle.index, dtype=float)
    for ay in triangle.index:
        age = len(triangle.loc[ay].dropna())
        latest = triangle.loc[ay, age - 1]
        pct_unreported = 1 - reported_pcts[age - 1]
        ultimate[ay] = latest + (premium[ay] * estimated_lr * pct_unreported)

    return ultimate, estimated_lr

# Using chainladder-python
cc_model = cl.CapeCod()
cc_model.fit(triangle, sample_weight=premium)
ultimate_cc = cc_model.ultimate_
```

**When to Use Cape Cod:**
- More credible than simple BF for heterogeneous book
- Volume varies significantly by accident year
- Want data-driven loss ratio estimate
- Pricing loss ratio is uncertain

---

## Development Factors

### Tail Factor Estimation

**Purpose:** Estimate development beyond last observed period to ultimate

**Medical malpractice tail factors:** Often 1.05 to 1.20 (5-20% more development after 10+ years)

**Tail Estimation Methods:**

**1. Curve Fitting:**
```python
from scipy.optimize import curve_fit

def exponential_decay(x, a, b, c):
    """Exponential decay model for LDFs."""
    return a * np.exp(-b * x) + c

# Fit to selected LDFs
ldfs = np.array([3.50, 2.10, 1.45, 1.22, 1.12, 1.08, 1.05])
ages = np.arange(1, len(ldfs) + 1)

params, _ = curve_fit(exponential_decay, ages, ldfs)

# Extrapolate to tail (e.g., age 20)
tail_ages = np.arange(len(ldfs) + 1, 21)
tail_ldfs = exponential_decay(tail_ages, *params)
tail_factor = np.prod(tail_ldfs)
```

**2. Inverse Power Curve:**
```python
def inverse_power(x, a, b):
    """Inverse power model: LDF = a * x^(-b) + 1"""
    return a * x**(-b) + 1

# Fit to development factors
params, _ = curve_fit(inverse_power, ages, ldfs - 1)

# Extrapolate
tail_ldfs = inverse_power(tail_ages, *params)
tail_factor = np.prod(tail_ldfs - 1) + 1
```

**3. Benchmark Approach:**
Use industry tail factors for similar lines of business:

| Line of Business | Typical Tail Factor (10+ years) |
|------------------|----------------------------------|
| Medical Malpractice | 1.10 - 1.20 |
| Workers Compensation | 1.05 - 1.10 |
| General Liability | 1.08 - 1.15 |
| Auto Liability | 1.02 - 1.05 |
| Property | 1.00 - 1.01 (short-tail) |

**4. Sherman-type Tail:**
```python
def sherman_tail(last_ldf, n_periods=10):
    """Sherman tail factor estimation."""
    # Assumes geometric decay
    if last_ldf <= 1.0:
        return 1.0
    decay_rate = (last_ldf - 1.0) / 2  # Halve the increment each period
    tail = 1.0
    current_ldf = last_ldf
    for _ in range(n_periods):
        tail *= current_ldf
        current_ldf = 1.0 + decay_rate
        decay_rate /= 2
    return tail
```

### Development Factor Smoothing

**When to Smooth:**
- High volatility in selected factors
- Small sample sizes
- Outlier years distorting pattern

**Smoothing Techniques:**

**1. Moving Average:**
```python
def moving_average_factors(factors_df, window=3):
    """Smooth factors with moving average."""
    return factors_df.rolling(window=window, axis=0, min_periods=1).mean()
```

**2. Exponential Smoothing:**
```python
def exponential_smooth_factors(factors_df, alpha=0.3):
    """Smooth factors with exponential weighting (recent years weighted more)."""
    return factors_df.ewm(alpha=alpha, axis=0).mean()
```

**3. Fit to Curve:**
```python
from scipy.interpolate import UnivariateSpline

def spline_smooth_factors(factors, ages):
    """Smooth factors using cubic spline."""
    spline = UnivariateSpline(ages, factors, s=0.1)  # s controls smoothness
    return spline(ages)
```

---

## Credibility Theory

### Classical Credibility (Limited Fluctuation)

**Purpose:** Determine how much weight to give historical data vs. complement

**Full Credibility Standard:**
```
n_full = (z²/k²) × (σ²/μ²)

where:
n_full = claims needed for full credibility
z = normal quantile (1.96 for 95% confidence)
k = acceptable error percentage (e.g., 0.05 for 5%)
σ = standard deviation
μ = mean
```

**Partial Credibility:**
```
Z = √(n / n_full)  if n < n_full
Z = 1              if n >= n_full

Estimate = Z × Actual + (1-Z) × Expected
```

**Implementation:**
```python
def classical_credibility(n_claims, mean, std, confidence=0.95, tolerance=0.05):
    """Calculate credibility factor using limited fluctuation approach."""
    from scipy.stats import norm

    z_score = norm.ppf((1 + confidence) / 2)  # 1.96 for 95%
    cv = std / mean  # Coefficient of variation

    n_full = (z_score / tolerance)**2 * cv**2

    if n_claims >= n_full:
        return 1.0
    else:
        return np.sqrt(n_claims / n_full)

# Example: Medical malpractice severity
mean_severity = 250000
std_severity = 500000
n_claims = 50

Z = classical_credibility(n_claims, mean_severity, std_severity)
print(f"Credibility factor: {Z:.2f}")

# Credibility-weighted estimate
actual_severity = 280000
expected_severity = 240000
credible_estimate = Z * actual_severity + (1 - Z) * expected_severity
```

### Buhlmann Credibility

**Purpose:** Bayesian credibility using variance decomposition

**Formula:**
```
Z = n / (n + K)

where:
K = VHM / EPV  (process variance / expected process variance)
n = number of observations

Estimate = Z × X̄ + (1-Z) × μ
```

**Implementation:**
```python
def buhlmann_credibility(data_by_group):
    """Calculate Buhlmann credibility parameters.

    Args:
        data_by_group: dict of {group_id: [observations]}

    Returns:
        K (credibility parameter), μ (overall mean)
    """
    # Calculate group means
    group_means = {g: np.mean(obs) for g, obs in data_by_group.items()}
    overall_mean = np.mean([x for obs in data_by_group.values() for x in obs])

    # Process variance (EPV): average within-group variance
    epv = np.mean([np.var(obs) for obs in data_by_group.values()])

    # Variance of hypothetical means (VHM): variance of group means
    vhm = np.var(list(group_means.values()))

    # Credibility parameter K
    K = epv / vhm if vhm > 0 else np.inf

    return K, overall_mean

def apply_buhlmann(n, K, actual_mean, overall_mean):
    """Apply Buhlmann credibility formula."""
    Z = n / (n + K)
    return Z * actual_mean + (1 - Z) * overall_mean

# Example: Different physician experience levels
data = {
    'physician_1': [200000, 180000, 220000, 190000],  # 4 claims
    'physician_2': [300000, 310000, 290000],          # 3 claims
    'physician_3': [150000, 160000, 155000, 145000, 170000],  # 5 claims
}

K, mu = buhlmann_credibility(data)
print(f"K = {K:.2f}, Overall mean = ${mu:,.0f}")

# Credibility for physician_1
n1 = len(data['physician_1'])
actual_mean1 = np.mean(data['physician_1'])
credible_estimate1 = apply_buhlmann(n1, K, actual_mean1, mu)
print(f"Physician 1 credible estimate: ${credible_estimate1:,.0f}")
```

### Credibility-Weighted Loss Development

**Blend selected factors with industry factors:**
```python
def credibility_weighted_factors(company_factors, industry_factors, credibility):
    """Blend company factors with industry using credibility."""
    return credibility * company_factors + (1 - credibility) * industry_factors

# Example
company_ldfs = np.array([2.5, 1.8, 1.4, 1.2, 1.1])
industry_ldfs = np.array([2.3, 1.7, 1.35, 1.18, 1.08])
Z = 0.7  # 70% credibility to company data

blended_ldfs = credibility_weighted_factors(company_ldfs, industry_ldfs, Z)
```

---

## Loss Distributions

### Severity Distributions

**Pareto Distribution (Type I and II):**

Most common for insurance severity due to fat tails.

**Pareto Type I:**
```
F(x) = 1 - (θ/x)^α  for x ≥ θ
E[X] = αθ/(α-1)  for α > 1
Var[X] = αθ²/[(α-1)²(α-2)]  for α > 2
```

**Pareto Type II (Lomax):**
```
F(x) = 1 - (1 + x/θ)^(-α)  for x ≥ 0
```

**Python Implementation:**
```python
from scipy.stats import pareto, genpareto
import matplotlib.pyplot as plt

# Fit Pareto to data
severity_data = np.array([50000, 80000, 120000, 200000, 350000, 500000])

# Pareto Type I
shape, loc, scale = pareto.fit(severity_data, floc=0)
fitted_pareto = pareto(shape, loc=loc, scale=scale)

# Pareto Type II (Generalized Pareto)
shape_gp, loc_gp, scale_gp = genpareto.fit(severity_data)
fitted_genpareto = genpareto(shape_gp, loc=loc_gp, scale=scale_gp)

# Expected severity
expected_severity_p1 = fitted_pareto.mean()
expected_severity_p2 = fitted_genpareto.mean()

# Tail probabilities
prob_exceed_1M = 1 - fitted_genpareto.cdf(1000000)
```

**Lognormal Distribution:**

Alternative for claim amounts with moderate tails.

```python
from scipy.stats import lognorm

# Fit lognormal
shape_ln, loc_ln, scale_ln = lognorm.fit(severity_data, floc=0)
fitted_lognorm = lognorm(shape_ln, loc=loc_ln, scale=scale_ln)

# Parameters
mu = np.log(scale_ln)
sigma = shape_ln

# Expected value and variance
E_X = np.exp(mu + sigma**2 / 2)
Var_X = (np.exp(sigma**2) - 1) * np.exp(2*mu + sigma**2)
```

**Weibull Distribution:**

Used for claim duration or time-to-settlement.

```python
from scipy.stats import weibull_min

# Fit Weibull
shape_w, loc_w, scale_w = weibull_min.fit(severity_data, floc=0)
fitted_weibull = weibull_min(shape_w, loc=loc_w, scale=scale_w)
```

### Distribution Fitting and Goodness-of-Fit

**Maximum Likelihood Estimation (MLE):**
```python
# scipy.stats.fit() uses MLE by default
fitted_params = genpareto.fit(severity_data)
```

**Method of Moments:**
```python
def fit_pareto_moments(data):
    """Fit Pareto using method of moments."""
    x_bar = np.mean(data)
    x_min = np.min(data)
    alpha = (x_bar + x_min) / (x_bar - x_min)
    theta = x_min * (alpha - 1)
    return alpha, theta
```

**Goodness-of-Fit Tests:**

**1. Kolmogorov-Smirnov Test:**
```python
from scipy.stats import kstest

# Test fitted distribution
ks_stat, p_value = kstest(severity_data, fitted_genpareto.cdf)
print(f"KS statistic: {ks_stat:.4f}, p-value: {p_value:.4f}")
# p-value > 0.05 suggests good fit
```

**2. Anderson-Darling Test:**
```python
from scipy.stats import anderson

# Transform to standard distribution first
standardized = (severity_data - loc_gp) / scale_gp
ad_result = anderson(standardized, dist='expon')  # For generalized pareto
```

**3. Chi-Square Test:**
```python
from scipy.stats import chisquare

# Create bins
bins = np.percentile(severity_data, [0, 25, 50, 75, 100])
observed, _ = np.histogram(severity_data, bins=bins)
expected = np.diff(fitted_genpareto.cdf(bins)) * len(severity_data)

chi2_stat, p_value = chisquare(observed, expected)
```

**4. Q-Q Plot (Visual):**
```python
from scipy.stats import probplot

fig, ax = plt.subplots()
probplot(severity_data, dist=fitted_genpareto, plot=ax)
plt.title("Q-Q Plot: Generalized Pareto")
plt.show()
```

### Limited Expected Value (LEV)

**Purpose:** Calculate expected value limited by policy limit

**Formula:**
```
LEV(x) = E[min(X, x)]
```

**For Pareto:**
```python
def lev_pareto(limit, alpha, theta):
    """Limited expected value for Pareto distribution."""
    if limit <= theta:
        return limit
    else:
        return (alpha * theta / (alpha - 1)) * (1 - (theta / limit)**(alpha - 1))

# Example: Policy limit $1M, Pareto(α=2.5, θ=100K)
lev_1m = lev_pareto(1000000, alpha=2.5, theta=100000)
print(f"LEV at $1M limit: ${lev_1m:,.0f}")
```

**For Lognormal:**
```python
from scipy.stats import norm

def lev_lognormal(limit, mu, sigma):
    """Limited expected value for lognormal distribution."""
    if limit <= 0:
        return 0
    exp_x = np.exp(mu + sigma**2 / 2)
    lev = exp_x * norm.cdf((np.log(limit) - mu - sigma**2) / sigma) + \
          limit * (1 - norm.cdf((np.log(limit) - mu) / sigma))
    return lev
```

### Excess Loss Expected Value

**Purpose:** Calculate expected value above deductible

**Formula:**
```
E[X - d | X > d] = E[X] - LEV(d) / (1 - F(d))
```

---

## Frequency-Severity Modeling

### Claim Count Distributions

**Poisson Distribution:**

Most common for claim frequency.

```python
from scipy.stats import poisson

# Fit Poisson to claim counts
claim_counts = np.array([5, 8, 6, 7, 9, 5, 6])
lambda_hat = np.mean(claim_counts)

poisson_dist = poisson(lambda_hat)

# Probability of exactly k claims
prob_10_claims = poisson_dist.pmf(10)

# Expected claim count
expected_claims = poisson_dist.mean()
```

**Negative Binomial Distribution:**

Overdispersed alternative when variance > mean.

```python
from scipy.stats import nbinom

# Negative binomial handles overdispersion
r, p = nbinom.fit(claim_counts)[:2]
nb_dist = nbinom(r, p)

# Variance > mean indicates overdispersion
print(f"Mean: {np.mean(claim_counts):.2f}, Variance: {np.var(claim_counts):.2f}")
```

### Aggregate Loss Distribution

**Purpose:** Model total losses = frequency × severity

**Compound Poisson:**
```
S = X₁ + X₂ + ... + X_N
where N ~ Poisson(λ), X_i ~ Severity distribution
```

**Monte Carlo Simulation:**
```python
def simulate_aggregate_loss(n_sim, lambda_freq, severity_dist):
    """Simulate aggregate loss using compound Poisson."""
    aggregate_losses = []

    for _ in range(n_sim):
        # Simulate claim count
        n_claims = np.random.poisson(lambda_freq)

        # Simulate severities
        if n_claims > 0:
            severities = severity_dist.rvs(size=n_claims)
            total_loss = severities.sum()
        else:
            total_loss = 0

        aggregate_losses.append(total_loss)

    return np.array(aggregate_losses)

# Example: λ=100 claims/year, Pareto severity
n_simulations = 10000
lambda_claims = 100
severity_dist = genpareto(shape_gp, loc=loc_gp, scale=scale_gp)

agg_losses = simulate_aggregate_loss(n_simulations, lambda_claims, severity_dist)

# Calculate risk measures
expected_loss = np.mean(agg_losses)
var_95 = np.percentile(agg_losses, 95)  # Value at Risk
tvar_95 = np.mean(agg_losses[agg_losses >= var_95])  # Tail VaR
```

**Fast Fourier Transform (FFT) Method:**

For exact calculation (without simulation):

```python
# Requires discretization of severity distribution
# See Panjer recursion or FFT convolution methods
# (Implementation complex - use actuar R package or reference texts)
```

---

## Trend Analysis

### Loss Cost Trend

**Components:**
```
Loss Cost Trend = Frequency Trend × Severity Trend
```

**Frequency Trend:**
```python
def fit_frequency_trend(years, claim_counts, exposures):
    """Fit exponential trend to claim frequency."""
    frequency = claim_counts / exposures
    log_freq = np.log(frequency)

    # Linear regression on log scale
    from scipy.stats import linregress
    slope, intercept, r_value, p_value, std_err = linregress(years, log_freq)

    # Annual trend factor
    trend_factor = np.exp(slope)

    return trend_factor, r_value**2

# Example
years = np.array([2019, 2020, 2021, 2022, 2023])
claim_counts = np.array([450, 460, 480, 490, 510])
exposures = np.array([5000, 5100, 5200, 5300, 5400])

freq_trend, r_squared = fit_frequency_trend(years, claim_counts, exposures)
print(f"Annual frequency trend: {(freq_trend - 1)*100:.2f}% (R² = {r_squared:.3f})")
```

**Severity Trend:**
```python
def fit_severity_trend(years, avg_severity):
    """Fit exponential trend to severity."""
    log_sev = np.log(avg_severity)

    from scipy.stats import linregress
    slope, intercept, r_value, p_value, std_err = linregress(years, log_sev)

    trend_factor = np.exp(slope)

    return trend_factor, r_value**2

# Example
avg_severity = np.array([250000, 265000, 280000, 295000, 310000])
sev_trend, r_squared = fit_severity_trend(years, avg_severity)
print(f"Annual severity trend: {(sev_trend - 1)*100:.2f}%")
```

**On-Level Premium:**
```python
def on_level_premium(premium, rate_changes, effective_dates, valuation_date):
    """Adjust historical premium to current rate level."""
    on_level = premium.copy()

    for i, (rate_change, eff_date) in enumerate(zip(rate_changes, effective_dates)):
        if eff_date <= valuation_date:
            # Apply cumulative rate changes after this effective date
            future_changes = rate_changes[i+1:]
            cumulative_change = np.prod([1 + rc for rc in future_changes])
            on_level *= cumulative_change

    return on_level
```

### Medical Cost Inflation

**Medical Severity Trend Components:**
- Medical inflation (CPI-Medical: ~4-5% annually)
- Legal/tort inflation (varies by jurisdiction)
- Practice pattern changes
- Technology advancement (diagnostic and treatment)

**Example Index:**
```python
# Medical malpractice severity index
base_year_severity = 250000
years_forward = 5
medical_inflation = 0.045  # 4.5%
legal_inflation = 0.03     # 3%

# Composite trend
composite_trend = (1 + medical_inflation) * (1 + legal_inflation) - 1

projected_severity = base_year_severity * (1 + composite_trend)**years_forward
```

---

## Regulatory and Standards Context

### NAIC Annual Statement Requirements

**Schedule P - Analysis of Losses and Loss Expenses:**

**Part 1:** Summary of losses and loss adjustment expenses (LAE)
- Losses paid
- Loss reserves (case + IBNR)
- LAE paid
- LAE reserves

**Part 2:** Loss and LAE development triangles (10 years)
- Cumulative paid losses by accident year
- Cumulative incurred losses by accident year
- Bulk and IBNR reserves by accident year

**Part 3:** Reconciliation of calendar year reserves

**Python Generation Example:**
```python
def generate_schedule_p_part2(paid_triangle, incurred_triangle, ibnr_triangle):
    """Generate NAIC Schedule P Part 2 exhibit."""
    # Format triangles with proper accident year rows and development columns
    schedule_p = {
        'paid': paid_triangle.to_dict(),
        'incurred': incurred_triangle.to_dict(),
        'ibnr': ibnr_triangle.to_dict()
    }
    return schedule_p
```

### SAP vs. GAAP Accounting

**Statutory Accounting Principles (SAP):**
- More conservative than GAAP
- Focuses on insurer solvency
- No discounting of reserves (except workers comp and structured settlements)
- Limited recognition of certain assets (non-admitted assets)
- Premium deficiency reserve required immediately

**GAAP Accounting:**
- Revenue matching (recognize over policy period)
- Discounting allowed for long-tail reserves
- More liberal asset admissibility
- Premium deficiency recognized only when expected losses > unearned premium

**Key Differences:**

| Aspect | SAP | GAAP |
|--------|-----|------|
| Reserve Discounting | Generally prohibited | Allowed |
| Premium Recognition | Immediate | Over policy period |
| Acquisition Costs | Expensed immediately | Deferred (DAC) |
| Focus | Solvency | Economic reality |

### ASOP 43 - Property/Casualty Unpaid Claim Estimates

**Actuarial Standards of Practice No. 43 Requirements:**

**1. Selection of Methods:**
- Document why methods were chosen
- Consider multiple methods
- Explain reliance on single method (if applicable)

**2. Assumptions:**
- Document all significant assumptions
- Explain basis for assumptions
- Consider reasonableness

**3. Data Quality:**
- Document data used
- Note limitations or concerns
- Describe adjustments made

**4. Uncertainty:**
- Discuss sources of variability
- Provide range estimates where appropriate
- Note limitations of point estimates

**5. Documentation:**
```python
# Example ASOP 43 compliant documentation structure

asop43_documentation = {
    'methods_used': ['Chain Ladder', 'Bornhuetter-Ferguson', 'Cape Cod'],
    'method_selection_rationale': 'CL for mature years, BF for recent',
    'data_sources': ['Claims system extract 2024-12-31', 'Premium from policy admin'],
    'data_quality_issues': ['2022 case reserve strengthening', 'COVID claims in 2020-2021'],
    'key_assumptions': [
        'Development patterns stable post-2022',
        'Expected loss ratio 65% based on 5-year avg',
        'Tail factor 1.08 based on industry benchmark'
    ],
    'sensitivity_analysis': {
        'tail_factor': {'low': 1.05, 'selected': 1.08, 'high': 1.12},
        'expected_lr': {'low': 0.62, 'selected': 0.65, 'high': 0.68}
    },
    'range_of_estimates': {'low': 45000000, 'point': 50000000, 'high': 56000000}
}
```

### Appointed Actuary Responsibilities

**Statement of Actuarial Opinion (SAO):**
- Required for P&C insurers
- Opine on reserve adequacy
- Reasonable/inadequate/excessive determination
- Sign as appointed actuary

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Mixing accident year and calendar year | Distorts development patterns | Use consistent origin period |
| Ignoring tail factors | Understates IBNR for long-tail lines | Fit curves or use industry benchmarks |
| Using paid data when case reserves are poor | Masks reserve deficiency | Use incurred or adjust case reserves |
| Not checking for case reserve changes | Artificial development patterns | Adjust for known changes |
| Selecting one method without sensitivity | Ignores uncertainty | Show range from multiple methods |
| Using simple average with volatile volume | Overweights small years | Use volume-weighted or medial |
| No credibility adjustment for small data | High variance, unstable estimates | Blend with industry or pricing |
| Applying frequency trend to severity | Conceptually incorrect | Separate frequency and severity trends |
| Not documenting assumptions | ASOP 43 violation, unauditable | Document everything in writing |
| Fitting normal distribution to severity | Severe underestimation of tail | Use Pareto, lognormal, or other fat-tailed |

---

## Quick Reference

### Common Actuarial Formulas

**Age-to-Age Factor:**
```
LDF_{k→k+1} = Cumulative_{k+1} / Cumulative_k
```

**Cumulative Development Factor (CDF):**
```
CDF_k = LDF_k × LDF_{k+1} × ... × LDF_n × Tail
```

**IBNR:**
```
IBNR = Ultimate - Reported
```

**Bornhuetter-Ferguson:**
```
Ultimate = Reported + (Expected × % Unreported)
% Unreported = 1 - (1 / CDF)
```

**Cape Cod:**
```
LR = Σ(Reported) / Σ(Premium × % Reported)
Ultimate = Reported + (Premium × LR × % Unreported)
```

**Credibility (Classical):**
```
Z = √(n / n_full)
Estimate = Z × Actual + (1-Z) × Expected
```

**Buhlmann Credibility:**
```
Z = n / (n + K)
K = EPV / VHM
```

### Python Libraries

| Library | Purpose |
|---------|---------|
| chainladder-python | Triangle manipulation, reserving methods |
| scipy.stats | Distribution fitting, statistical tests |
| pandas | Data manipulation, triangle construction |
| numpy | Numerical calculations |
| matplotlib/seaborn | Visualization |

### Triangle Terminology

| Term | Definition |
|------|------------|
| Origin | Accident year, policy year, or report year |
| Development Period | Age of claims (months or years since origin) |
| Diagonal | Latest reported values (different ages) |
| Calendar Year | Column of payments in a specific year |
| Cumulative | Running total from origin to development period |
| Incremental | Amount paid/incurred in specific development period |

### Method Selection Guide

| Scenario | Recommended Method |
|----------|-------------------|
| Stable patterns, mature data | Chain Ladder |
| Recent years, immature | Bornhuetter-Ferguson |
| Heterogeneous book, varying volume | Cape Cod |
| Unstable patterns | Credibility-weighted blend |
| Paid vs. case reserve issues | Use incurred or separate case adequacy |
| Long-tail with limited data | Use industry tail factors |
| New line of business | Pure premium method or BF with pricing LR |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Loss triangle ETL | `insurance-analytics/etl/loss_triangles.py` |
| Reserving models | `insurance-analytics/models/reserves.py` |
| Distribution fitting | `insurance-analytics/analysis/severity_models.py` |
| NAIC Schedule P generator | `insurance-analytics/reporting/schedule_p.py` |
| Credibility calculations | `insurance-analytics/models/credibility.py` |

---

## References

- [Casualty Actuarial Society (CAS)](https://www.casact.org/)
- [CAS Exam 5 Syllabus - Basic Techniques](https://www.casact.org/exam/exam-5-basic-techniques-ratemaking-and-estimating-claim-liabilities)
- [ASOP 43 - Property/Casualty Unpaid Claims](https://www.actuary.org/sites/default/files/2021-05/ASOP_043.pdf)
- [chainladder-python Documentation](https://chainladder-python.readthedocs.io/)
- [Loss Models (Klugman, Panjer, Willmot)](https://www.wiley.com/go/lossmodels5)
- [Estimating Unpaid Claims Using Basic Techniques (CAS Study Note)](https://www.casact.org/sites/default/files/2021-02/estimating-unpaid-claims-using-basic-techniques.pdf)
- [NAIC Annual Statement Instructions](https://content.naic.org/pbr-annual-reporting.htm)

---

**Document Version:** 1.0
**Last Updated:** 2026-02-05
**Maintained By:** Aliunde AI Team
**Target Certification:** CAS Exams 5 and 7 alignment
