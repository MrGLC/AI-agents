# Project 10: Anomaly Detection Visualizer

## Overview
Build a comprehensive visualization tool for anomaly detection that displays detected anomalies, confidence scores, temporal patterns, feature contributions, and comparison of multiple anomaly detection algorithms.

## Difficulty Level
Advanced

## Learning Objectives
- Implement multiple anomaly detection algorithms
- Visualize anomalies in time series and multivariate data
- Compare algorithm performance
- Analyze feature contributions to anomalies
- Create interactive anomaly exploration tools
- Build alerting mechanisms
- Generate anomaly reports

## Technical Stack
- **Backend**: Python
- **ML Framework**: scikit-learn, PyOD
- **Visualization**: Plotly, Matplotlib, Seaborn
- **Dashboard**: Streamlit
- **Data Processing**: Pandas, NumPy
- **Dataset**: Time series (sensor data, logs) or multivariate (credit card fraud, network intrusion)

## Project Requirements

### 1. Anomaly Detection Algorithms
- Isolation Forest
- Local Outlier Factor (LOF)
- One-Class SVM
- Autoencoder (deep learning)
- Statistical methods (Z-score, IQR)
- DBSCAN clustering
- Seasonal decomposition (for time series)

### 2. Core Visualizations
- **Anomaly Scatter Plot**: Normal vs anomalous points
- **Time Series with Anomalies**: Highlighted anomalous periods
- **Anomaly Score Distribution**: Histogram of scores
- **Feature Contribution**: Which features caused anomaly
- **Temporal Heatmap**: Anomaly density over time
- **3D Visualization**: For multivariate data
- **Comparison Matrix**: Algorithm agreement/disagreement

### 3. Interactive Features
- Algorithm selector
- Threshold adjuster
- Feature selector
- Date range filter
- Anomaly detail viewer
- Export anomaly reports
- Real-time detection mode

### 4. Analysis Tools
- Algorithm comparison metrics
- False positive analysis
- Anomaly clustering
- Pattern recognition
- Root cause analysis

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy scikit-learn pyod plotly seaborn matplotlib
```

### Step 2: Generate Synthetic Data with Anomalies
```python
import numpy as np
import pandas as pd
from datetime import datetime, timedelta

def generate_data_with_anomalies(n_samples=1000, n_anomalies=50, n_features=3):
    """Generate synthetic multivariate data with anomalies"""

    np.random.seed(42)

    # Normal data
    normal_data = np.random.randn(n_samples - n_anomalies, n_features)

    # Anomalies (outliers)
    anomalies = np.random.uniform(low=-5, high=5, size=(n_anomalies, n_features))

    # Combine
    X = np.vstack([normal_data, anomalies])

    # Labels (0 = normal, 1 = anomaly)
    y = np.hstack([
        np.zeros(n_samples - n_anomalies),
        np.ones(n_anomalies)
    ])

    # Shuffle
    indices = np.random.permutation(n_samples)
    X = X[indices]
    y = y[indices]

    # Create timestamps
    timestamps = [datetime.now() - timedelta(hours=n_samples-i) for i in range(n_samples)]

    # Create DataFrame
    df = pd.DataFrame(X, columns=[f'feature_{i}' for i in range(n_features)])
    df['timestamp'] = timestamps
    df['is_anomaly'] = y

    return df

# Generate data
df = generate_data_with_anomalies(n_samples=1000, n_anomalies=50, n_features=3)
```

### Step 3: Implement Multiple Anomaly Detection Algorithms
```python
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor
from sklearn.svm import OneClassSVM
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import DBSCAN
from scipy import stats

