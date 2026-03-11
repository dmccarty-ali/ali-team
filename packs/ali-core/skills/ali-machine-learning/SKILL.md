---
name: ali-machine-learning
description: |
  Machine learning patterns for predictive modeling and forecasting. Use when:

  PLANNING: Designing ML pipelines, selecting model architectures, planning
  feature engineering, choosing evaluation strategies, architecting training workflows

  IMPLEMENTATION: Building scikit-learn/XGBoost/PyTorch models, implementing
  feature pipelines, creating training loops, writing evaluation code, deploying models

  GUIDANCE: Asking about model selection, hyperparameter tuning, cross-validation,
  handling imbalanced data, time series forecasting, feature importance

  REVIEW: Checking model evaluation methodology, validating feature engineering,
  auditing training pipelines, reviewing prediction accuracy metrics

  Do NOT use for statistical analysis, EDA, or hypothesis testing (use ali-data-science
  instead), or data warehouse modeling and ETL pipelines (use ali-data-architecture instead)
---

# Machine Learning

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Selecting model types (regression, classification, clustering, time series)
- Designing feature engineering pipelines
- Planning cross-validation strategies
- Architecting ML training workflows
- Choosing evaluation metrics

**Implementation:**
- Building models with scikit-learn, XGBoost, LightGBM, PyTorch, TensorFlow
- Implementing feature transformations
- Creating training and evaluation loops
- Writing prediction pipelines
- Deploying models to production

**Guidance/Best Practices:**
- Asking about model selection for specific problems
- Asking about hyperparameter tuning approaches
- Asking about handling overfitting/underfitting
- Asking about time series forecasting methods

**Review/Validation:**
- Reviewing model evaluation methodology
- Checking for data leakage
- Validating feature engineering correctness
- Auditing prediction accuracy and metrics

---

## Key Principles

- **No free lunch**: No single algorithm works best for all problems - selection depends on data characteristics
- **Garbage in, garbage out**: Feature quality matters more than model complexity
- **Validation is everything**: Proper cross-validation prevents overfitting delusions
- **Simplicity first**: Start with simple models (linear regression, decision trees) before complex ones
- **Leakage kills**: Data leakage invalidates all results - be paranoid about train/test contamination
- **Metrics match goals**: Choose evaluation metrics that align with business objectives
- **Reproducibility required**: Set random seeds, version data, log hyperparameters

---

## Model Selection Guide

### By Problem Type

| Problem | First Try | If Insufficient | Complex Option |
|---------|-----------|-----------------|----------------|
| **Regression** | Linear Regression, Ridge | Random Forest, XGBoost | Neural Network |
| **Classification** | Logistic Regression | Random Forest, XGBoost | Neural Network |
| **Time Series** | ARIMA, Exponential Smoothing | Prophet, XGBoost | LSTM, Transformer |
| **Clustering** | K-Means | DBSCAN, Hierarchical | Gaussian Mixture |
| **Anomaly Detection** | Isolation Forest | One-Class SVM | Autoencoder |

### By Data Characteristics

| Characteristic | Recommended Approach |
|----------------|---------------------|
| Small dataset (<1K rows) | Linear models, regularization, simple trees |
| Medium dataset (1K-100K) | Random Forest, XGBoost, LightGBM |
| Large dataset (>100K) | Neural networks, gradient boosting |
| High dimensionality | Regularization (L1/L2), PCA, feature selection |
| Many categorical features | CatBoost, target encoding, embedding layers |
| Missing values | XGBoost (native), imputation + any model |
| Imbalanced classes | SMOTE, class weights, threshold tuning |

---

## Feature Engineering Patterns

### Numeric Features

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler
from sklearn.preprocessing import QuantileTransformer, PowerTransformer

# Standardization (mean=0, std=1) - most common
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)

# Robust to outliers (uses median and IQR)
robust_scaler = RobustScaler()
X_robust = robust_scaler.fit_transform(X_train)

# For skewed distributions
power_transformer = PowerTransformer(method='yeo-johnson')
X_normalized = power_transformer.fit_transform(X_train)

# Binning continuous variables
pd.cut(df['age'], bins=[0, 18, 35, 50, 65, 100], labels=['child', 'young', 'middle', 'senior', 'elderly'])
```

### Categorical Features

```python
from sklearn.preprocessing import OneHotEncoder, OrdinalEncoder, LabelEncoder

