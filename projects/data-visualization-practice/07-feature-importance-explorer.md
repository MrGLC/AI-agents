# Project 7: Feature Importance Explorer

## Overview
Create an interactive tool to explore and visualize feature importance across different ML models, including SHAP values, permutation importance, and correlation analysis.

## Difficulty Level
Advanced

## Learning Objectives
- Calculate feature importance using multiple methods
- Visualize SHAP values and dependencies
- Analyze feature correlations and interactions
- Create partial dependence plots
- Compare feature importance across models
- Build interactive feature selection tools

## Technical Stack
- **Backend**: Python
- **ML Framework**: scikit-learn, XGBoost, LightGBM
- **Interpretation**: SHAP, eli5
- **Visualization**: Plotly, Matplotlib
- **Dashboard**: Streamlit
- **Data Processing**: Pandas, NumPy
- **Dataset**: Tabular dataset with multiple features (Titanic, Housing, etc.)

## Project Requirements

### 1. Feature Importance Calculation
- Tree-based importance (Random Forest, XGBoost)
- Permutation importance
- SHAP values (TreeExplainer, KernelExplainer)
- Coefficients (Linear models)
- Correlation-based importance

### 2. Core Visualizations
- **Feature Importance Bar Chart**: Ranked features
- **SHAP Summary Plot**: Beeswarm plot showing feature impact
- **SHAP Waterfall Plot**: Individual prediction explanation
- **SHAP Dependence Plot**: Feature vs SHAP value scatter
- **Partial Dependence Plot**: How features affect predictions
- **Feature Correlation Heatmap**: Feature relationships
- **Feature Interaction Plot**: Two-way interactions

### 3. Interactive Features
- Model selector
- Feature selector
- Sample selector for individual explanations
- Importance method selector
- Top-K features slider
- Export plots and data

### 4. Analysis Tools
- Feature ranking comparison
- Redundant feature detection
- Feature selection recommendations
- Model simplification suggestions

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy scikit-learn xgboost shap plotly matplotlib seaborn
```

### Step 2: Load Data and Train Models
```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from xgboost import XGBClassifier
import shap

# Load dataset
df = pd.read_csv('data.csv')
X = df.drop('target', axis=1)
y = df['target']

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Train models
rf_model = RandomForestClassifier(n_estimators=100, random_state=42)
rf_model.fit(X_train, y_train)

xgb_model = XGBClassifier(random_state=42)
xgb_model.fit(X_train, y_train)
```

### Step 3: Calculate Multiple Feature Importance Methods
```python
from sklearn.inspection import permutation_importance

def get_feature_importances(model, X_train, y_train, X_test, y_test):
    """Calculate feature importance using multiple methods"""

    importances = {}

    # 1. Built-in feature importance (tree-based models)
    if hasattr(model, 'feature_importances_'):
        importances['builtin'] = pd.Series(
            model.feature_importances_,
            index=X_train.columns
        ).sort_values(ascending=False)

    # 2. Permutation importance
    perm_importance = permutation_importance(
        model, X_test, y_test,
        n_repeats=10,
        random_state=42
    )
    importances['permutation'] = pd.Series(
        perm_importance.importances_mean,
        index=X_train.columns
    ).sort_values(ascending=False)

    # 3. SHAP values
    explainer = shap.TreeExplainer(model)
    shap_values = explainer.shap_values(X_test)

    if isinstance(shap_values, list):  # Multi-class
        shap_values = shap_values[1]  # Positive class

    importances['shap'] = pd.Series(
        np.abs(shap_values).mean(axis=0),
        index=X_train.columns
    ).sort_values(ascending=False)

    return importances, explainer, shap_values
```

### Step 4: Create Interactive Dashboard
```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

st.set_page_config(page_title='Feature Importance Explorer', layout='wide')

st.title('🔍 Feature Importance Explorer')

# Sidebar
st.sidebar.header('Configuration')
model_choice = st.sidebar.selectbox('Select Model', ['Random Forest', 'XGBoost'])
importance_method = st.sidebar.selectbox(
    'Importance Method',
    ['Built-in', 'Permutation', 'SHAP']
)
top_k = st.sidebar.slider('Top K Features', 5, 20, 10)