class AnomalyDetectorEnsemble:
    """Ensemble of anomaly detection algorithms"""

    def __init__(self):
        self.models = {
            'Isolation Forest': IsolationForest(contamination=0.1, random_state=42),
            'LOF': LocalOutlierFactor(novelty=True, contamination=0.1),
            'One-Class SVM': OneClassSVM(nu=0.1),
        }
        self.scaler = StandardScaler()
        self.scores = {}

    def fit_predict(self, X):
        """Fit all models and predict anomalies"""

        # Scale data
        X_scaled = self.scaler.fit_transform(X)

        results = {}

        for name, model in self.models.items():
            # Fit and predict
            if name == 'LOF':
                model.fit(X_scaled)
                predictions = model.predict(X_scaled)
                scores = -model.score_samples(X_scaled)
            else:
                predictions = model.fit_predict(X_scaled)
                scores = -model.score_samples(X_scaled)

            # Convert to binary (1 = anomaly, 0 = normal)
            predictions = (predictions == -1).astype(int)

            results[name] = {
                'predictions': predictions,
                'scores': scores
            }

        # Statistical methods
        z_scores = np.abs(stats.zscore(X_scaled, axis=0)).max(axis=1)
        z_predictions = (z_scores > 3).astype(int)

        results['Z-Score'] = {
            'predictions': z_predictions,
            'scores': z_scores
        }

        return results

    def get_ensemble_prediction(self, results, threshold=0.5):
        """Get ensemble prediction (voting)"""

        predictions = np.array([r['predictions'] for r in results.values()])
        ensemble = (predictions.mean(axis=0) >= threshold).astype(int)

        return ensemble
```

### Step 4: Create Interactive Dashboard
```python
import streamlit as st
import plotly.graph_objects as go
import plotly.express as px
from plotly.subplots import make_subplots

st.set_page_config(page_title='Anomaly Detection Visualizer', layout='wide')

st.title('🔍 Anomaly Detection Visualizer')

# Load data
df = generate_data_with_anomalies(n_samples=1000, n_anomalies=50, n_features=3)

# Sidebar
st.sidebar.header('Configuration')

algorithm = st.sidebar.selectbox(
    'Detection Algorithm',
    ['Isolation Forest', 'LOF', 'One-Class SVM', 'Z-Score', 'Ensemble (Voting)']
)

contamination = st.sidebar.slider(
    'Expected Contamination',
    0.01, 0.3, 0.1, 0.01,
    help='Expected proportion of anomalies in the data'
)

threshold = st.sidebar.slider(
    'Anomaly Score Threshold',
    0.0, 1.0, 0.5, 0.05,
    help='Threshold for ensemble voting'
)

# Prepare features
feature_columns = [col for col in df.columns if col.startswith('feature_')]
X = df[feature_columns].values

# Run anomaly detection
detector = AnomalyDetectorEnsemble()
results = detector.fit_predict(X)

# Get predictions
if algorithm == 'Ensemble (Voting)':
    predictions = detector.get_ensemble_prediction(results, threshold)
    scores = np.array([r['scores'] for r in results.values()]).mean(axis=0)
else:
    predictions = results[algorithm]['predictions']
    scores = results[algorithm]['scores']

df['predicted_anomaly'] = predictions
df['anomaly_score'] = scores

# Summary metrics
st.subheader('📊 Detection Summary')

col1, col2, col3, col4 = st.columns(4)

n_detected = predictions.sum()
n_true_anomalies = df['is_anomaly'].sum()
true_positives = ((predictions == 1) & (df['is_anomaly'] == 1)).sum()
false_positives = ((predictions == 1) & (df['is_anomaly'] == 0)).sum()

precision = true_positives / n_detected if n_detected > 0 else 0
recall = true_positives / n_true_anomalies if n_true_anomalies > 0 else 0

col1.metric('Detected Anomalies', f'{n_detected}')
col2.metric('True Anomalies', f'{int(n_true_anomalies)}')
col3.metric('Precision', f'{precision:.2%}')
col4.metric('Recall', f'{recall:.2%}')
```

### Step 5: Time Series Visualization with Anomalies
```python
st.subheader('Time Series View')

# Select feature to visualize
selected_feature = st.selectbox('Select Feature', feature_columns)

fig = go.Figure()

# Normal points
normal_df = df[df['predicted_anomaly'] == 0]
fig.add_trace(go.Scatter(
    x=normal_df['timestamp'],
    y=normal_df[selected_feature],
    mode='markers',
    name='Normal',
    marker=dict(color='blue', size=5)
))

# Anomalous points
anomaly_df = df[df['predicted_anomaly'] == 1]
fig.add_trace(go.Scatter(
    x=anomaly_df['timestamp'],
    y=anomaly_df[selected_feature],
    mode='markers',
    name='Anomaly',
    marker=dict(color='red', size=10, symbol='x')
))

