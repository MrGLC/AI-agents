# Project 5: ML Model Performance Comparison Tool

## Overview
Build a comprehensive visualization tool to compare multiple machine learning models side-by-side, showing performance metrics, learning curves, feature importance, and decision boundaries.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Train and evaluate multiple ML models
- Visualize model comparison metrics
- Create learning curves and validation curves
- Display ROC/PR curves for classification
- Visualize feature importance across models
- Build interactive model selection interface

## Technical Stack
- **Backend**: Python
- **ML Framework**: scikit-learn, XGBoost, LightGBM
- **Visualization**: Plotly, Matplotlib, Seaborn
- **Dashboard**: Streamlit
- **Data Processing**: Pandas, NumPy
- **Dataset**: Classification (Iris, Titanic, etc.) or Regression (Housing, etc.)

## Project Requirements

### 1. Model Training
- Support multiple model types (LogisticRegression, RandomForest, XGBoost, SVM, etc.)
- Cross-validation for robust evaluation
- Hyperparameter tuning (optional)
- Train on same dataset for fair comparison

### 2. Performance Metrics
- **Classification**: Accuracy, Precision, Recall, F1-Score, ROC-AUC
- **Regression**: MAE, RMSE, R², MAPE
- Confusion matrices for classification
- Residual plots for regression

### 3. Core Visualizations
- **Metrics Comparison**: Bar chart of all metrics by model
- **ROC Curves**: All models on same plot (classification)
- **Precision-Recall Curves**: Model comparison (classification)
- **Confusion Matrices**: Heatmap for each model
- **Learning Curves**: Training vs validation score over training size
- **Feature Importance**: Bar chart comparison across models
- **Prediction Comparison**: Actual vs Predicted scatter plots

### 4. Interactive Features
- Model selector (choose which models to compare)
- Metric selector (choose which metrics to display)
- Test size slider
- Random state selector for reproducibility
- Download comparison report

### 5. Analysis Tools
- Best model highlighter
- Statistical significance testing
- Training time comparison
- Model complexity comparison

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy scikit-learn xgboost lightgbm plotly seaborn matplotlib
```

### Step 2: Load Data and Prepare Models
```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC
from xgboost import XGBClassifier
from lightgbm import LGBMClassifier

# Load dataset
from sklearn.datasets import load_breast_cancer
data = load_breast_cancer()
X = pd.DataFrame(data.data, columns=data.feature_names)
y = pd.Series(data.target)

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Scale features
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

# Define models
models = {
    'Logistic Regression': LogisticRegression(max_iter=1000),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42),
    'Gradient Boosting': GradientBoostingClassifier(random_state=42),
    'XGBoost': XGBClassifier(random_state=42),
    'LightGBM': LGBMClassifier(random_state=42),
    'SVM': SVC(probability=True, random_state=42)
}
```

### Step 3: Train and Evaluate Models
```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, roc_curve, confusion_matrix
)
import time

results = {}

for name, model in models.items():
    # Train model and measure time
    start_time = time.time()
    model.fit(X_train_scaled, y_train)
    training_time = time.time() - start_time

    # Predictions
    y_pred = model.predict(X_test_scaled)
    y_pred_proba = model.predict_proba(X_test_scaled)[:, 1] if hasattr(model, 'predict_proba') else None

    # Calculate metrics
    results[name] = {
        'model': model,
        'accuracy': accuracy_score(y_test, y_pred),
        'precision': precision_score(y_test, y_pred),
        'recall': recall_score(y_test, y_pred),
        'f1': f1_score(y_test, y_pred),
        'roc_auc': roc_auc_score(y_test, y_pred_proba) if y_pred_proba is not None else None,
        'training_time': training_time,
        'predictions': y_pred,
        'probabilities': y_pred_proba,
        'confusion_matrix': confusion_matrix(y_test, y_pred)
    }
```

### Step 4: Create Comparison Dashboard
```python
import streamlit as st
import plotly.graph_objects as go
import plotly.express as px

st.title('🔬 ML Model Performance Comparator')

st.sidebar.header('Configuration')
selected_models = st.sidebar.multiselect(
    'Select Models',
    list(models.keys()),
    default=list(models.keys())[:3]
)

# Metrics comparison
st.subheader('Performance Metrics Comparison')

metrics_df = pd.DataFrame({
    model: {
        'Accuracy': results[model]['accuracy'],
        'Precision': results[model]['precision'],
        'Recall': results[model]['recall'],
        'F1-Score': results[model]['f1'],
        'ROC-AUC': results[model]['roc_auc'],
    }
    for model in selected_models
}).T

st.dataframe(metrics_df.style.highlight_max(axis=0, color='lightgreen'))

# Bar chart comparison
fig = go.Figure()
for metric in metrics_df.columns:
    fig.add_trace(go.Bar(
        name=metric,
        x=metrics_df.index,
        y=metrics_df[metric]
    ))

fig.update_layout(
    title='Model Metrics Comparison',
    xaxis_title='Model',
    yaxis_title='Score',
    barmode='group'
)

st.plotly_chart(fig)
```

### Step 5: ROC Curve Comparison
```python
st.subheader('ROC Curve Comparison')

fig_roc = go.Figure()

