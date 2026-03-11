---
name: ali-data-science
description: |
  Data science patterns for statistical analysis and exploratory data analysis. Use when:

  PLANNING: Designing EDA workflows, planning statistical analyses, choosing
  visualization strategies, architecting data quality assessments

  IMPLEMENTATION: Building EDA notebooks, implementing hypothesis tests, creating
  visualizations, writing data profiling code, handling missing data

  GUIDANCE: Asking about statistical tests, correlation analysis, outlier detection,
  visualization best practices, A/B testing, sampling strategies

  REVIEW: Checking statistical methodology, validating hypothesis tests, auditing
  data quality assessments, reviewing visualization accuracy

  Do NOT use for data warehouse modeling, ETL/ELT pipelines, or medallion architecture
  (use ali-data-architecture instead), or predictive ML models and training pipelines
  (use ali-machine-learning instead)
---

# Data Science

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing exploratory data analysis workflows
- Planning statistical hypothesis tests
- Choosing appropriate visualizations
- Architecting data quality frameworks

**Implementation:**
- Building EDA notebooks and reports
- Implementing statistical tests
- Creating visualizations with matplotlib/seaborn/plotly
- Writing data profiling and quality code
- Handling missing data and outliers

**Guidance/Best Practices:**
- Asking about which statistical test to use
- Asking about correlation vs causation
- Asking about data distribution analysis
- Asking about A/B test design

**Review/Validation:**
- Reviewing statistical methodology
- Checking for p-hacking or multiple comparison issues
- Validating sample size calculations
- Auditing data quality assessments

---

## Key Principles

- **Understand before modeling**: EDA must precede any predictive modeling
- **Visualize first**: A picture is worth a thousand statistics
- **Correlation is not causation**: Be explicit about what analysis can and cannot prove
- **Sample size matters**: Underpowered studies produce unreliable results
- **Multiple comparisons inflate error**: Adjust p-values when testing many hypotheses
- **Missing data is informative**: The pattern of missingness often tells a story
- **Outliers demand investigation**: Don't remove until you understand why they exist
- **Reproducibility required**: Document analysis steps, set random seeds, version data

---

## EDA Workflow Checklist

### Phase 1: Data Overview

```python
import pandas as pd
import numpy as np

def data_overview(df):
    """Generate comprehensive data overview."""
    print(f"Shape: {df.shape[0]:,} rows x {df.shape[1]} columns")
    print(f"\nColumn types:\n{df.dtypes.value_counts()}")
    print(f"\nMemory usage: {df.memory_usage(deep=True).sum() / 1e6:.2f} MB")

    # Missing values summary
    missing = df.isnull().sum()
    missing_pct = (missing / len(df) * 100).round(2)
    missing_df = pd.DataFrame({
        'missing_count': missing,
        'missing_pct': missing_pct
    }).query('missing_count > 0').sort_values('missing_pct', ascending=False)

    if len(missing_df) > 0:
        print(f"\nMissing values:\n{missing_df}")
    else:
        print("\nNo missing values")

    return df.describe(include='all')
```

### Phase 2: Univariate Analysis

