# Project 8: A/B Test Results Dashboard

## Overview
Build a comprehensive dashboard for analyzing A/B test results, including statistical significance testing, visualizations of conversion rates, confidence intervals, and business impact calculations.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Understand A/B testing statistics
- Calculate and visualize confidence intervals
- Perform hypothesis testing
- Create conversion funnel visualizations
- Calculate statistical power and sample size
- Analyze segmented results
- Estimate business impact

## Technical Stack
- **Backend**: Python
- **Statistics**: SciPy, statsmodels
- **Visualization**: Plotly, Seaborn
- **Dashboard**: Streamlit
- **Data Processing**: Pandas, NumPy
- **Dataset**: Simulated A/B test data or real experiment data

## Project Requirements

### 1. Data Input
- Load A/B test data (conversions, users, metrics)
- Support multiple variants (not just A/B)
- Handle continuous and binary metrics
- Date range filtering

### 2. Statistical Analysis
- Conversion rate calculation
- Confidence intervals (95%, 99%)
- Statistical significance testing (t-test, chi-square, Mann-Whitney)
- P-value calculation
- Effect size (Cohen's d, relative uplift)
- Statistical power analysis
- Sample size recommendations

### 3. Core Visualizations
- **Conversion Rate Comparison**: Bar chart with error bars
- **Conversion Over Time**: Daily/weekly trends
- **Confidence Interval Plot**: Visual significance testing
- **Distribution Comparison**: Histograms or KDE plots
- **Funnel Analysis**: Multi-step conversion visualization
- **Segmented Analysis**: Results by user segment
- **Sample Size Calculator**: Interactive power analysis

### 4. Interactive Features
- Variant selector
- Metric selector
- Confidence level adjuster
- Segment filters
- Date range selector
- Export results to PDF/CSV

### 5. Business Impact
- Revenue calculations
- ROI estimates
- Winner declaration with recommendations
- Risk assessment

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy scipy statsmodels plotly seaborn
```

### Step 2: Generate or Load A/B Test Data
```python
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

def generate_ab_test_data(n_users=10000, control_rate=0.10, treatment_rate=0.12):
    """Generate synthetic A/B test data"""

    np.random.seed(42)

    # Generate users
    n_control = n_users // 2
    n_treatment = n_users - n_control

    # Control group
    control = pd.DataFrame({
        'user_id': range(n_control),
        'variant': 'A',
        'converted': np.random.binomial(1, control_rate, n_control),
        'revenue': np.random.exponential(50, n_control),
        'date': [datetime.now() - timedelta(days=np.random.randint(0, 30))
                 for _ in range(n_control)],
        'segment': np.random.choice(['mobile', 'desktop', 'tablet'],
                                   n_control, p=[0.5, 0.4, 0.1])
    })

    # Treatment group
    treatment = pd.DataFrame({
        'user_id': range(n_control, n_users),
        'variant': 'B',
        'converted': np.random.binomial(1, treatment_rate, n_treatment),
        'revenue': np.random.exponential(55, n_treatment),
        'date': [datetime.now() - timedelta(days=np.random.randint(0, 30))
                 for _ in range(n_treatment)],
        'segment': np.random.choice(['mobile', 'desktop', 'tablet'],
                                   n_treatment, p=[0.5, 0.4, 0.1])
    })

    df = pd.concat([control, treatment], ignore_index=True)

    # Only count revenue for converted users
    df.loc[df['converted'] == 0, 'revenue'] = 0

    return df

# Generate data
df = generate_ab_test_data()
```

### Step 3: Statistical Functions
```python
from scipy import stats
import numpy as np

def calculate_conversion_rate(data, variant):
    """Calculate conversion rate for a variant"""
    variant_data = data[data['variant'] == variant]
    conversions = variant_data['converted'].sum()
    users = len(variant_data)
    rate = conversions / users if users > 0 else 0
    return rate, conversions, users

def calculate_confidence_interval(conversions, users, confidence=0.95):
    """Calculate Wilson score confidence interval"""
    if users == 0:
        return 0, 0

    rate = conversions / users
    z = stats.norm.ppf((1 + confidence) / 2)

    denominator = 1 + z**2 / users
    center = (rate + z**2 / (2 * users)) / denominator
    margin = z * np.sqrt((rate * (1 - rate) + z**2 / (4 * users)) / users) / denominator

    return center - margin, center + margin

def calculate_statistical_significance(data, variant_a='A', variant_b='B'):
    """Perform chi-square test for significance"""

    # Get data for each variant
    a_data = data[data['variant'] == variant_a]
    b_data = data[data['variant'] == variant_b]

    # Create contingency table
    a_converted = a_data['converted'].sum()
    a_total = len(a_data)
    b_converted = b_data['converted'].sum()
    b_total = len(b_data)

    contingency_table = np.array([
        [a_converted, a_total - a_converted],
        [b_converted, b_total - b_converted]
    ])

    # Perform chi-square test
    chi2, p_value, dof, expected = stats.chi2_contingency(contingency_table)

    # Calculate effect size (relative uplift)
    a_rate = a_converted / a_total if a_total > 0 else 0
    b_rate = b_converted / b_total if b_total > 0 else 0
    relative_uplift = ((b_rate - a_rate) / a_rate * 100) if a_rate > 0 else 0

    return {
        'chi2': chi2,
        'p_value': p_value,
        'significant': p_value < 0.05,
        'relative_uplift': relative_uplift,
        'a_rate': a_rate,
        'b_rate': b_rate
    }

def calculate_sample_size(baseline_rate, mde, power=0.8, alpha=0.05):
    """Calculate required sample size per variant"""

    # Convert MDE (minimum detectable effect) to absolute difference
    delta = baseline_rate * mde

    # Z-scores
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta = stats.norm.ppf(power)

    # Pooled proportion
    p1 = baseline_rate
    p2 = baseline_rate + delta
    p_avg = (p1 + p2) / 2

    # Sample size formula
    n = (z_alpha * np.sqrt(2 * p_avg * (1 - p_avg)) +
         z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2)))**2 / delta**2

    return int(np.ceil(n))