# One-hot encoding (use for nominal categories)
encoder = OneHotEncoder(sparse_output=False, handle_unknown='ignore')
X_encoded = encoder.fit_transform(df[['category_column']])

# Ordinal encoding (use for ordered categories)
ordinal_encoder = OrdinalEncoder(categories=[['low', 'medium', 'high']])
X_ordinal = ordinal_encoder.fit_transform(df[['priority']])

# Target encoding (for high-cardinality categoricals)
# Use with cross-validation to prevent leakage
from category_encoders import TargetEncoder
target_encoder = TargetEncoder(cols=['high_card_column'])
X_target_encoded = target_encoder.fit_transform(X_train, y_train)
```

### Time-Based Features

```python
# Extract temporal components
df['year'] = df['date'].dt.year
df['month'] = df['date'].dt.month
df['day_of_week'] = df['date'].dt.dayofweek
df['day_of_year'] = df['date'].dt.dayofyear
df['is_weekend'] = df['date'].dt.dayofweek >= 5
df['quarter'] = df['date'].dt.quarter

# Cyclical encoding for periodic features
import numpy as np
df['month_sin'] = np.sin(2 * np.pi * df['month'] / 12)
df['month_cos'] = np.cos(2 * np.pi * df['month'] / 12)

# Lag features for time series
df['value_lag_1'] = df['value'].shift(1)
df['value_lag_7'] = df['value'].shift(7)
df['value_rolling_mean_7'] = df['value'].rolling(window=7).mean()
df['value_rolling_std_7'] = df['value'].rolling(window=7).std()
```

### Feature Selection

```python
from sklearn.feature_selection import SelectKBest, f_regression, mutual_info_regression
from sklearn.feature_selection import RFE
from sklearn.ensemble import RandomForestRegressor

# Filter method: statistical tests
selector = SelectKBest(score_func=f_regression, k=10)
X_selected = selector.fit_transform(X, y)

# Wrapper method: recursive feature elimination
estimator = RandomForestRegressor(n_estimators=100, random_state=42)
rfe = RFE(estimator, n_features_to_select=10)
X_rfe = rfe.fit_transform(X, y)

# Embedded method: feature importance from tree models
rf = RandomForestRegressor(n_estimators=100, random_state=42)
rf.fit(X_train, y_train)
importance = pd.DataFrame({
    'feature': X_train.columns,
    'importance': rf.feature_importances_
}).sort_values('importance', ascending=False)
```

---

## Cross-Validation Strategies

### Standard K-Fold

```python
from sklearn.model_selection import cross_val_score, KFold

# Basic k-fold (use for IID data)
kfold = KFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=kfold, scoring='neg_mean_absolute_error')
print(f"MAE: {-scores.mean():.4f} (+/- {scores.std():.4f})")
```

### Stratified K-Fold (Classification)

```python
from sklearn.model_selection import StratifiedKFold

# Maintains class distribution in each fold
stratified_kfold = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=stratified_kfold, scoring='f1_weighted')
```

### Time Series Split (CRITICAL for temporal data)

```python
from sklearn.model_selection import TimeSeriesSplit

# Never use random shuffle for time series - causes data leakage!
tscv = TimeSeriesSplit(n_splits=5)
for train_idx, test_idx in tscv.split(X):
    X_train, X_test = X.iloc[train_idx], X.iloc[test_idx]
    y_train, y_test = y.iloc[train_idx], y.iloc[test_idx]
    # Train and evaluate
```

### Group K-Fold (Grouped data)

```python
from sklearn.model_selection import GroupKFold

# Use when samples from same group must stay together
# Example: multiple measurements from same fish/cage
group_kfold = GroupKFold(n_splits=5)
for train_idx, test_idx in group_kfold.split(X, y, groups=df['cage_id']):
    # All samples from a cage are in either train OR test, never both
    pass