```python
import matplotlib.pyplot as plt
import seaborn as sns

def analyze_numeric(df, column):
    """Analyze a single numeric column."""
    data = df[column].dropna()

    fig, axes = plt.subplots(1, 3, figsize=(15, 4))

    # Histogram with KDE
    sns.histplot(data, kde=True, ax=axes[0])
    axes[0].set_title(f'{column} Distribution')

    # Box plot
    sns.boxplot(x=data, ax=axes[1])
    axes[1].set_title(f'{column} Box Plot')

    # QQ plot for normality
    from scipy import stats
    stats.probplot(data, dist="norm", plot=axes[2])
    axes[2].set_title(f'{column} Q-Q Plot')

    plt.tight_layout()

    # Statistics
    print(f"\n{column} Statistics:")
    print(f"  Count:    {len(data):,}")
    print(f"  Mean:     {data.mean():.4f}")
    print(f"  Median:   {data.median():.4f}")
    print(f"  Std:      {data.std():.4f}")
    print(f"  Skewness: {data.skew():.4f}")
    print(f"  Kurtosis: {data.kurtosis():.4f}")

    # Normality test
    if len(data) >= 20:
        stat, p_value = stats.shapiro(data[:5000])  # Shapiro-Wilk limited to 5000
        print(f"  Shapiro-Wilk p-value: {p_value:.4f} ({'Normal' if p_value > 0.05 else 'Not Normal'})")

def analyze_categorical(df, column, top_n=10):
    """Analyze a single categorical column."""
    data = df[column].dropna()
    value_counts = data.value_counts()

    fig, axes = plt.subplots(1, 2, figsize=(12, 4))

    # Bar plot of top categories
    value_counts.head(top_n).plot(kind='bar', ax=axes[0])
    axes[0].set_title(f'{column} Top {top_n} Categories')
    axes[0].tick_params(axis='x', rotation=45)

    # Pie chart if few categories
    if len(value_counts) <= 6:
        value_counts.plot(kind='pie', autopct='%1.1f%%', ax=axes[1])
        axes[1].set_title(f'{column} Distribution')
    else:
        # Show category frequency distribution
        axes[1].hist(value_counts.values, bins=20)
        axes[1].set_title(f'{column} Category Frequency Distribution')
        axes[1].set_xlabel('Frequency')

    plt.tight_layout()

    print(f"\n{column} Statistics:")
    print(f"  Unique values: {data.nunique():,}")
    print(f"  Most common:   {value_counts.index[0]} ({value_counts.iloc[0]:,})")
    print(f"  Least common:  {value_counts.index[-1]} ({value_counts.iloc[-1]:,})")
```

### Phase 3: Bivariate Analysis

```python
def correlation_analysis(df, method='pearson'):
    """Generate correlation matrix and identify strong correlations."""
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    corr_matrix = df[numeric_cols].corr(method=method)

    # Heatmap
    plt.figure(figsize=(12, 10))
    mask = np.triu(np.ones_like(corr_matrix, dtype=bool))
    sns.heatmap(corr_matrix, mask=mask, annot=True, fmt='.2f',
                cmap='coolwarm', center=0, vmin=-1, vmax=1)
    plt.title(f'{method.title()} Correlation Matrix')
    plt.tight_layout()

    # Find strong correlations
    strong_corr = []
    for i in range(len(corr_matrix.columns)):
        for j in range(i+1, len(corr_matrix.columns)):
            if abs(corr_matrix.iloc[i, j]) >= 0.7:
                strong_corr.append({
                    'var1': corr_matrix.columns[i],
                    'var2': corr_matrix.columns[j],
                    'correlation': corr_matrix.iloc[i, j]
                })

    if strong_corr:
        print("\nStrong correlations (|r| >= 0.7):")
        for c in sorted(strong_corr, key=lambda x: abs(x['correlation']), reverse=True):
            print(f"  {c['var1']} <-> {c['var2']}: {c['correlation']:.3f}")

    return corr_matrix

def scatter_with_regression(df, x_col, y_col, hue_col=None):
    """Scatter plot with regression line."""
    plt.figure(figsize=(10, 6))

    if hue_col:
        sns.scatterplot(data=df, x=x_col, y=y_col, hue=hue_col, alpha=0.6)
    else:
        sns.regplot(data=df, x=x_col, y=y_col, scatter_kws={'alpha': 0.5})

    # Calculate correlation
    corr = df[[x_col, y_col]].corr().iloc[0, 1]
    plt.title(f'{x_col} vs {y_col} (r = {corr:.3f})')
    plt.tight_layout()
```

### Phase 4: Time Series Analysis