for model_name in selected_models:
    if results[model_name]['probabilities'] is not None:
        fpr, tpr, _ = roc_curve(y_test, results[model_name]['probabilities'])
        auc = results[model_name]['roc_auc']

        fig_roc.add_trace(go.Scatter(
            x=fpr, y=tpr,
            name=f'{model_name} (AUC = {auc:.3f})',
            mode='lines'
        ))

# Add diagonal line
fig_roc.add_trace(go.Scatter(
    x=[0, 1], y=[0, 1],
    name='Random Classifier',
    mode='lines',
    line=dict(dash='dash', color='gray')
))

fig_roc.update_layout(
    title='ROC Curves',
    xaxis_title='False Positive Rate',
    yaxis_title='True Positive Rate',
    width=700,
    height=500
)

st.plotly_chart(fig_roc)
```

### Step 6: Confusion Matrices
```python
st.subheader('Confusion Matrices')

cols = st.columns(len(selected_models))

for idx, model_name in enumerate(selected_models):
    with cols[idx]:
        cm = results[model_name]['confusion_matrix']

        fig = px.imshow(
            cm,
            labels=dict(x='Predicted', y='Actual'),
            x=['Negative', 'Positive'],
            y=['Negative', 'Positive'],
            text_auto=True,
            title=model_name
        )

        st.plotly_chart(fig, use_container_width=True)
```

### Step 7: Feature Importance Comparison
```python
st.subheader('Feature Importance Comparison')

importance_data = {}

for model_name in selected_models:
    model = results[model_name]['model']

    if hasattr(model, 'feature_importances_'):
        importance_data[model_name] = model.feature_importances_
    elif hasattr(model, 'coef_'):
        importance_data[model_name] = np.abs(model.coef_[0])

if importance_data:
    importance_df = pd.DataFrame(importance_data, index=X.columns)

    # Plot top 10 features
    top_features = importance_df.mean(axis=1).nlargest(10).index

    fig = go.Figure()
    for model_name in importance_data.keys():
        fig.add_trace(go.Bar(
            name=model_name,
            x=top_features,
            y=importance_df.loc[top_features, model_name]
        ))

    fig.update_layout(
        title='Top 10 Feature Importance by Model',
        xaxis_title='Features',
        yaxis_title='Importance',
        barmode='group'
    )

    st.plotly_chart(fig)
```

### Step 8: Learning Curves
```python
from sklearn.model_selection import learning_curve

st.subheader('Learning Curves')

selected_model_for_curve = st.selectbox('Select model for learning curve', selected_models)

model = models[selected_model_for_curve]

train_sizes, train_scores, val_scores = learning_curve(
    model, X_train_scaled, y_train,
    cv=5, n_jobs=-1,
    train_sizes=np.linspace(0.1, 1.0, 10),
    scoring='accuracy'
)

train_mean = np.mean(train_scores, axis=1)
train_std = np.std(train_scores, axis=1)
val_mean = np.mean(val_scores, axis=1)
val_std = np.std(val_scores, axis=1)

fig = go.Figure()

fig.add_trace(go.Scatter(
    x=train_sizes, y=train_mean,
    name='Training Score',
    mode='lines+markers',
    line=dict(color='blue')
))

fig.add_trace(go.Scatter(
    x=train_sizes, y=val_mean,
    name='Validation Score',
    mode='lines+markers',
    line=dict(color='orange')
))

fig.update_layout(
    title=f'Learning Curve - {selected_model_for_curve}',
    xaxis_title='Training Examples',
    yaxis_title='Accuracy Score'
)

st.plotly_chart(fig)
```

## Expected Outputs

1. **Metrics Dashboard**:
   - Table with all metrics highlighted
   - Bar chart comparing all models
   - Best model indicator

2. **ROC/PR Curves**:
   - All models on same plot
   - AUC scores in legend
   - Interactive zoom and pan

3. **Confusion Matrices**:
   - Side-by-side heatmaps
   - True/false positive/negative counts

4. **Feature Importance**:
   - Comparison across models
   - Top features highlighted

5. **Learning Curves**:
   - Training vs validation performance
   - Overfitting/underfitting detection

## Bonus Challenges

- [ ] Add cross-validation score distribution plots
- [ ] Implement hyperparameter tuning visualization
- [ ] Add decision boundary visualization (2D)
- [ ] Include training time vs performance scatter plot
- [ ] Add statistical significance tests (paired t-test)
- [ ] Create calibration curves for probability predictions
- [ ] Add ensemble voting visualization
- [ ] Implement cost-benefit analysis
- [ ] Add model explanation with SHAP values
- [ ] Create automated model selection recommendation

## Resources

- [scikit-learn Model Selection](https://scikit-learn.org/stable/model_selection.html)
- [Plotly Python](https://plotly.com/python/)
- [ROC Curve Explained](https://developers.google.com/machine-learning/crash-course/classification/roc-and-auc)
- [Learning Curves](https://scikit-learn.org/stable/modules/learning_curve.html)

## Success Criteria

- Multiple models trained and evaluated successfully
- All metrics calculated correctly
- ROC curves display properly for all models
- Confusion matrices are accurate
- Feature importance shows meaningful patterns
- Learning curves reveal training behavior
- Dashboard is interactive and responsive
- Best model is clearly identified