fig.update_layout(
    title=f'{selected_feature} Over Time',
    xaxis_title='Time',
    yaxis_title='Value',
    height=400
)

st.plotly_chart(fig, use_container_width=True)
```

### Step 6: Multivariate Anomaly Visualization
```python
st.subheader('Multivariate Analysis')

if len(feature_columns) >= 2:
    col1, col2 = st.columns(2)

    with col1:
        x_feature = st.selectbox('X-axis', feature_columns, index=0)

    with col2:
        y_feature = st.selectbox('Y-axis', feature_columns, index=1)

    # 2D scatter plot
    fig = px.scatter(
        df,
        x=x_feature,
        y=y_feature,
        color=df['predicted_anomaly'].map({0: 'Normal', 1: 'Anomaly'}),
        color_discrete_map={'Normal': 'blue', 'Anomaly': 'red'},
        hover_data=['timestamp', 'anomaly_score'],
        title=f'{x_feature} vs {y_feature}'
    )

    st.plotly_chart(fig, use_container_width=True)

    # 3D scatter plot if 3+ features
    if len(feature_columns) >= 3:
        st.write('**3D Visualization:**')

        z_feature = st.selectbox('Z-axis', feature_columns, index=2)

        fig_3d = px.scatter_3d(
            df,
            x=x_feature,
            y=y_feature,
            z=z_feature,
            color=df['predicted_anomaly'].map({0: 'Normal', 1: 'Anomaly'}),
            color_discrete_map={'Normal': 'blue', 'Anomaly': 'red'},
            hover_data=['timestamp', 'anomaly_score']
        )

        fig_3d.update_layout(height=600)
        st.plotly_chart(fig_3d, use_container_width=True)
```

### Step 7: Anomaly Score Distribution
```python
st.subheader('Anomaly Score Distribution')

fig = go.Figure()

# Histogram for normal points
fig.add_trace(go.Histogram(
    x=df[df['predicted_anomaly'] == 0]['anomaly_score'],
    name='Normal',
    opacity=0.7,
    marker_color='blue',
    nbinsx=30
))

# Histogram for anomalies
fig.add_trace(go.Histogram(
    x=df[df['predicted_anomaly'] == 1]['anomaly_score'],
    name='Anomaly',
    opacity=0.7,
    marker_color='red',
    nbinsx=30
))

fig.update_layout(
    title='Distribution of Anomaly Scores',
    xaxis_title='Anomaly Score',
    yaxis_title='Count',
    barmode='overlay'
)

st.plotly_chart(fig, use_container_width=True)
```

### Step 8: Algorithm Comparison
```python
st.subheader('Algorithm Comparison')

# Calculate metrics for each algorithm
comparison_data = []

for algo_name, result in results.items():
    preds = result['predictions']
    n_detected = preds.sum()
    tp = ((preds == 1) & (df['is_anomaly'] == 1)).sum()
    fp = ((preds == 1) & (df['is_anomaly'] == 0)).sum()

    precision = tp / n_detected if n_detected > 0 else 0
    recall = tp / n_true_anomalies if n_true_anomalies > 0 else 0
    f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0

    comparison_data.append({
        'Algorithm': algo_name,
        'Detected': int(n_detected),
        'Precision': precision,
        'Recall': recall,
        'F1-Score': f1
    })

comparison_df = pd.DataFrame(comparison_data)

# Display table
st.dataframe(comparison_df.style.highlight_max(axis=0, subset=['Precision', 'Recall', 'F1-Score']))

# Bar chart
fig = go.Figure()

fig.add_trace(go.Bar(
    x=comparison_df['Algorithm'],
    y=comparison_df['Precision'],
    name='Precision'
))

fig.add_trace(go.Bar(
    x=comparison_df['Algorithm'],
    y=comparison_df['Recall'],
    name='Recall'
))

fig.add_trace(go.Bar(
    x=comparison_df['Algorithm'],
    y=comparison_df['F1-Score'],
    name='F1-Score'
))

fig.update_layout(
    title='Algorithm Performance Comparison',
    xaxis_title='Algorithm',
    yaxis_title='Score',
    barmode='group'
)

st.plotly_chart(fig, use_container_width=True)
```

### Step 9: Anomaly Detail Viewer
```python
st.subheader('Anomaly Details')