```python
def time_series_eda(df, date_col, value_col):
    """EDA for time series data."""
    df = df.sort_values(date_col).copy()

    fig, axes = plt.subplots(3, 1, figsize=(14, 10))

    # Time series plot
    axes[0].plot(df[date_col], df[value_col])
    axes[0].set_title(f'{value_col} Over Time')
    axes[0].set_xlabel('Date')

    # Rolling statistics
    window = min(30, len(df) // 10)
    rolling_mean = df[value_col].rolling(window=window).mean()
    rolling_std = df[value_col].rolling(window=window).std()

    axes[1].plot(df[date_col], df[value_col], label='Original', alpha=0.5)
    axes[1].plot(df[date_col], rolling_mean, label=f'{window}-period Mean', color='red')
    axes[1].fill_between(df[date_col], rolling_mean - 2*rolling_std,
                         rolling_mean + 2*rolling_std, alpha=0.2, color='red')
    axes[1].legend()
    axes[1].set_title('Rolling Mean and 2-Std Band')

    # Seasonality check (if enough data)
    if len(df) > 60:
        from statsmodels.tsa.seasonal import seasonal_decompose
        try:
            decomposition = seasonal_decompose(df.set_index(date_col)[value_col],
                                               period=min(12, len(df)//4))
            decomposition.seasonal.plot(ax=axes[2])
            axes[2].set_title('Seasonal Component')
        except Exception as e:
            axes[2].text(0.5, 0.5, f'Decomposition failed: {e}',
                        ha='center', va='center')

    plt.tight_layout()
```

---

## Statistical Distributions

### Common Distributions and When They Appear

| Distribution | Shape | Common In |
|--------------|-------|-----------|
| **Normal** | Bell curve, symmetric | Heights, weights, measurement errors |
| **Log-normal** | Right-skewed | Income, stock prices, file sizes |
| **Exponential** | Steep decline | Wait times, lifetimes |
| **Poisson** | Discrete counts | Event counts per time period |
| **Binomial** | Discrete | Success/failure outcomes |
| **Uniform** | Flat | Random IDs, shuffled data |
| **Bimodal** | Two peaks | Mixed populations |

### Distribution Testing

```python
from scipy import stats

def test_distribution(data, dist_name='norm'):
    """Test if data follows a specific distribution."""

    # Fit distribution
    if dist_name == 'norm':
        params = stats.norm.fit(data)
        dist = stats.norm(*params)
    elif dist_name == 'lognorm':
        params = stats.lognorm.fit(data)
        dist = stats.lognorm(*params)
    elif dist_name == 'expon':
        params = stats.expon.fit(data)
        dist = stats.expon(*params)

    # Kolmogorov-Smirnov test
    ks_stat, ks_p = stats.kstest(data, dist.cdf)

    # Visual comparison
    fig, axes = plt.subplots(1, 2, figsize=(12, 4))

    # Histogram vs fitted distribution
    axes[0].hist(data, bins=30, density=True, alpha=0.7, label='Data')
    x = np.linspace(data.min(), data.max(), 100)
    axes[0].plot(x, dist.pdf(x), 'r-', lw=2, label=f'Fitted {dist_name}')
    axes[0].legend()
    axes[0].set_title(f'Data vs Fitted {dist_name}')

    # Q-Q plot
    stats.probplot(data, dist=dist_name, plot=axes[1])
    axes[1].set_title(f'Q-Q Plot ({dist_name})')

    plt.tight_layout()

    print(f"K-S Test: statistic={ks_stat:.4f}, p-value={ks_p:.4f}")
    print(f"Conclusion: {'Consistent with' if ks_p > 0.05 else 'Different from'} {dist_name}")
```

---

## Hypothesis Testing

### Test Selection Guide

| Question | Data Type | Test |
|----------|-----------|------|
| Is the mean different from X? | Numeric | One-sample t-test |
| Are two group means different? | Numeric | Two-sample t-test |
| Are paired measurements different? | Numeric | Paired t-test |
| Are 3+ group means different? | Numeric | ANOVA |
| Is there association between categories? | Categorical | Chi-square |
| Are two distributions different? | Any | Mann-Whitney U |
| Is there correlation? | Numeric | Pearson/Spearman |

### Common Statistical Tests

