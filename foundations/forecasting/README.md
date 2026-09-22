# Cloud Cost Forecasting

Accurate cloud cost forecasting enables proactive financial management, better budget planning, and earlier detection of cost trends. Unlike traditional IT where costs were known at procurement time, cloud costs require continuous forecasting based on usage patterns, business growth, and planned changes.

---

## Forecasting Methods

### 1. Trend-Based Forecasting

The simplest approach: extrapolate historical spend trends into the future.

**When to use:**
- Stable workloads with consistent growth patterns
- Short-term forecasts (1–3 months)
- When business driver data is unavailable

**Method:**
```python
import numpy as np
from datetime import datetime, timedelta

def trend_forecast(
    historical_costs: list[float],  # Monthly costs, oldest first
    months_ahead: int = 3
) -> list[float]:
    """
    Simple linear trend forecast using least squares regression.
    """
    n = len(historical_costs)
    x = np.arange(n)
    y = np.array(historical_costs)
    
    # Fit linear trend
    coeffs = np.polyfit(x, y, 1)
    slope, intercept = coeffs
    
    # Project forward
    forecasts = []
    for i in range(months_ahead):
        forecast = slope * (n + i) + intercept
        forecasts.append(max(0, forecast))  # Cost can't be negative
    
    return forecasts

# Example
historical = [45000, 47200, 49800, 51000, 53400, 55200]
forecast = trend_forecast(historical, months_ahead=3)
print(f"3-month forecast: {[f'${f:,.0f}' for f in forecast]}")
# Output: ['$57,400', '$59,600', '$61,800']
```

**Limitations:**
- Assumes the future looks like the past
- Doesn't account for planned changes (new product launches, migrations)
- Poor accuracy for seasonal workloads

### 2. Driver-Based Forecasting

Connect cloud costs to business drivers (users, transactions, data volume) and forecast based on business projections.

**When to use:**
- Workloads with clear cost-to-business-metric relationships
- Medium-term forecasts (3–12 months)
- When business growth projections are available

**Method:**
```python
class DriverBasedForecast:
    """
    Forecast cloud costs based on business driver projections.
    
    Model: Total Cost = Fixed Cost + (Variable Cost Rate × Driver Volume)
    """
    
    def __init__(self, historical_data: list[dict]):
        """
        historical_data: list of {month, cost, driver_value} dicts
        """
        self.historical = historical_data
        self._fit_model()
    
    def _fit_model(self):
        """Fit linear model: cost = fixed + rate * driver."""
        import numpy as np
        
        drivers = np.array([d['driver_value'] for d in self.historical])
        costs = np.array([d['cost'] for d in self.historical])
        
        # Add constant for fixed cost
        X = np.column_stack([np.ones(len(drivers)), drivers])
        
        # Least squares fit
        result = np.linalg.lstsq(X, costs, rcond=None)
        self.fixed_cost, self.variable_rate = result[0]
    
    def forecast(self, projected_driver_values: list[float]) -> list[dict]:
        """Generate cost forecast for projected driver values."""
        forecasts = []
        for driver_value in projected_driver_values:
            cost = self.fixed_cost + (self.variable_rate * driver_value)
            forecasts.append({
                'driver_value': driver_value,
                'forecasted_cost': max(0, cost),
                'fixed_component': self.fixed_cost,
                'variable_component': self.variable_rate * driver_value,
                'cost_per_driver_unit': cost / driver_value if driver_value > 0 else 0
            })
        return forecasts

# Example: Forecast based on MAU growth
historical_data = [
    {'month': '2024-01', 'cost': 45000, 'driver_value': 20000},
    {'month': '2024-02', 'cost': 48000, 'driver_value': 22000},
    {'month': '2024-03', 'cost': 52000, 'driver_value': 25000},
    {'month': '2024-04', 'cost': 55000, 'driver_value': 27000},
    {'month': '2024-05', 'cost': 59000, 'driver_value': 30000},
]

model = DriverBasedForecast(historical_data)

# Project for next 3 months (MAU growth: 33K, 37K, 42K)
projections = model.forecast([33000, 37000, 42000])
for p in projections:
    print(f"MAU: {p['driver_value']:,} → Forecast: ${p['forecasted_cost']:,.0f} "
          f"(${p['cost_per_driver_unit']:.2f}/MAU)")
```

### 3. ML-Based Forecasting

Use machine learning models to capture complex patterns including seasonality, trends, and anomalies.

**When to use:**
- Complex workloads with seasonal patterns
- Long-term forecasts (6–24 months)
- When you have 12+ months of historical data