# Select model
model = rf_model if model_choice == 'Random Forest' else xgb_model

# Calculate importances
importances, explainer, shap_values = get_feature_importances(
    model, X_train, y_train, X_test, y_test
)

# Display feature importance
st.subheader(f'Feature Importance ({importance_method})')

method_key = importance_method.lower().replace('-', '')
if method_key in importances:
    top_features = importances[method_key].head(top_k)

    fig = px.bar(
        x=top_features.values,
        y=top_features.index,
        orientation='h',
        labels={'x': 'Importance', 'y': 'Feature'},
        title=f'Top {top_k} Most Important Features'
    )
    fig.update_layout(yaxis={'categoryorder': 'total ascending'})
    st.plotly_chart(fig, use_container_width=True)

    # Show values
    st.dataframe(top_features.to_frame('Importance'))
```

### Step 5: SHAP Visualizations
```python
st.subheader('SHAP Analysis')

tab1, tab2, tab3, tab4 = st.tabs([
    'Summary Plot',
    'Waterfall Plot',
    'Dependence Plot',
    'Force Plot'
])

with tab1:
    st.write('**SHAP Summary Plot** - Shows the impact of each feature')

    # Create summary plot
    fig, ax = plt.subplots(figsize=(10, 8))
    shap.summary_plot(shap_values, X_test, show=False)
    st.pyplot(fig)

with tab2:
    st.write('**SHAP Waterfall Plot** - Individual prediction explanation')

    sample_idx = st.slider('Select Sample', 0, len(X_test)-1, 0)

    fig, ax = plt.subplots(figsize=(10, 6))
    shap.waterfall_plot(
        shap.Explanation(
            values=shap_values[sample_idx],
            base_values=explainer.expected_value if isinstance(explainer.expected_value, float) else explainer.expected_value[1],
            data=X_test.iloc[sample_idx],
            feature_names=X_test.columns.tolist()
        ),
        show=False
    )
    st.pyplot(fig)

    # Show sample details
    st.write('**Sample Details:**')
    st.dataframe(X_test.iloc[sample_idx].to_frame('Value'))

with tab3:
    st.write('**SHAP Dependence Plot** - Feature vs SHAP value')

    selected_feature = st.selectbox(
        'Select Feature',
        importances['shap'].head(top_k).index.tolist()
    )

    fig, ax = plt.subplots(figsize=(10, 6))
    shap.dependence_plot(
        selected_feature,
        shap_values,
        X_test,
        show=False
    )
    st.pyplot(fig)

with tab4:
    st.write('**SHAP Force Plot** - Detailed breakdown')

    sample_idx_force = st.slider('Sample Index', 0, min(100, len(X_test)-1), 0)

    # Note: Force plots work best in Jupyter, showing text alternative
    st.write(f'Prediction for sample {sample_idx_force}:')
    st.write(f'Base value: {explainer.expected_value if isinstance(explainer.expected_value, float) else explainer.expected_value[1]:.3f}')
    st.write(f'Predicted value: {model.predict_proba(X_test.iloc[[sample_idx_force]])[0][1]:.3f}')

    # Top positive and negative contributions
    feature_contrib = pd.Series(
        shap_values[sample_idx_force],
        index=X_test.columns
    ).sort_values()

    col1, col2 = st.columns(2)

    with col1:
        st.write('**Top Positive Contributions:**')
        st.dataframe(feature_contrib.tail(5))

    with col2:
        st.write('**Top Negative Contributions:**')
        st.dataframe(feature_contrib.head(5))
```

### Step 6: Partial Dependence Plots
```python
from sklearn.inspection import PartialDependenceDisplay

st.subheader('Partial Dependence Plots')

features_for_pdp = st.multiselect(
    'Select features for PDP (max 4)',
    importances['shap'].head(10).index.tolist(),
    default=importances['shap'].head(2).index.tolist()[:2]
)