```python
from scipy import stats

# One-sample t-test: Is mean different from hypothesized value?
def one_sample_ttest(data, hypothesized_mean):
    stat, p_value = stats.ttest_1samp(data, hypothesized_mean)
    print(f"Sample mean: {data.mean():.4f}")
    print(f"Hypothesized mean: {hypothesized_mean}")
    print(f"t-statistic: {stat:.4f}, p-value: {p_value:.4f}")
    print(f"Conclusion: {'Reject' if p_value < 0.05 else 'Fail to reject'} H0")
    return stat, p_value

# Two-sample t-test: Are two group means different?
def two_sample_ttest(group1, group2, equal_var=False):
    """Welch's t-test by default (unequal variances)."""
    stat, p_value = stats.ttest_ind(group1, group2, equal_var=equal_var)
    print(f"Group 1 mean: {group1.mean():.4f} (n={len(group1)})")
    print(f"Group 2 mean: {group2.mean():.4f} (n={len(group2)})")
    print(f"Difference: {group1.mean() - group2.mean():.4f}")
    print(f"t-statistic: {stat:.4f}, p-value: {p_value:.4f}")
    print(f"Conclusion: {'Significant' if p_value < 0.05 else 'Not significant'} difference")
    return stat, p_value

# Paired t-test: Are paired measurements different?
def paired_ttest(before, after):
    """For before/after or matched pairs."""
    stat, p_value = stats.ttest_rel(before, after)
    diff = after - before
    print(f"Mean difference: {diff.mean():.4f}")
    print(f"t-statistic: {stat:.4f}, p-value: {p_value:.4f}")
    return stat, p_value

# ANOVA: Are 3+ group means different?
def one_way_anova(*groups):
    """Test if any group mean differs from others."""
    stat, p_value = stats.f_oneway(*groups)
    print(f"Group means: {[g.mean() for g in groups]}")
    print(f"F-statistic: {stat:.4f}, p-value: {p_value:.4f}")
    print(f"Conclusion: {'At least one' if p_value < 0.05 else 'No'} group differs")
    return stat, p_value

# Chi-square test: Association between categorical variables?
def chi_square_test(df, col1, col2):
    """Test independence of two categorical variables."""
    contingency = pd.crosstab(df[col1], df[col2])
    chi2, p_value, dof, expected = stats.chi2_contingency(contingency)
    print(f"Chi-square: {chi2:.4f}, p-value: {p_value:.4f}, df: {dof}")
    print(f"Conclusion: {'Dependent' if p_value < 0.05 else 'Independent'} variables")
    return chi2, p_value, contingency

# Mann-Whitney U: Non-parametric comparison of distributions
def mann_whitney_test(group1, group2):
    """Use when data is not normally distributed."""
    stat, p_value = stats.mannwhitneyu(group1, group2, alternative='two-sided')
    print(f"U-statistic: {stat:.4f}, p-value: {p_value:.4f}")
    return stat, p_value
```

### Multiple Comparison Correction

```python
from statsmodels.stats.multitest import multipletests

def correct_pvalues(p_values, method='fdr_bh'):
    """
    Correct for multiple comparisons.

    Methods:
    - 'bonferroni': Conservative, controls family-wise error rate
    - 'fdr_bh': Benjamini-Hochberg, controls false discovery rate (preferred)
    - 'holm': Less conservative than Bonferroni
    """
    reject, corrected_p, _, _ = multipletests(p_values, method=method)

    results = pd.DataFrame({
        'original_p': p_values,
        'corrected_p': corrected_p,
        'reject_h0': reject
    })

    print(f"Method: {method}")
    print(f"Tests significant before correction: {sum(np.array(p_values) < 0.05)}")
    print(f"Tests significant after correction: {sum(reject)}")

    return results
```

---

## Correlation Analysis

### Types of Correlation

| Type | Use When | Range | Sensitive To |
|------|----------|-------|--------------|
| **Pearson** | Linear relationships | -1 to 1 | Outliers |
| **Spearman** | Monotonic relationships | -1 to 1 | Robust to outliers |
| **Kendall** | Ordinal data, small samples | -1 to 1 | More robust than Spearman |

