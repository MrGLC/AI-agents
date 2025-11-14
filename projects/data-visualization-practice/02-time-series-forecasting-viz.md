# Project 2: Time Series Forecasting Visualization

## Overview
Create an interactive visualization tool for time series forecasting that displays historical data, predictions, confidence intervals, and forecast accuracy metrics.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Work with time series data
- Visualize temporal patterns and trends
- Display prediction intervals and uncertainty
- Create interactive date range selectors
- Compare multiple forecasting models
- Analyze forecast errors and residuals

## Technical Stack
- **Backend**: Python
- **ML Framework**: Prophet, ARIMA (statsmodels), or LSTM (TensorFlow/PyTorch)
- **Visualization**: Plotly (for interactivity)
- **Dashboard**: Streamlit or Dash
- **Data Processing**: Pandas, NumPy
- **Dataset**: Stock prices, weather data, sales data, or COVID-19 cases

## Project Requirements

### 1. Data Loading & Preparation
- Load time series data from CSV or API
- Handle missing values and outliers
- Support multiple time series (e.g., multiple products, stocks)
- Date range filtering

### 2. Core Visualizations
- **Time Series Plot**: Historical data with trend line
- **Forecast Plot**: Predictions with confidence intervals
- **Decomposition**: Trend, seasonality, and residuals
- **Forecast vs Actual**: Comparison plot
- **Error Metrics**: MAE, RMSE, MAPE displayed prominently
- **Residual Analysis**: Residual plot and distribution

### 3. Interactive Features
- Date range selector
- Forecast horizon slider (7, 30, 90 days)
- Model selector (Prophet, ARIMA, LSTM)
- Zoom and pan on time series
- Hover tooltips with exact values
- Toggle components (trend, seasonality, predictions)

### 4. Advanced Analytics
- Seasonality detection and visualization
- Trend analysis with annotations
- Anomaly detection highlighting
- Forecast accuracy by time period
- Model comparison metrics

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas plotly prophet statsmodels scikit-learn
```

### Step 2: Load and Prepare Data
```python
import pandas as pd
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# Load data
df = pd.read_csv('time_series_data.csv', parse_dates=['date'])
df = df.sort_values('date')
```

### Step 3: Implement Forecasting Model
```python
from prophet import Prophet

def forecast_prophet(df, periods=30):
    model = Prophet(
        daily_seasonality=True,
        yearly_seasonality=True
    )
    model.fit(df[['ds', 'y']])

    future = model.make_future_dataframe(periods=periods)
    forecast = model.predict(future)

    return forecast
```

### Step 4: Create Interactive Visualizations
```python
import streamlit as st

st.title('📈 Time Series Forecasting Dashboard')

# Sidebar controls
forecast_days = st.sidebar.slider('Forecast Horizon (days)', 7, 90, 30)
model_type = st.sidebar.selectbox('Model', ['Prophet', 'ARIMA', 'LSTM'])

# Main visualization
fig = go.Figure()

# Historical data
fig.add_trace(go.Scatter(
    x=df['date'], y=df['value'],
    name='Historical',
    line=dict(color='blue')
))

# Forecast
fig.add_trace(go.Scatter(
    x=forecast['ds'], y=forecast['yhat'],
    name='Forecast',
    line=dict(color='red', dash='dash')
))

# Confidence interval
fig.add_trace(go.Scatter(
    x=forecast['ds'],
    y=forecast['yhat_upper'],
    fill=None,
    mode='lines',
    line_color='rgba(255,0,0,0.1)',
    showlegend=False
))

fig.add_trace(go.Scatter(
    x=forecast['ds'],
    y=forecast['yhat_lower'],
    fill='tonexty',
    mode='lines',
    line_color='rgba(255,0,0,0.1)',
    name='Confidence Interval'
))

st.plotly_chart(fig)
```

### Step 5: Add Metrics and Comparisons
```python
from sklearn.metrics import mean_absolute_error, mean_squared_error

# Calculate metrics
mae = mean_absolute_error(actual, predicted)
rmse = mean_squared_error(actual, predicted, squared=False)
mape = np.mean(np.abs((actual - predicted) / actual)) * 100

# Display metrics
col1, col2, col3 = st.columns(3)
col1.metric('MAE', f'{mae:.2f}')
col2.metric('RMSE', f'{rmse:.2f}')
col3.metric('MAPE', f'{mape:.2f}%')
```

## Expected Outputs

1. **Main Dashboard** with:
   - Interactive time series plot
   - Forecast with confidence intervals
   - Date range selector

2. **Decomposition View**:
   - Trend component
   - Seasonal component
   - Residuals

3. **Accuracy Metrics**:
   - MAE, RMSE, MAPE cards
   - Error over time plot
   - Residual distribution histogram

4. **Model Comparison**:
   - Side-by-side forecasts
   - Comparative metrics table

## Bonus Challenges

- [ ] Add multiple time series comparison
- [ ] Implement automatic anomaly detection with highlighting
- [ ] Add export to CSV/Excel functionality
- [ ] Include external regressors (holidays, events)
- [ ] Real-time data updates (via API)
- [ ] Backtesting visualization
- [ ] Add changepoint detection
- [ ] Create downloadable forecast report
- [ ] Implement ensemble forecasting

## Resources

- [Prophet Documentation](https://facebook.github.io/prophet/)
- [Plotly Time Series](https://plotly.com/python/time-series/)
- [Statsmodels ARIMA](https://www.statsmodels.org/stable/generated/statsmodels.tsa.arima.model.ARIMA.html)
- [Time Series in Streamlit](https://docs.streamlit.io/knowledge-base/tutorials/databases)

## Success Criteria

- Can load and visualize time series data
- Forecasting model produces reasonable predictions
- Interactive controls work smoothly
- Confidence intervals displayed correctly
- At least 3 accuracy metrics shown
- Visualizations update in real-time based on user input