# Show detected anomalies
anomalies_detected = df[df['predicted_anomaly'] == 1].sort_values('anomaly_score', ascending=False)

if len(anomalies_detected) > 0:
    st.write(f'**Top {min(10, len(anomalies_detected))} Anomalies by Score:**')

    st.dataframe(
        anomalies_detected.head(10)[['timestamp'] + feature_columns + ['anomaly_score', 'is_anomaly']],
        use_container_width=True
    )

    # Feature contribution analysis
    st.write('**Feature Contribution to Anomalies:**')

    # Calculate how much each feature deviates from normal
    normal_mean = df[df['predicted_anomaly'] == 0][feature_columns].mean()
    normal_std = df[df['predicted_anomaly'] == 0][feature_columns].std()

    anomaly_deviations = []
    for _, row in anomalies_detected.head(10).iterrows():
        deviations = {}
        for feature in feature_columns:
            z_score = abs((row[feature] - normal_mean[feature]) / normal_std[feature])
            deviations[feature] = z_score

        anomaly_deviations.append(deviations)

    deviation_df = pd.DataFrame(anomaly_deviations)

    fig = px.imshow(
        deviation_df.T,
        labels=dict(x='Anomaly Index', y='Feature', color='Z-Score'),
        x=[f'Anomaly {i+1}' for i in range(len(deviation_df))],
        y=feature_columns,
        color_continuous_scale='Reds',
        title='Feature Deviation Heatmap (Z-Scores)'
    )

    st.plotly_chart(fig, use_container_width=True)

else:
    st.info('No anomalies detected with current settings.')
```

### Step 10: Export and Reporting
```python
st.subheader('💾 Export Results')

col1, col2 = st.columns(2)

with col1:
    if st.button('Download Anomaly Report'):
        report_df = df[df['predicted_anomaly'] == 1][['timestamp'] + feature_columns + ['anomaly_score']]

        csv = report_df.to_csv(index=False)
        st.download_button(
            label='Download CSV',
            data=csv,
            file_name=f'anomaly_report_{algorithm.replace(" ", "_")}.csv',
            mime='text/csv'
        )

with col2:
    if st.button('Download All Results'):
        all_results = df[['timestamp'] + feature_columns + ['predicted_anomaly', 'anomaly_score', 'is_anomaly']]

        csv = all_results.to_csv(index=False)
        st.download_button(
            label='Download CSV',
            data=csv,
            file_name='anomaly_detection_results.csv',
            mime='text/csv'
        )
```

## Expected Outputs

1. **Detection Summary**:
   - Number of detected anomalies
   - Precision and recall metrics
   - Algorithm performance

2. **Visualizations**:
   - Time series with highlighted anomalies
   - 2D/3D scatter plots
   - Anomaly score distributions
   - Feature contribution heatmaps

3. **Algorithm Comparison**:
   - Performance metrics table
   - Comparison bar chart
   - Agreement/disagreement analysis

4. **Anomaly Details**:
   - Top anomalies ranked by score
   - Feature contributions
   - Temporal patterns

5. **Export Functionality**:
   - CSV download of anomalies
   - Full results export

## Bonus Challenges

- [ ] Add streaming anomaly detection
- [ ] Implement autoencoder-based detection
- [ ] Add seasonal decomposition for time series
- [ ] Create anomaly clustering analysis
- [ ] Implement feedback loop for labeling
- [ ] Add automated alerting system
- [ ] Create anomaly pattern recognition
- [ ] Implement incremental learning
- [ ] Add explainability with SHAP
- [ ] Create anomaly forecasting

## Resources

- [PyOD Documentation](https://pyod.readthedocs.io/)
- [scikit-learn Outlier Detection](https://scikit-learn.org/stable/modules/outlier_detection.html)
- [Isolation Forest Paper](https://cs.nju.edu.cn/zhouzh/zhouzh.files/publication/icdm08b.pdf)
- [Anomaly Detection Survey](https://www.sciencedirect.com/science/article/pii/S1566253517302282)

## Success Criteria

- Multiple algorithms implemented correctly
- Anomalies detected and visualized clearly
- Time series shows anomalies prominently
- Algorithm comparison is meaningful
- Feature contributions are calculated accurately
- Export functionality works
- Dashboard is intuitive and responsive
- Performance metrics are accurate