```python
from scipy import stats

def comprehensive_correlation(x, y):
    """Calculate all correlation types with significance."""

    # Remove NaN pairs
    mask = ~(np.isnan(x) | np.isnan(y))
    x, y = x[mask], y[mask]

    # Pearson (linear)
    pearson_r, pearson_p = stats.pearsonr(x, y)

    # Spearman (monotonic)
    spearman_r, spearman_p = stats.spearmanr(x, y)

    # Kendall (ordinal)
    kendall_r, kendall_p = stats.kendalltau(x, y)

    print(f"Pearson:  r = {pearson_r:.4f}, p = {pearson_p:.4f}")
    print(f"Spearman: r = {spearman_r:.4f}, p = {spearman_p:.4f}")
    print(f"Kendall:  r = {kendall_r:.4f}, p = {kendall_p:.4f}")

    # Interpretation
    if abs(pearson_r - spearman_r) > 0.1:
        print("\nNote: Pearson and Spearman differ significantly.")
        print("This suggests non-linear relationship or outliers.")

    return {
        'pearson': (pearson_r, pearson_p),
        'spearman': (spearman_r, spearman_p),
        'kendall': (kendall_r, kendall_p)
    }
```

### Correlation Strength Guidelines

| |r| Value | Interpretation |
|-----------|----------------|
| 0.00 - 0.19 | Very weak |
| 0.20 - 0.39 | Weak |
| 0.40 - 0.59 | Moderate |
| 0.60 - 0.79 | Strong |
| 0.80 - 1.00 | Very strong |

---

## Missing Data Analysis

### Missing Data Types

| Type | Pattern | Handling |
|------|---------|----------|
| **MCAR** (Missing Completely at Random) | Random, no pattern | Safe to delete or impute |
| **MAR** (Missing at Random) | Depends on observed data | Impute with observed predictors |
| **MNAR** (Missing Not at Random) | Depends on missing value itself | Most problematic, may need domain knowledge |

### Missing Data Assessment

```python
import missingno as msno

def missing_data_analysis(df):
    """Comprehensive missing data analysis."""

    # Summary statistics
    missing = df.isnull().sum()
    missing_pct = (missing / len(df) * 100)
    missing_df = pd.DataFrame({
        'missing_count': missing,
        'missing_pct': missing_pct.round(2)
    }).query('missing_count > 0').sort_values('missing_pct', ascending=False)

    print("Missing Data Summary:")
    print(missing_df)
    print(f"\nRows with any missing: {df.isnull().any(axis=1).sum():,} ({df.isnull().any(axis=1).mean()*100:.1f}%)")

    # Visualizations
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))

    # Missing data matrix
    msno.matrix(df, ax=axes[0], sparkline=False)
    axes[0].set_title('Missing Data Matrix')

    # Missing data correlation (which columns tend to be missing together)
    msno.heatmap(df, ax=axes[1])
    axes[1].set_title('Missing Data Correlation')

    plt.tight_layout()

    return missing_df

def test_mcar(df, missing_col, test_cols):
    """
    Test if missingness is completely at random (MCAR).
    Uses Little's MCAR test approximation via t-tests.
    """
    missing_mask = df[missing_col].isnull()

    print(f"Testing if {missing_col} missingness is MCAR:")

    for col in test_cols:
        if col == missing_col:
            continue
        if df[col].dtype in [np.float64, np.int64]:
            group_with = df.loc[~missing_mask, col].dropna()
            group_without = df.loc[missing_mask, col].dropna()

            if len(group_with) > 0 and len(group_without) > 0:
                stat, p = stats.ttest_ind(group_with, group_without)
                sig = '*' if p < 0.05 else ''
                print(f"  {col}: t={stat:.3f}, p={p:.4f} {sig}")
```

### Imputation Strategies