**Using AWS Forecast (Amazon Forecast):**
```python
import boto3
import json
from datetime import datetime

forecast_client = boto3.client('forecast')
s3_client = boto3.client('s3')

def create_cost_forecast(
    dataset_arn: str,
    forecast_horizon: int = 12,  # months
    forecast_frequency: str = 'M'  # Monthly
) -> str:
    """Create a cost forecast using Amazon Forecast."""
    
    # Create predictor
    predictor = forecast_client.create_auto_predictor(
        PredictorName=f'cloud-cost-predictor-{datetime.now().strftime("%Y%m%d")}',
        ForecastHorizon=forecast_horizon,
        ForecastFrequency=forecast_frequency,
        DataConfig={
            'DatasetGroupArn': dataset_arn
        },
        OptimizationMetric='WAPE',  # Weighted Absolute Percentage Error
        ExplainPredictor=True
    )
    
    return predictor['PredictorArn']

def get_forecast_accuracy(predictor_arn: str) -> dict:
    """Retrieve forecast accuracy metrics."""
    metrics = forecast_client.get_accuracy_metrics(
        PredictorArn=predictor_arn
    )
    
    wape = metrics['PredictorEvaluationResults'][0]['TestWindows'][0]['Metrics']['WeightedQuantileLosses'][0]['LossValue']
    
    return {
        'WAPE': wape,
        'accuracy_pct': (1 - wape) * 100
    }
```

**Using Prophet (open source):**
```python
from prophet import Prophet
import pandas as pd

def forecast_with_prophet(
    cost_data: pd.DataFrame,  # columns: ds (date), y (cost)
    periods: int = 12,
    include_holidays: bool = True
) -> pd.DataFrame:
    """
    Forecast cloud costs using Facebook Prophet.
    Handles seasonality, trends, and holidays automatically.
    """
    
    model = Prophet(
        yearly_seasonality=True,
        weekly_seasonality=False,  # Monthly data doesn't need weekly
        daily_seasonality=False,
        changepoint_prior_scale=0.05,  # Flexibility of trend changes
        seasonality_prior_scale=10.0
    )
    
    # Add custom seasonality for cloud spending patterns
    # (e.g., higher spend at end of quarter for new projects)
    model.add_seasonality(
        name='quarterly',
        period=91.25,
        fourier_order=5
    )
    
    model.fit(cost_data)
    
    future = model.make_future_dataframe(periods=periods, freq='M')
    forecast = model.predict(future)
    
    return forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].tail(periods)
```

---

## AWS Cost Explorer Forecasting

AWS Cost Explorer provides built-in forecasting based on your historical usage patterns.

### Accessing Forecasts

```python
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce')

def get_cost_forecast(
    start_date: str,  # YYYY-MM-DD
    end_date: str,
    granularity: str = 'MONTHLY',
    filter_tags: dict = None
) -> dict:
    """Get cost forecast from AWS Cost Explorer."""
    
    params = {
        'TimePeriod': {
            'Start': start_date,
            'End': end_date
        },
        'Metric': 'UNBLENDED_COST',
        'Granularity': granularity,
        'PredictionIntervalLevel': 95  # 95% confidence interval
    }
    
    if filter_tags:
        params['Filter'] = {
            'Tags': {
                'Key': filter_tags['key'],
                'Values': filter_tags['values']
            }
        }
    
    response = ce.get_cost_forecast(**params)
    
    total_forecast = float(response['Total']['Amount'])
    lower_bound = float(response['Total']['Amount']) * 0.9  # Approximate
    upper_bound = float(response['Total']['Amount']) * 1.1
    
    monthly_forecasts = [
        {
            'period': result['TimePeriod']['Start'],
            'mean': float(result['MeanValue']),
            'lower': float(result['PredictionIntervalLowerBound']),
            'upper': float(result['PredictionIntervalUpperBound'])
        }
        for result in response['ForecastResultsByTime']
    ]
    
    return {
        'total_forecast': total_forecast,
        'monthly_forecasts': monthly_forecasts
    }

# Get 3-month forecast for platform team
forecast = get_cost_forecast(
    start_date='2024-04-01',
    end_date='2024-07-01',
    filter_tags={'key': 'team', 'values': ['platform']}
)
```

**Cost Explorer forecast accuracy:**
- Typically ±10–15% for stable workloads
- Less accurate for rapidly growing or changing workloads
- Improves with more historical data (12+ months recommended)

---

## Azure Cost Management Forecasting

Azure Cost Management provides forecast views in the portal and via API:

```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.costmanagement import CostManagementClient
from azure.mgmt.costmanagement.models import (
    ForecastDefinition, ForecastType, TimeframeType, ForecastTimePeriod
)

def get_azure_forecast(subscription_id: str, months_ahead: int = 3):
    """Get Azure cost forecast for a subscription."""
    
    credential = DefaultAzureCredential()
    client = CostManagementClient(credential)
    
    scope = f"/subscriptions/{subscription_id}"
    
    forecast_def = ForecastDefinition(
        type=ForecastType.ACTUAL_COST,
        timeframe=TimeframeType.CUSTOM,
        time_period=ForecastTimePeriod(
            from_property="2024-01-01T00:00:00Z",
            to="2024-12-31T23:59:59Z"
        ),
        dataset={
            "granularity": "Monthly",
            "aggregation": {
                "totalCost": {
                    "name": "PreTaxCost",
                    "function": "Sum"
                }
            }
        },
        include_actual_cost=True,
        include_fresh_partial_cost=False
    )
    
    result = client.forecast.usage(scope=scope, parameters=forecast_def)
    return result
```

---

## GCP Cost Forecasting

GCP provides forecasting through the Billing API and BigQuery:

```sql
-- BigQuery: Forecast GCP costs using historical billing data
-- Requires billing export to BigQuery

WITH monthly_costs AS (
  SELECT
    DATE_TRUNC(usage_start_time, MONTH) AS month,
    SUM(cost) AS total_cost,
    SUM(cost) / COUNT(DISTINCT project.id) AS cost_per_project
  FROM `billing_dataset.gcp_billing_export_v1_*`
  WHERE usage_start_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12 MONTH)
  GROUP BY 1
),
trend AS (
  SELECT
    month,
    total_cost,
    AVG(total_cost) OVER (
      ORDER BY month
      ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3m,
    -- Calculate month-over-month growth rate
    (total_cost - LAG(total_cost) OVER (ORDER BY month)) / 
      LAG(total_cost) OVER (ORDER BY month) AS mom_growth_rate
  FROM monthly_costs
)
SELECT
  month,
  total_cost AS actual_cost,
  moving_avg_3m,
  AVG(mom_growth_rate) OVER (ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS avg_growth_rate
FROM trend
ORDER BY month DESC
LIMIT 6;
```

---

## Forecasting for New Workloads

New workloads have no historical data, requiring a different approach:

### Estimation Framework

```python
class NewWorkloadEstimator:
    """Estimate cloud costs for new workloads before deployment."""
    
    def __init__(self, cloud_provider: str = 'aws', region: str = 'us-east-1'):
        self.provider = cloud_provider
        self.region = region
    
    def estimate_web_application(
        self,
        expected_rps: float,           # Requests per second (average)
        peak_rps_multiplier: float,    # Peak vs average (e.g., 3x)
        avg_response_time_ms: float,   # Average response time
        data_storage_gb: float,        # Initial data storage
        data_growth_gb_per_month: float,
        cdn_traffic_gb_per_month: float
    ) -> dict:
        """Estimate monthly cost for a web application."""
        
        # Compute: size for peak load with 50% headroom
        peak_rps = expected_rps * peak_rps_multiplier
        # Assume each vCPU handles ~100 RPS at target response time
        vcpus_needed = (peak_rps / 100) * 1.5  # 50% headroom
        
        # Rough EC2 cost: $0.05/vCPU/hour for t3/m6i family
        compute_monthly = vcpus_needed * 0.05 * 730
        
        # Storage
        storage_monthly = data_storage_gb * 0.023  # S3 standard
        
        # Database (RDS): assume 2x storage, db.t3.medium
        db_monthly = 50 + (data_storage_gb * 2 * 0.115)  # Instance + storage
        
        # CDN
        cdn_monthly = cdn_traffic_gb_per_month * 0.0085  # CloudFront
        
        # Data transfer (10% of CDN traffic as origin egress)
        transfer_monthly = (cdn_traffic_gb_per_month * 0.1) * 0.09
        
        total_monthly = compute_monthly + storage_monthly + db_monthly + cdn_monthly + transfer_monthly
        
        return {
            'compute': compute_monthly,
            'storage': storage_monthly,
            'database': db_monthly,
            'cdn': cdn_monthly,
            'data_transfer': transfer_monthly,
            'total_monthly': total_monthly,
            'total_annual': total_monthly * 12,
            'assumptions': {
                'vcpus': vcpus_needed,
                'peak_rps': peak_rps
            }
        }

# Example: Estimate for a new API service
estimator = NewWorkloadEstimator()
estimate = estimator.estimate_web_application(
    expected_rps=50,
    peak_rps_multiplier=4,
    avg_response_time_ms=100,
    data_storage_gb=500,
    data_growth_gb_per_month=50,
    cdn_traffic_gb_per_month=10000
)
print(f"Estimated monthly cost: ${estimate['total_monthly']:,.0f}")
```