```

### Step 4: Create Dashboard
```python
import streamlit as st
import plotly.graph_objects as go
import plotly.express as px

st.set_page_config(page_title='A/B Test Dashboard', layout='wide')

st.title('🧪 A/B Test Results Dashboard')

# Load data
df = generate_ab_test_data(n_users=10000)

# Sidebar
st.sidebar.header('Configuration')
confidence_level = st.sidebar.slider('Confidence Level', 0.90, 0.99, 0.95, 0.01)
selected_segment = st.sidebar.selectbox('Segment', ['All'] + df['segment'].unique().tolist())

# Filter by segment
if selected_segment != 'All':
    df_filtered = df[df['segment'] == selected_segment]
else:
    df_filtered = df

# Calculate metrics
variants = df_filtered['variant'].unique()
results = {}

for variant in variants:
    rate, conversions, users = calculate_conversion_rate(df_filtered, variant)
    ci_lower, ci_upper = calculate_confidence_interval(conversions, users, confidence_level)

    results[variant] = {
        'rate': rate,
        'conversions': conversions,
        'users': users,
        'ci_lower': ci_lower,
        'ci_upper': ci_upper,
        'revenue': df_filtered[df_filtered['variant'] == variant]['revenue'].sum()
    }

# Display key metrics
st.subheader('📊 Summary Metrics')

cols = st.columns(len(variants))
for idx, variant in enumerate(variants):
    with cols[idx]:
        st.metric(
            f'Variant {variant} Conversion Rate',
            f"{results[variant]['rate']:.2%}",
            f"{results[variant]['conversions']:,} / {results[variant]['users']:,}"
        )

# Statistical significance test
if len(variants) >= 2:
    sig_results = calculate_statistical_significance(df_filtered, variants[0], variants[1])

    col1, col2, col3 = st.columns(3)
    col1.metric('P-Value', f"{sig_results['p_value']:.4f}")
    col2.metric('Relative Uplift', f"{sig_results['relative_uplift']:.2f}%")

    if sig_results['significant']:
        col3.success('✅ Statistically Significant')
    else:
        col3.warning('⚠️ Not Significant')
```

### Step 5: Visualization - Conversion Rate Comparison
```python
st.subheader('Conversion Rate Comparison')

# Create bar chart with confidence intervals
fig = go.Figure()

for variant in variants:
    fig.add_trace(go.Bar(
        name=f'Variant {variant}',
        x=[f'Variant {variant}'],
        y=[results[variant]['rate']],
        error_y=dict(
            type='data',
            symmetric=False,
            array=[results[variant]['ci_upper'] - results[variant]['rate']],
            arrayminus=[results[variant]['rate'] - results[variant]['ci_lower']]
        )
    ))

fig.update_layout(
    title=f'Conversion Rate with {int(confidence_level*100)}% Confidence Intervals',
    yaxis_title='Conversion Rate',
    yaxis=dict(tickformat='.2%'),
    showlegend=False
)

st.plotly_chart(fig, use_container_width=True)
```

### Step 6: Conversion Over Time
```python
st.subheader('Conversion Rate Over Time')

# Group by date and variant
df_filtered['date'] = pd.to_datetime(df_filtered['date']).dt.date
daily_conv = df_filtered.groupby(['date', 'variant']).agg({
    'converted': ['sum', 'count']
}).reset_index()

daily_conv.columns = ['date', 'variant', 'conversions', 'users']
daily_conv['rate'] = daily_conv['conversions'] / daily_conv['users']

fig = px.line(
    daily_conv,
    x='date',
    y='rate',
    color='variant',
    title='Daily Conversion Rate Trend',
    labels={'rate': 'Conversion Rate', 'date': 'Date'}
)

fig.update_yaxis(tickformat='.2%')
st.plotly_chart(fig, use_container_width=True)
```

### Step 7: Distribution Comparison
```python
st.subheader('Revenue Distribution Comparison')