```python
from sklearn.impute import SimpleImputer, KNNImputer
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer

# Simple imputation
def simple_impute(df, strategy='median'):
    """
    Strategies: 'mean', 'median', 'most_frequent', 'constant'
    """
    imputer = SimpleImputer(strategy=strategy)
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    df[numeric_cols] = imputer.fit_transform(df[numeric_cols])
    return df

# KNN imputation (uses similar rows)
def knn_impute(df, n_neighbors=5):
    """Use when missingness depends on similar observations."""
    imputer = KNNImputer(n_neighbors=n_neighbors)
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    df[numeric_cols] = imputer.fit_transform(df[numeric_cols])
    return df

# Multiple imputation (MICE)
def mice_impute(df, n_imputations=5):
    """
    Multiple Imputation by Chained Equations.
    Use for MAR data.
    """
    imputer = IterativeImputer(random_state=42, max_iter=10)
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    df[numeric_cols] = imputer.fit_transform(df[numeric_cols])
    return df
```

---

## Outlier Detection

### Detection Methods

```python
def detect_outliers_iqr(data, multiplier=1.5):
    """IQR method - robust to distribution shape."""
    Q1 = data.quantile(0.25)
    Q3 = data.quantile(0.75)
    IQR = Q3 - Q1
    lower = Q1 - multiplier * IQR
    upper = Q3 + multiplier * IQR
    outliers = (data < lower) | (data > upper)
    return outliers, lower, upper

def detect_outliers_zscore(data, threshold=3):
    """Z-score method - assumes normality."""
    z_scores = np.abs(stats.zscore(data.dropna()))
    outliers = z_scores > threshold
    return outliers

def detect_outliers_isolation_forest(df, contamination=0.05):
    """Isolation Forest - works for multivariate outliers."""
    from sklearn.ensemble import IsolationForest

    numeric_cols = df.select_dtypes(include=[np.number]).columns
    X = df[numeric_cols].dropna()

    iso_forest = IsolationForest(contamination=contamination, random_state=42)
    outliers = iso_forest.fit_predict(X) == -1

    return outliers

def outlier_summary(df, column):
    """Comprehensive outlier analysis for a single column."""
    data = df[column].dropna()

    # IQR method
    iqr_outliers, lower, upper = detect_outliers_iqr(data)

    # Z-score method
    zscore_outliers = detect_outliers_zscore(data)

    print(f"Outlier Analysis for {column}:")
    print(f"  Total observations: {len(data):,}")
    print(f"  IQR outliers: {iqr_outliers.sum():,} ({iqr_outliers.mean()*100:.1f}%)")
    print(f"    Lower bound: {lower:.4f}")
    print(f"    Upper bound: {upper:.4f}")
    print(f"  Z-score outliers (|z|>3): {zscore_outliers.sum():,} ({zscore_outliers.mean()*100:.1f}%)")

    # Show outlier values
    outlier_values = data[iqr_outliers]
    if len(outlier_values) > 0 and len(outlier_values) <= 20:
        print(f"  Outlier values: {sorted(outlier_values.values)}")
```

### Outlier Handling Decision Tree

```
Is the outlier a data error?
├── Yes → Fix or remove
└── No → Is the outlier a valid extreme value?
    ├── Yes → Keep (maybe use robust methods)
    └── No/Unsure → Investigate further
        ├── Collect more data
        ├── Consult domain expert
        └── Run analysis with and without
```

---

## Visualization Best Practices

### Chart Selection Guide

| Data Type | Comparison | Distribution | Relationship | Composition |
|-----------|------------|--------------|--------------|-------------|
| **Numeric** | Bar, Lollipop | Histogram, Box, Violin | Scatter, Heatmap | Stacked Bar, Area |
| **Categorical** | Bar, Dot | Bar | Grouped Bar | Pie, Stacked Bar |
| **Time Series** | Line | - | Line with Markers | Stacked Area |

### Visualization Code Patterns