---

## Seasonal Patterns and Anomalies

### Identifying Seasonal Patterns

```python
import pandas as pd
import numpy as np

def analyze_seasonality(monthly_costs: pd.Series) -> dict:
    """
    Analyze seasonal patterns in cloud costs.
    Returns seasonal indices for each month.
    """
    # Calculate 12-month moving average (trend)
    trend = monthly_costs.rolling(window=12, center=True).mean()
    
    # Detrend: ratio of actual to trend
    detrended = monthly_costs / trend
    
    # Calculate seasonal index for each month
    monthly_indices = {}
    for month in range(1, 13):
        month_values = detrended[detrended.index.month == month].dropna()
        monthly_indices[month] = month_values.mean()
    
    return {
        'seasonal_indices': monthly_indices,
        'peak_month': max(monthly_indices, key=monthly_indices.get),
        'trough_month': min(monthly_indices, key=monthly_indices.get),
        'seasonality_strength': max(monthly_indices.values()) / min(monthly_indices.values())
    }
```

**Common cloud cost seasonal patterns:**

| Pattern | Cause | Typical Impact |
|---|---|---|
| Q4 spike | Holiday traffic, year-end projects | +20–50% |
| January dip | Post-holiday traffic drop | -10–20% |
| End-of-quarter spike | New project launches, budget spending | +5–15% |
| Summer dip (B2B) | Reduced business activity | -5–10% |
| Black Friday/Cyber Monday | E-commerce traffic peak | +100–500% |

---

## Forecast Accuracy Metrics

Track these metrics to measure and improve forecast quality:

### Key Accuracy Metrics

```python
def calculate_forecast_accuracy(
    actuals: list[float],
    forecasts: list[float]
) -> dict:
    """Calculate standard forecast accuracy metrics."""
    
    import numpy as np
    
    actuals = np.array(actuals)
    forecasts = np.array(forecasts)
    errors = actuals - forecasts
    
    # Mean Absolute Error
    mae = np.mean(np.abs(errors))
    
    # Mean Absolute Percentage Error
    mape = np.mean(np.abs(errors / actuals)) * 100
    
    # Root Mean Square Error
    rmse = np.sqrt(np.mean(errors ** 2))
    
    # Weighted Absolute Percentage Error (used by AWS Forecast)
    wape = np.sum(np.abs(errors)) / np.sum(actuals) * 100
    
    # Bias (positive = over-forecast, negative = under-forecast)
    bias = np.mean(errors / actuals) * 100
    
    return {
        'MAE': mae,
        'MAPE': mape,
        'RMSE': rmse,
        'WAPE': wape,
        'Bias_pct': bias,
        'accuracy_pct': 100 - mape
    }
```

**Accuracy targets by forecast horizon:**

| Horizon | MAPE Target | Notes |
|---|---|---|
| 1 month | < 5% | Should be highly accurate |
| 3 months | < 10% | Acceptable range |
| 6 months | < 15% | Directionally correct |
| 12 months | < 20% | Order of magnitude accuracy |

---

## AI-Assisted Forecasting Tools

| Tool | Provider | Approach | Best For |
|---|---|---|---|
| AWS Cost Explorer | AWS | Statistical ML | AWS-native, quick setup |
| Amazon Forecast | AWS | AutoML (DeepAR+, Prophet, etc.) | Custom ML forecasting |
| Azure Cost Management | Azure | Statistical | Azure-native |
| GCP Billing Forecasts | GCP | Statistical | GCP-native |
| Apptio Cloudability | Third-party | ML + anomaly detection | Enterprise multi-cloud |
| CloudHealth | VMware | Statistical + ML | Multi-cloud |
| Spot.io | NetApp | ML-based | Compute optimization focus |
| Anodot | Third-party | Unsupervised ML | Anomaly detection + forecast |

### Choosing a Forecasting Tool

```
Single cloud, < $500K/month spend:
  → Use native cloud provider forecasting (free, good enough)

Multi-cloud or > $500K/month:
  → Evaluate Apptio Cloudability or CloudHealth

Need custom ML models:
  → Amazon Forecast or Prophet (open source)

Need anomaly detection + forecasting:
  → Anodot or AWS Cost Anomaly Detection + Cost Explorer
```