if features_for_pdp:
    feature_indices = [X_train.columns.get_loc(f) for f in features_for_pdp]

    fig, ax = plt.subplots(figsize=(12, 4 * ((len(features_for_pdp) + 1) // 2)))

    display = PartialDependenceDisplay.from_estimator(
        model,
        X_train,
        feature_indices,
        ax=ax,
        n_jobs=-1
    )

    st.pyplot(fig)
```

### Step 7: Feature Correlation Analysis
```python
st.subheader('Feature Correlation Analysis')

# Calculate correlation matrix
correlation_matrix = X_train[importances['shap'].head(top_k).index].corr()

fig = px.imshow(
    correlation_matrix,
    labels=dict(color='Correlation'),
    x=correlation_matrix.columns,
    y=correlation_matrix.columns,
    color_continuous_scale='RdBu_r',
    zmin=-1,
    zmax=1,
    title='Feature Correlation Heatmap'
)

st.plotly_chart(fig, use_container_width=True)

# Identify highly correlated features
st.subheader('Highly Correlated Feature Pairs')

threshold = st.slider('Correlation Threshold', 0.5, 0.95, 0.8)

high_corr_pairs = []
for i in range(len(correlation_matrix.columns)):
    for j in range(i+1, len(correlation_matrix.columns)):
        if abs(correlation_matrix.iloc[i, j]) > threshold:
            high_corr_pairs.append({
                'Feature 1': correlation_matrix.columns[i],
                'Feature 2': correlation_matrix.columns[j],
                'Correlation': correlation_matrix.iloc[i, j]
            })

if high_corr_pairs:
    st.dataframe(pd.DataFrame(high_corr_pairs))
    st.info('💡 Consider removing one feature from each highly correlated pair')
else:
    st.success('No highly correlated features found')
```

### Step 8: Feature Selection Recommendations
```python
st.subheader('📊 Feature Selection Recommendations')

col1, col2 = st.columns(2)

with col1:
    st.write('**Most Important Features:**')
    recommended_features = importances['shap'].head(10)
    st.dataframe(recommended_features.to_frame('Importance Score'))

with col2:
    st.write('**Least Important Features:**')
    low_importance = importances['shap'].tail(10)
    st.dataframe(low_importance.to_frame('Importance Score'))

# Export options
st.subheader('💾 Export')

if st.button('Download Feature Importance Report'):
    report_df = pd.DataFrame({
        'Feature': importances['shap'].index,
        'SHAP Importance': importances['shap'].values,
        'Builtin Importance': [importances['builtin'].get(f, 0) for f in importances['shap'].index],
        'Permutation Importance': [importances['permutation'].get(f, 0) for f in importances['shap'].index]
    })

    csv = report_df.to_csv(index=False)
    st.download_button(
        label='Download CSV',
        data=csv,
        file_name='feature_importance_report.csv',
        mime='text/csv'
    )
```

## Expected Outputs

1. **Feature Importance Rankings**:
   - Bar charts for different methods
   - Comparison table
   - Top features highlighted

2. **SHAP Visualizations**:
   - Summary plot (beeswarm)
   - Waterfall plots for individuals
   - Dependence plots
   - Force plots

3. **Partial Dependence Plots**:
   - Individual feature effects
   - Interaction effects

4. **Correlation Analysis**:
   - Heatmap
   - Highly correlated pairs
   - Redundancy detection

5. **Recommendations**:
   - Feature selection suggestions
   - Downloadable reports

## Bonus Challenges

- [ ] Add feature interaction detection
- [ ] Implement automatic feature engineering suggestions
- [ ] Add LIME explanations for comparison
- [ ] Create feature importance stability analysis
- [ ] Add adversarial robustness testing
- [ ] Implement ICE (Individual Conditional Expectation) plots
- [ ] Add anchors for rule-based explanations
- [ ] Create feature importance evolution over training
- [ ] Add counterfactual explanations
- [ ] Implement global surrogate models

## Resources

- [SHAP Documentation](https://shap.readthedocs.io/)
- [Permutation Importance](https://scikit-learn.org/stable/modules/permutation_importance.html)
- [Partial Dependence](https://scikit-learn.org/stable/modules/partial_dependence.html)
- [Interpretable ML Book](https://christophm.github.io/interpretable-ml-book/)

## Success Criteria

- Multiple importance methods calculated correctly
- SHAP visualizations render properly
- Interactive controls work smoothly
- Partial dependence plots are informative
- Correlation analysis identifies redundant features
- Recommendations are actionable
- Export functionality works
- Dashboard is responsive and intuitive