```python
import matplotlib.pyplot as plt
import seaborn as sns

# Set consistent style
plt.style.use('seaborn-v0_8-whitegrid')
sns.set_palette('colorblind')  # Accessible colors

def distribution_comparison(df, column, group_col):
    """Compare distributions across groups."""
    fig, axes = plt.subplots(1, 3, figsize=(15, 4))

    # Overlapping histograms
    for group in df[group_col].unique():
        data = df[df[group_col] == group][column]
        axes[0].hist(data, alpha=0.5, label=group, bins=30)
    axes[0].legend()
    axes[0].set_title(f'{column} by {group_col} (Histogram)')

    # Box plots
    sns.boxplot(data=df, x=group_col, y=column, ax=axes[1])
    axes[1].set_title(f'{column} by {group_col} (Box Plot)')

    # Violin plots
    sns.violinplot(data=df, x=group_col, y=column, ax=axes[2])
    axes[2].set_title(f'{column} by {group_col} (Violin Plot)')

    plt.tight_layout()

def time_series_plot(df, date_col, value_col, group_col=None):
    """Professional time series visualization."""
    fig, ax = plt.subplots(figsize=(14, 6))

    if group_col:
        for group in df[group_col].unique():
            group_data = df[df[group_col] == group]
            ax.plot(group_data[date_col], group_data[value_col], label=group)
        ax.legend()
    else:
        ax.plot(df[date_col], df[value_col])

    ax.set_xlabel('Date')
    ax.set_ylabel(value_col)
    ax.set_title(f'{value_col} Over Time')

    # Format x-axis dates
    fig.autofmt_xdate()
    plt.tight_layout()

def correlation_matrix_plot(df, figsize=(12, 10)):
    """Publication-ready correlation matrix."""
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    corr = df[numeric_cols].corr()

    # Mask upper triangle
    mask = np.triu(np.ones_like(corr, dtype=bool))

    fig, ax = plt.subplots(figsize=figsize)
    sns.heatmap(corr, mask=mask, annot=True, fmt='.2f',
                cmap='RdBu_r', center=0, vmin=-1, vmax=1,
                square=True, linewidths=0.5, ax=ax)
    ax.set_title('Correlation Matrix')
    plt.tight_layout()
```

### Visualization Anti-Patterns

| Anti-Pattern | Problem | Fix |
|--------------|---------|-----|
| Pie chart with many slices | Hard to compare | Use bar chart |
| 3D charts | Distorts perception | Use 2D |
| Dual y-axes | Confusing | Two separate charts |
| Missing axis labels | Uninterpretable | Always label axes |
| Truncated y-axis | Exaggerates differences | Start at 0 (usually) |
| Rainbow color maps | Not perceptually uniform | Use viridis, colorblind |
| No title or legend | Missing context | Always include |

---

## A/B Testing

### Sample Size Calculation