```

---

## Evaluation Metrics

### Regression Metrics

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
from sklearn.metrics import mean_absolute_percentage_error

y_pred = model.predict(X_test)

# Mean Absolute Error - interpretable, robust to outliers
mae = mean_absolute_error(y_test, y_pred)

# Root Mean Squared Error - penalizes large errors more
rmse = np.sqrt(mean_squared_error(y_test, y_pred))

# Mean Absolute Percentage Error - scale-independent
mape = mean_absolute_percentage_error(y_test, y_pred) * 100  # as percentage

# R-squared - proportion of variance explained
r2 = r2_score(y_test, y_pred)

# Custom MAPE that handles zeros
def safe_mape(y_true, y_pred, epsilon=1e-10):
    """MAPE that handles zero values."""
    return np.mean(np.abs((y_true - y_pred) / (y_true + epsilon))) * 100
```

### Classification Metrics

```python
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score
from sklearn.metrics import confusion_matrix, classification_report, roc_auc_score

y_pred = model.predict(X_test)
y_proba = model.predict_proba(X_test)[:, 1]  # for binary

# Accuracy - use only for balanced classes
accuracy = accuracy_score(y_test, y_pred)

# Precision - of predicted positives, how many are correct
precision = precision_score(y_test, y_pred, average='weighted')

# Recall - of actual positives, how many did we find
recall = recall_score(y_test, y_pred, average='weighted')

# F1 - harmonic mean of precision and recall
f1 = f1_score(y_test, y_pred, average='weighted')

# AUC-ROC - probability ranking quality (binary only)
auc = roc_auc_score(y_test, y_proba)

# Full report
print(classification_report(y_test, y_pred))
```

### Metric Selection Guide

| Goal | Primary Metric | Why |
|------|----------------|-----|
| Minimize average error | MAE | Robust, interpretable in original units |
| Penalize large errors | RMSE | Sensitive to outliers, same units as target |
| Scale-independent error | MAPE | Comparable across different scales |
| Predict rare events | Recall, F1 | Accuracy misleading for imbalanced |
| Minimize false positives | Precision | When FP cost is high |
| Rank predictions | AUC-ROC | For probability outputs |

---

## Time Series Forecasting

### Classical Methods

```python
from statsmodels.tsa.arima.model import ARIMA
from statsmodels.tsa.holtwinters import ExponentialSmoothing

# ARIMA (AutoRegressive Integrated Moving Average)
model = ARIMA(y_train, order=(p, d, q))  # (AR, differencing, MA)
fitted = model.fit()
forecast = fitted.forecast(steps=30)

# Exponential Smoothing (Holt-Winters)
model = ExponentialSmoothing(
    y_train,
    trend='add',           # 'add', 'mul', or None
    seasonal='add',        # 'add', 'mul', or None
    seasonal_periods=12    # for monthly data with yearly seasonality
)
fitted = model.fit()
forecast = fitted.forecast(steps=30)
```

### Prophet (Facebook)

```python
from prophet import Prophet

# Prepare data (requires 'ds' and 'y' columns)
df_prophet = df.rename(columns={'date': 'ds', 'value': 'y'})

model = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=True,
    daily_seasonality=False,
    changepoint_prior_scale=0.05  # flexibility of trend
)

# Add regressors (external features)
model.add_regressor('temperature')
model.add_regressor('feeding_rate')

model.fit(df_prophet)

# Create future dataframe
future = model.make_future_dataframe(periods=30)
future['temperature'] = ...  # must provide regressor values
future['feeding_rate'] = ...

forecast = model.predict(future)
```

### ML Approach to Time Series

```python
import xgboost as xgb

def create_time_features(df, target_col, lags=[1, 7, 14, 30]):
    """Create lag and rolling features for time series."""
    result = df.copy()

    # Lag features
    for lag in lags:
        result[f'{target_col}_lag_{lag}'] = result[target_col].shift(lag)

    # Rolling statistics
    for window in [7, 14, 30]:
        result[f'{target_col}_rolling_mean_{window}'] = (
            result[target_col].shift(1).rolling(window=window).mean()
        )
        result[f'{target_col}_rolling_std_{window}'] = (
            result[target_col].shift(1).rolling(window=window).std()
        )

    # Time features
    result['day_of_week'] = result['date'].dt.dayofweek
    result['month'] = result['date'].dt.month
    result['day_of_year'] = result['date'].dt.dayofyear

    return result.dropna()

# Train XGBoost on engineered features
df_features = create_time_features(df, 'weight')
X = df_features.drop(['date', 'weight'], axis=1)
y = df_features['weight']

# Use time-based split
train_size = int(len(X) * 0.8)
X_train, X_test = X[:train_size], X[train_size:]
y_train, y_test = y[:train_size], y[train_size:]

model = xgb.XGBRegressor(
    n_estimators=100,
    max_depth=6,
    learning_rate=0.1,
    random_state=42
)
model.fit(X_train, y_train)
```

