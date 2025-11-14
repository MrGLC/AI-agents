# Project 6: Real-Time Data Stream Visualizer

## Overview
Build a real-time data visualization dashboard that monitors streaming data, displays live metrics, detects anomalies, and provides alerts for critical events.

## Difficulty Level
Advanced

## Learning Objectives
- Handle real-time data streams
- Create auto-updating visualizations
- Implement anomaly detection algorithms
- Build live monitoring dashboards
- Manage state in streaming applications
- Implement alerting mechanisms

## Technical Stack
- **Backend**: Python, WebSockets or Server-Sent Events
- **ML Framework**: scikit-learn (anomaly detection)
- **Visualization**: Plotly (with live updates)
- **Dashboard**: Streamlit (with auto-refresh) or Dash (callbacks)
- **Data Generation**: Random/simulated or real API (Twitter, Stock market, IoT sensors)
- **Storage**: Redis or in-memory buffer

## Project Requirements

### 1. Data Streaming
- Generate or fetch real-time data (simulated sensor data, stock prices, etc.)
- Buffer recent data points (last N minutes)
- Handle missing data and connection errors
- Configurable update frequency

### 2. Live Visualizations
- **Time Series Plot**: Scrolling/updating line chart
- **Gauge Charts**: Current values (speed, temperature, etc.)
- **Metrics Cards**: Latest values with trend indicators
- **Histogram**: Distribution of recent values
- **Status Indicators**: Health/alert status
- **Event Log**: Recent events and anomalies

### 3. Anomaly Detection
- Statistical methods (Z-score, IQR)
- ML-based (Isolation Forest, One-Class SVM)
- Threshold-based alerts
- Visual highlighting of anomalies

### 4. Interactive Controls
- Start/stop streaming
- Adjust update frequency
- Set alert thresholds
- Select metrics to display
- Export historical data

### 5. Alerting System
- Trigger alerts on anomalies
- Visual notifications
- Sound alerts (optional)
- Alert history log

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy plotly scikit-learn redis
```

### Step 2: Create Data Stream Simulator
```python
import numpy as np
import pandas as pd
from datetime import datetime, timedelta
import time

class DataStreamSimulator:
    """Simulates real-time sensor data"""

    def __init__(self, noise_level=0.1, anomaly_rate=0.05):
        self.noise_level = noise_level
        self.anomaly_rate = anomaly_rate
        self.base_value = 100

    def generate_point(self):
        """Generate a single data point"""
        timestamp = datetime.now()

        # Normal pattern with some seasonality
        hour_factor = np.sin(timestamp.hour * 2 * np.pi / 24)
        base = self.base_value + 20 * hour_factor

        # Add noise
        noise = np.random.randn() * self.noise_level * base

        # Occasionally inject anomalies
        if np.random.rand() < self.anomaly_rate:
            value = base + np.random.choice([-1, 1]) * np.random.uniform(50, 100)
            is_anomaly = True
        else:
            value = base + noise
            is_anomaly = False

        return {
            'timestamp': timestamp,
            'value': value,
            'is_anomaly': is_anomaly
        }

    def generate_batch(self, n=10):
        """Generate multiple points"""
        return [self.generate_point() for _ in range(n)]
```

### Step 3: Anomaly Detection
```python
from sklearn.ensemble import IsolationForest
from collections import deque

class AnomalyDetector:
    """Detect anomalies in streaming data"""

    def __init__(self, window_size=100, contamination=0.1):
        self.window_size = window_size
        self.buffer = deque(maxlen=window_size)
        self.model = IsolationForest(contamination=contamination, random_state=42)
        self.is_trained = False

    def add_point(self, value):
        """Add new data point to buffer"""
        self.buffer.append(value)

        # Train model when buffer is full
        if len(self.buffer) >= self.window_size and not self.is_trained:
            X = np.array(self.buffer).reshape(-1, 1)
            self.model.fit(X)
            self.is_trained = True

    def detect(self, value):
        """Detect if value is anomaly"""
        if not self.is_trained:
            return False

        prediction = self.model.predict([[value]])
        return prediction[0] == -1  # -1 indicates anomaly
```

### Step 4: Build Real-Time Dashboard
```python
import streamlit as st
import plotly.graph_objects as go
from collections import deque
import time

st.set_page_config(page_title='Real-Time Monitor', layout='wide')

st.title('📊 Real-Time Data Monitor')

# Initialize session state
if 'data_buffer' not in st.session_state:
    st.session_state.data_buffer = deque(maxlen=500)
    st.session_state.simulator = DataStreamSimulator()
    st.session_state.detector = AnomalyDetector()
    st.session_state.streaming = False
    st.session_state.alerts = []

# Sidebar controls
st.sidebar.header('Controls')
update_freq = st.sidebar.slider('Update Frequency (seconds)', 0.1, 5.0, 1.0)
alert_threshold = st.sidebar.slider('Alert Threshold', 50, 200, 150)

if st.sidebar.button('Start Streaming' if not st.session_state.streaming else 'Stop Streaming'):
    st.session_state.streaming = not st.session_state.streaming

# Metrics display
col1, col2, col3, col4 = st.columns(4)

# Create placeholders
chart_placeholder = st.empty()
metrics_placeholder = st.empty()
alerts_placeholder = st.empty()