```python
from statsmodels.stats.power import TTestIndPower

def calculate_sample_size(baseline_rate, minimum_detectable_effect,
                          alpha=0.05, power=0.8):
    """
    Calculate required sample size per group.

    Args:
        baseline_rate: Current conversion rate (e.g., 0.10 for 10%)
        minimum_detectable_effect: Relative change to detect (e.g., 0.10 for 10% lift)
        alpha: Significance level (Type I error rate)
        power: 1 - Type II error rate
    """
    # Calculate effect size (Cohen's d for proportions)
    p1 = baseline_rate
    p2 = baseline_rate * (1 + minimum_detectable_effect)
    pooled_std = np.sqrt((p1 * (1-p1) + p2 * (1-p2)) / 2)
    effect_size = abs(p2 - p1) / pooled_std

    # Calculate sample size
    analysis = TTestIndPower()
    sample_size = analysis.solve_power(
        effect_size=effect_size,
        alpha=alpha,
        power=power,
        ratio=1.0,  # equal group sizes
        alternative='two-sided'
    )

    print(f"Baseline rate: {baseline_rate:.2%}")
    print(f"Target rate: {p2:.2%} ({minimum_detectable_effect:+.0%} relative)")
    print(f"Effect size (Cohen's d): {effect_size:.4f}")
    print(f"Required sample size per group: {int(np.ceil(sample_size)):,}")
    print(f"Total sample size: {int(np.ceil(sample_size) * 2):,}")

    return int(np.ceil(sample_size))

def analyze_ab_test(control_conversions, control_total,
                    treatment_conversions, treatment_total):
    """
    Analyze A/B test results.
    """
    # Conversion rates
    control_rate = control_conversions / control_total
    treatment_rate = treatment_conversions / treatment_total
    lift = (treatment_rate - control_rate) / control_rate

    # Chi-square test
    contingency = np.array([
        [control_conversions, control_total - control_conversions],
        [treatment_conversions, treatment_total - treatment_conversions]
    ])
    chi2, p_value, dof, expected = stats.chi2_contingency(contingency)

    # Confidence interval for difference
    se = np.sqrt(control_rate*(1-control_rate)/control_total +
                 treatment_rate*(1-treatment_rate)/treatment_total)
    ci_low = (treatment_rate - control_rate) - 1.96 * se
    ci_high = (treatment_rate - control_rate) + 1.96 * se

    print("A/B Test Results:")
    print(f"  Control:   {control_rate:.2%} ({control_conversions:,}/{control_total:,})")
    print(f"  Treatment: {treatment_rate:.2%} ({treatment_conversions:,}/{treatment_total:,})")
    print(f"  Lift:      {lift:+.2%}")
    print(f"  95% CI:    [{ci_low:+.2%}, {ci_high:+.2%}]")
    print(f"  p-value:   {p_value:.4f}")
    print(f"  Conclusion: {'Significant' if p_value < 0.05 else 'Not significant'}")

    return {
        'control_rate': control_rate,
        'treatment_rate': treatment_rate,
        'lift': lift,
        'p_value': p_value,
        'ci': (ci_low, ci_high)
    }
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| p-hacking | Inflates false positives | Pre-register analysis, adjust for multiple tests |
| Ignoring assumptions | Invalid test results | Check assumptions first |
| Correlation as causation | Misleading conclusions | Use experimental design or causal inference |
| Cherry-picking visualizations | Biased presentation | Show full picture, warts and all |
| Stopping A/B early | Inflated effect sizes | Pre-determine sample size |
| Removing outliers blindly | Losing valid information | Investigate first |
| Using mean for skewed data | Misleading summary | Use median or show distribution |
| Ignoring missing data | Biased analysis | Analyze missingness pattern |

---

## Quick Reference

### Common Statistical Tests Cheat Sheet

```python
from scipy import stats

# Normality
stats.shapiro(data)           # Shapiro-Wilk (n < 5000)
stats.normaltest(data)        # D'Agostino-Pearson

# Compare means
stats.ttest_1samp(data, mu)   # One-sample t-test
stats.ttest_ind(a, b)         # Two-sample t-test
stats.ttest_rel(a, b)         # Paired t-test
stats.f_oneway(a, b, c)       # One-way ANOVA

# Non-parametric
stats.mannwhitneyu(a, b)      # Mann-Whitney U
stats.wilcoxon(a, b)          # Wilcoxon signed-rank
stats.kruskal(a, b, c)        # Kruskal-Wallis

# Categorical
stats.chi2_contingency(table) # Chi-square
stats.fisher_exact(table)     # Fisher's exact (2x2)

# Correlation
stats.pearsonr(x, y)          # Pearson
stats.spearmanr(x, y)         # Spearman
stats.kendalltau(x, y)        # Kendall
```

### Effect Size Interpretation

| Measure | Small | Medium | Large |
|---------|-------|--------|-------|
| Cohen's d | 0.2 | 0.5 | 0.8 |
| Pearson r | 0.1 | 0.3 | 0.5 |
| Eta-squared | 0.01 | 0.06 | 0.14 |
| Odds ratio | 1.5 | 2.5 | 4.0 |

---

## References

- [Scipy Statistics Documentation](https://docs.scipy.org/doc/scipy/reference/stats.html)
- [Statsmodels Documentation](https://www.statsmodels.org/)
- [Seaborn Gallery](https://seaborn.pydata.org/examples/index.html)
- [Matplotlib Cheat Sheets](https://matplotlib.org/cheatsheets/)
- [Missing Data in Python (fancyimpute)](https://github.com/iskandr/fancyimpute)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-27
**Maintained By:** ALI AI Team