---

## Hyperparameter Tuning

### Grid Search

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    'n_estimators': [100, 200, 300],
    'max_depth': [3, 5, 7, 10],
    'learning_rate': [0.01, 0.1, 0.2],
    'min_child_weight': [1, 3, 5]
}

grid_search = GridSearchCV(
    xgb.XGBRegressor(random_state=42),
    param_grid,
    cv=5,
    scoring='neg_mean_absolute_error',
    n_jobs=-1,
    verbose=1
)

grid_search.fit(X_train, y_train)
print(f"Best params: {grid_search.best_params_}")
print(f"Best score: {-grid_search.best_score_:.4f}")
```

### Randomized Search (Faster)

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import uniform, randint

param_distributions = {
    'n_estimators': randint(50, 500),
    'max_depth': randint(3, 15),
    'learning_rate': uniform(0.01, 0.3),
    'min_child_weight': randint(1, 10),
    'subsample': uniform(0.6, 0.4),
    'colsample_bytree': uniform(0.6, 0.4)
}

random_search = RandomizedSearchCV(
    xgb.XGBRegressor(random_state=42),
    param_distributions,
    n_iter=100,  # number of combinations to try
    cv=5,
    scoring='neg_mean_absolute_error',
    n_jobs=-1,
    random_state=42
)

random_search.fit(X_train, y_train)
```

### Optuna (Bayesian Optimization)

```python
import optuna

def objective(trial):
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 50, 500),
        'max_depth': trial.suggest_int('max_depth', 3, 15),
        'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.3, log=True),
        'min_child_weight': trial.suggest_int('min_child_weight', 1, 10),
        'subsample': trial.suggest_float('subsample', 0.6, 1.0),
        'colsample_bytree': trial.suggest_float('colsample_bytree', 0.6, 1.0),
    }

    model = xgb.XGBRegressor(**params, random_state=42)

    # Use cross-validation
    scores = cross_val_score(
        model, X_train, y_train,
        cv=5,
        scoring='neg_mean_absolute_error'
    )
    return -scores.mean()

study = optuna.create_study(direction='minimize')
study.optimize(objective, n_trials=100)

print(f"Best params: {study.best_params}")
print(f"Best MAE: {study.best_value:.4f}")
```

---

## Handling Imbalanced Data

### Resampling Techniques

```python
from imblearn.over_sampling import SMOTE, ADASYN
from imblearn.under_sampling import RandomUnderSampler
from imblearn.combine import SMOTETomek

# Oversample minority class
smote = SMOTE(random_state=42)
X_resampled, y_resampled = smote.fit_resample(X_train, y_train)

# Undersample majority class (when you have lots of data)
rus = RandomUnderSampler(random_state=42)
X_resampled, y_resampled = rus.fit_resample(X_train, y_train)

# Combined approach
smt = SMOTETomek(random_state=42)
X_resampled, y_resampled = smt.fit_resample(X_train, y_train)
```

### Class Weights

```python
from sklearn.utils.class_weight import compute_class_weight

# Compute balanced weights
class_weights = compute_class_weight(
    'balanced',
    classes=np.unique(y_train),
    y=y_train
)
weight_dict = dict(zip(np.unique(y_train), class_weights))

# Use in model
model = xgb.XGBClassifier(scale_pos_weight=weight_dict[1]/weight_dict[0])
# Or for sklearn
model = RandomForestClassifier(class_weight='balanced')
```

### Threshold Tuning

```python
from sklearn.metrics import precision_recall_curve

y_proba = model.predict_proba(X_test)[:, 1]

# Find threshold that maximizes F1
precisions, recalls, thresholds = precision_recall_curve(y_test, y_proba)
f1_scores = 2 * (precisions * recalls) / (precisions + recalls + 1e-10)
best_threshold = thresholds[np.argmax(f1_scores[:-1])]

# Use custom threshold
y_pred_custom = (y_proba >= best_threshold).astype(int)
```

---

## Model Pipeline Pattern

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer

# Define feature groups
numeric_features = ['age', 'weight', 'temperature']
categorical_features = ['species', 'cage_location']

# Create preprocessing pipelines
numeric_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

categorical_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='constant', fill_value='missing')),
    ('encoder', OneHotEncoder(handle_unknown='ignore'))
])

# Combine preprocessors
preprocessor = ColumnTransformer(transformers=[
    ('num', numeric_transformer, numeric_features),
    ('cat', categorical_transformer, categorical_features)
])

# Full pipeline with model
pipeline = Pipeline(steps=[
    ('preprocessor', preprocessor),
    ('model', xgb.XGBRegressor(n_estimators=100, random_state=42))
])

# Fit entire pipeline
pipeline.fit(X_train, y_train)

# Predict (handles all preprocessing automatically)
y_pred = pipeline.predict(X_test)
```

---

## Common Data Leakage Patterns

| Leakage Type | Example | Prevention |
|--------------|---------|------------|
| **Target leakage** | Feature derived from target variable | Review feature derivation logic |
| **Train-test contamination** | Fit scaler on full data, then split | Fit only on training data |
| **Future information** | Using tomorrow's data to predict today | Strict temporal ordering |
| **Group leakage** | Same patient in train and test | Use GroupKFold |
| **Preprocessing leakage** | Impute missing with global mean | Impute within CV folds |

### Leakage Detection Checklist

- [ ] Features created BEFORE train/test split
- [ ] Scalers/encoders fit ONLY on training data
- [ ] Time series uses temporal split (no shuffle)
- [ ] No features that "know" the target
- [ ] Grouped data stays together (GroupKFold)
- [ ] Cross-validation mirrors production scenario

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Random split for time series | Future data leaks into training | Use TimeSeriesSplit |
| Accuracy for imbalanced data | Misleading (90% accuracy = always predict majority) | Use F1, precision, recall, AUC |
| Fit scaler on all data | Information leakage from test set | Fit only on training data |
| No cross-validation | Single split can be misleading | Use k-fold CV |
| Complex model first | Overfitting, slow iteration | Start simple, add complexity |
| Ignoring feature importance | Wasted compute, potential leakage | Analyze and prune features |
| Same random seed everywhere | Reproducible but may hide instability | Test with multiple seeds |
| MAPE with zero values | Division by zero, infinite errors | Use safe_mape or SMAPE |

---

## Quick Reference

### Scikit-learn Model Cheat Sheet

```python
# Regression
from sklearn.linear_model import LinearRegression, Ridge, Lasso, ElasticNet
from sklearn.tree import DecisionTreeRegressor
from sklearn.ensemble import RandomForestRegressor, GradientBoostingRegressor
from sklearn.svm import SVR

# Classification
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC
from sklearn.naive_bayes import GaussianNB

# Clustering
from sklearn.cluster import KMeans, DBSCAN, AgglomerativeClustering
from sklearn.mixture import GaussianMixture
```

### XGBoost Key Parameters

| Parameter | Description | Typical Range |
|-----------|-------------|---------------|
| n_estimators | Number of trees | 100-1000 |
| max_depth | Tree depth | 3-10 |
| learning_rate | Step size | 0.01-0.3 |
| min_child_weight | Min samples in leaf | 1-10 |
| subsample | Row sampling ratio | 0.6-1.0 |
| colsample_bytree | Column sampling ratio | 0.6-1.0 |
| reg_alpha | L1 regularization | 0-1 |
| reg_lambda | L2 regularization | 0-1 |

### Metric Formulas

```
MAE  = mean(|y_true - y_pred|)
RMSE = sqrt(mean((y_true - y_pred)^2))
MAPE = mean(|y_true - y_pred| / |y_true|) * 100
R^2  = 1 - SS_res / SS_tot

Precision = TP / (TP + FP)
Recall    = TP / (TP + FN)
F1        = 2 * (Precision * Recall) / (Precision + Recall)
```

---

## References

- [Scikit-learn Documentation](https://scikit-learn.org/stable/)
- [XGBoost Documentation](https://xgboost.readthedocs.io/)
- [Prophet Documentation](https://facebook.github.io/prophet/)
- [Optuna Documentation](https://optuna.org/)
- [Imbalanced-learn Documentation](https://imbalanced-learn.org/)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-27
**Maintained By:** ALI AI Team