# Filter to only converted users
converted_df = df_filtered[df_filtered['converted'] == 1]

fig = go.Figure()

for variant in variants:
    variant_revenue = converted_df[converted_df['variant'] == variant]['revenue']

    fig.add_trace(go.Histogram(
        x=variant_revenue,
        name=f'Variant {variant}',
        opacity=0.7,
        nbinsx=30
    ))

fig.update_layout(
    title='Revenue Distribution (Converted Users Only)',
    xaxis_title='Revenue',
    yaxis_title='Count',
    barmode='overlay'
)

st.plotly_chart(fig, use_container_width=True)

# Revenue statistics
col1, col2 = st.columns(2)

for idx, variant in enumerate(variants):
    variant_revenue = converted_df[converted_df['variant'] == variant]['revenue']

    with (col1 if idx == 0 else col2):
        st.write(f'**Variant {variant} Revenue Stats:**')
        st.write(f'Mean: ${variant_revenue.mean():.2f}')
        st.write(f'Median: ${variant_revenue.median():.2f}')
        st.write(f'Total: ${variant_revenue.sum():,.2f}')
```

### Step 8: Sample Size Calculator
```python
st.subheader('📐 Sample Size Calculator')

col1, col2, col3 = st.columns(3)

with col1:
    baseline_rate = st.number_input('Baseline Conversion Rate', 0.01, 0.50, 0.10, 0.01, format='%.2f')

with col2:
    mde = st.number_input('Minimum Detectable Effect (MDE)', 0.05, 0.50, 0.20, 0.05, format='%.2f', help='Relative change you want to detect (e.g., 0.20 = 20% improvement)')

with col3:
    power = st.number_input('Statistical Power', 0.70, 0.95, 0.80, 0.05, format='%.2f')

required_n = calculate_sample_size(baseline_rate, mde, power)

st.info(f'📊 Required sample size per variant: **{required_n:,}** users')

current_n = len(df_filtered[df_filtered['variant'] == variants[0]])
if current_n >= required_n:
    st.success(f'✅ Current sample size ({current_n:,}) meets the requirement!')
else:
    st.warning(f'⚠️ Need {required_n - current_n:,} more users per variant')
```

### Step 9: Business Impact
```python
st.subheader('💰 Business Impact Analysis')

if sig_results['significant']:
    winner = variants[1] if sig_results['relative_uplift'] > 0 else variants[0]
    loser = variants[0] if winner == variants[1] else variants[1]

    st.success(f'🏆 Winner: Variant {winner}')

    # Projected impact
    annual_users = st.number_input('Projected Annual Users', 100000, 10000000, 1000000, 100000)

    control_conversions = annual_users * results[loser]['rate']
    treatment_conversions = annual_users * results[winner]['rate']
    additional_conversions = treatment_conversions - control_conversions

    avg_revenue_per_conversion = converted_df[converted_df['variant'] == winner]['revenue'].mean()
    additional_revenue = additional_conversions * avg_revenue_per_conversion

    col1, col2, col3 = st.columns(3)
    col1.metric('Additional Annual Conversions', f'{additional_conversions:,.0f}')
    col2.metric('Additional Annual Revenue', f'${additional_revenue:,.2f}')
    col3.metric('Revenue Uplift', f'{sig_results["relative_uplift"]:.1f}%')

else:
    st.info('🔍 No statistically significant winner yet. Consider running the test longer.')
```

## Expected Outputs

1. **Summary Dashboard**:
   - Conversion rates with confidence intervals
   - Statistical significance indicator
   - Relative uplift percentage

2. **Visualizations**:
   - Bar chart with error bars
   - Conversion trends over time
   - Revenue distribution comparison

3. **Statistical Analysis**:
   - P-values and significance tests
   - Effect size calculations
   - Power analysis

4. **Sample Size Calculator**:
   - Required sample size
   - Current progress indicator

5. **Business Impact**:
   - Winner declaration
   - Projected revenue impact
   - Recommendations

## Bonus Challenges

- [ ] Add Bayesian A/B testing analysis
- [ ] Implement sequential testing (early stopping)
- [ ] Add multi-armed bandit simulation
- [ ] Create funnel analysis visualization
- [ ] Add segment-wise results breakdown
- [ ] Implement novelty effect detection
- [ ] Add confidence in recommendation metric
- [ ] Create automated email reports
- [ ] Add meta-analysis of historical tests
- [ ] Implement cost-benefit analysis

## Resources

- [Evan Miller's A/B Tools](https://www.evanmiller.org/ab-testing/)
- [Statistical Power](https://en.wikipedia.org/wiki/Power_of_a_test)
- [Wilson Score Interval](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval#Wilson_score_interval)
- [Optimizely Stats Engine](https://www.optimizely.com/insights/blog/stats-engine/)

## Success Criteria

- Conversion rates calculated correctly
- Confidence intervals displayed properly
- Statistical significance tested accurately
- Visualizations clearly show differences
- Sample size calculator works correctly
- Business impact is meaningful and actionable
- Dashboard is intuitive for non-technical users