# Streaming loop
while st.session_state.streaming:
    # Generate new data point
    point = st.session_state.simulator.generate_point()
    st.session_state.data_buffer.append(point)

    # Detect anomaly
    st.session_state.detector.add_point(point['value'])
    is_anomaly = st.session_state.detector.detect(point['value'])

    if is_anomaly or point['value'] > alert_threshold:
        st.session_state.alerts.append({
            'timestamp': point['timestamp'],
            'value': point['value'],
            'type': 'Anomaly' if is_anomaly else 'Threshold'
        })

    # Prepare data for visualization
    if len(st.session_state.data_buffer) > 0:
        df = pd.DataFrame(list(st.session_state.data_buffer))

        # Create time series chart
        fig = go.Figure()

        # Normal points
        normal_df = df[~df['is_anomaly']]
        fig.add_trace(go.Scatter(
            x=normal_df['timestamp'],
            y=normal_df['value'],
            mode='lines+markers',
            name='Normal',
            line=dict(color='blue')
        ))

        # Anomaly points
        anomaly_df = df[df['is_anomaly']]
        if len(anomaly_df) > 0:
            fig.add_trace(go.Scatter(
                x=anomaly_df['timestamp'],
                y=anomaly_df['value'],
                mode='markers',
                name='Anomaly',
                marker=dict(color='red', size=10, symbol='x')
            ))

        # Alert threshold line
        fig.add_hline(y=alert_threshold, line_dash='dash',
                     line_color='orange', annotation_text='Alert Threshold')

        fig.update_layout(
            title='Real-Time Data Stream',
            xaxis_title='Time',
            yaxis_title='Value',
            height=400,
            showlegend=True
        )

        chart_placeholder.plotly_chart(fig, use_container_width=True)

        # Update metrics
        current_value = df.iloc[-1]['value']
        avg_value = df['value'].mean()
        max_value = df['value'].max()
        min_value = df['value'].min()

        with metrics_placeholder.container():
            col1, col2, col3, col4 = st.columns(4)
            col1.metric('Current', f'{current_value:.1f}')
            col2.metric('Average', f'{avg_value:.1f}')
            col3.metric('Max', f'{max_value:.1f}')
            col4.metric('Min', f'{min_value:.1f}')

        # Display recent alerts
        if len(st.session_state.alerts) > 0:
            with alerts_placeholder.container():
                st.subheader('🚨 Recent Alerts')
                recent_alerts = st.session_state.alerts[-5:]  # Last 5 alerts
                for alert in reversed(recent_alerts):
                    st.warning(f"{alert['type']} at {alert['timestamp'].strftime('%H:%M:%S')}: Value = {alert['value']:.1f}")

    # Wait before next update
    time.sleep(update_freq)
    st.rerun()
```

### Step 5: Add Statistical Dashboard
```python
st.subheader('📈 Statistical Analysis')

if len(st.session_state.data_buffer) > 0:
    df = pd.DataFrame(list(st.session_state.data_buffer))

    col1, col2 = st.columns(2)

    with col1:
        # Distribution histogram
        fig_hist = go.Figure()
        fig_hist.add_trace(go.Histogram(
            x=df['value'],
            nbinsx=30,
            name='Distribution'
        ))
        fig_hist.update_layout(title='Value Distribution')
        st.plotly_chart(fig_hist, use_container_width=True)

    with col2:
        # Rolling statistics
        df['rolling_mean'] = df['value'].rolling(window=20).mean()
        df['rolling_std'] = df['value'].rolling(window=20).std()

        fig_stats = go.Figure()
        fig_stats.add_trace(go.Scatter(
            x=df['timestamp'],
            y=df['rolling_mean'],
            name='Rolling Mean',
            line=dict(color='green')
        ))
        fig_stats.add_trace(go.Scatter(
            x=df['timestamp'],
            y=df['rolling_std'],
            name='Rolling Std',
            line=dict(color='purple')
        ))
        fig_stats.update_layout(title='Rolling Statistics')
        st.plotly_chart(fig_stats, use_container_width=True)
```

## Expected Outputs

1. **Live Dashboard**:
   - Auto-updating time series chart
   - Real-time metrics cards
   - Anomaly highlighting

2. **Alert System**:
   - Visual alerts for anomalies
   - Alert history log
   - Configurable thresholds

3. **Statistical Views**:
   - Distribution histogram
   - Rolling statistics
   - Trend indicators

4. **Controls**:
   - Start/stop streaming
   - Adjust update frequency
   - Configure alert thresholds

## Bonus Challenges

- [ ] Add multiple data streams (multi-sensor monitoring)
- [ ] Implement predictive alerting (forecast anomalies)
- [ ] Add database storage for historical data
- [ ] Create downloadable reports
- [ ] Implement WebSocket for true real-time updates
- [ ] Add sound/email notifications
- [ ] Create custom alert rules engine
- [ ] Add data replay functionality
- [ ] Implement seasonal decomposition
- [ ] Add correlation analysis between streams

## Resources

- [Streamlit Auto-refresh](https://docs.streamlit.io/library/api-reference/control-flow/st.rerun)
- [Plotly Streaming](https://plotly.com/python/streaming/)
- [Isolation Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html)
- [Redis for Streaming](https://redis.io/topics/streams-intro)

## Success Criteria

- Dashboard updates in real-time
- Anomalies are detected and highlighted
- Alerts trigger correctly
- No performance degradation over time
- Visual indicators are clear and intuitive
- Controls are responsive
- Data buffering works efficiently
