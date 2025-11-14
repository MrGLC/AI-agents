# Project 3: Customer Segmentation Visualization

## Overview
Build an interactive visualization tool for customer segmentation analysis using clustering algorithms, showing customer groups, their characteristics, and actionable insights.

## Difficulty Level
Intermediate

## Learning Objectives
- Apply clustering algorithms (K-Means, DBSCAN, Hierarchical)
- Visualize high-dimensional data in 2D/3D
- Create cluster profiles and interpretations
- Build interactive cluster explorers
- Generate business insights from segments

## Technical Stack
- **Backend**: Python
- **ML Framework**: scikit-learn
- **Visualization**: Plotly, Seaborn
- **Dashboard**: Streamlit or Dash
- **Data Processing**: Pandas, NumPy
- **Dimensionality Reduction**: PCA, t-SNE, UMAP
- **Dataset**: E-commerce customers, retail transactions, or marketing data

## Project Requirements

### 1. Data Preparation
- Load customer data with multiple features
- Handle missing values and outliers
- Feature scaling and normalization
- Feature engineering (RFM analysis, customer lifetime value)

### 2. Clustering Analysis
- Implement multiple clustering algorithms
- Determine optimal number of clusters (elbow method, silhouette score)
- Assign customers to segments
- Calculate cluster statistics

### 3. Core Visualizations
- **2D Scatter Plot**: Customers colored by cluster (using PCA/t-SNE)
- **3D Scatter Plot**: Interactive 3D visualization
- **Cluster Profiles**: Radar/spider charts for each segment
- **Distribution Plots**: Feature distributions by cluster
- **Dendrogram**: Hierarchical clustering tree
- **Cluster Size**: Pie chart or bar chart of segment sizes

### 4. Interactive Features
- Algorithm selector (K-Means, DBSCAN, etc.)
- Number of clusters slider
- Feature selector for analysis
- Cluster detail view on click
- Filter customers by segment
- Export segment lists

### 5. Business Insights
- Segment descriptions and naming
- Key characteristics of each segment
- Recommended actions for each group
- Profitability analysis by segment

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy scikit-learn plotly seaborn umap-learn
```

### Step 2: Load and Prepare Data
```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler

# Load customer data
df = pd.read_csv('customers.csv')

# Feature engineering - RFM
df['recency'] = (pd.Timestamp.now() - pd.to_datetime(df['last_purchase'])).dt.days
df['frequency'] = df['num_purchases']
df['monetary'] = df['total_spent']

# Select features for clustering
features = ['recency', 'frequency', 'monetary', 'avg_order_value', 'age']
X = df[features]

# Scale features
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)
```

### Step 3: Implement Clustering
```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score

def perform_clustering(X, n_clusters=4, algorithm='kmeans'):
    if algorithm == 'kmeans':
        model = KMeans(n_clusters=n_clusters, random_state=42)
    elif algorithm == 'dbscan':
        from sklearn.cluster import DBSCAN
        model = DBSCAN(eps=0.5, min_samples=5)

    clusters = model.fit_predict(X)

    # Calculate metrics
    if len(np.unique(clusters)) > 1:
        silhouette = silhouette_score(X, clusters)
    else:
        silhouette = 0

    return clusters, silhouette
```

### Step 4: Dimensionality Reduction for Visualization
```python
from sklearn.decomposition import PCA
from sklearn.manifold import TSNE

# PCA for 2D visualization
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X_scaled)

# t-SNE for better separation
tsne = TSNE(n_components=2, random_state=42)
X_tsne = tsne.fit_transform(X_scaled)
```

### Step 5: Create Interactive Dashboard
```python
import streamlit as st
import plotly.express as px

st.title('🎯 Customer Segmentation Analysis')

# Sidebar controls
algorithm = st.sidebar.selectbox('Algorithm', ['K-Means', 'DBSCAN', 'Hierarchical'])
n_clusters = st.sidebar.slider('Number of Clusters', 2, 10, 4)
viz_method = st.sidebar.selectbox('Visualization', ['PCA', 't-SNE', 'UMAP'])

# Perform clustering
clusters, silhouette = perform_clustering(X_scaled, n_clusters, algorithm.lower())

# Add clusters to dataframe
df['Cluster'] = clusters

# Create visualization
fig = px.scatter(
    x=X_pca[:, 0],
    y=X_pca[:, 1],
    color=clusters.astype(str),
    title=f'Customer Segments ({algorithm})',
    labels={'x': 'Component 1', 'y': 'Component 2'},
    hover_data={'customer_id': df['customer_id']}
)

st.plotly_chart(fig)

# Display silhouette score
st.metric('Silhouette Score', f'{silhouette:.3f}')
```

### Step 6: Cluster Profiling
```python
# Calculate cluster statistics
cluster_profiles = df.groupby('Cluster')[features].mean()

# Create radar chart for each cluster
import plotly.graph_objects as go

fig = go.Figure()

for cluster in cluster_profiles.index:
    fig.add_trace(go.Scatterpolar(
        r=cluster_profiles.loc[cluster].values,
        theta=features,
        fill='toself',
        name=f'Cluster {cluster}'
    ))

fig.update_layout(
    polar=dict(radialaxis=dict(visible=True)),
    title='Cluster Profiles'
)

st.plotly_chart(fig)
```

## Expected Outputs

1. **Cluster Visualization**:
   - 2D scatter plot with color-coded segments
   - Interactive hover showing customer details
   - Clear cluster separation

2. **Cluster Profiles**:
   - Radar charts showing segment characteristics
   - Summary statistics table
   - Cluster size distribution

3. **Analysis Metrics**:
   - Silhouette score
   - Within-cluster sum of squares
   - Cluster separation visualization

4. **Business Insights**:
   - Named segments (e.g., "High-Value Loyalists")
   - Recommended marketing strategies
   - Segment comparison table

## Bonus Challenges

- [ ] Add elbow method plot for optimal K
- [ ] Implement custom distance metrics
- [ ] Add customer journey visualization by segment
- [ ] Create segment migration analysis over time
- [ ] Build predictive model for new customer assignment
- [ ] Add A/B test simulator for segment targeting
- [ ] Include geographic distribution of segments
- [ ] Generate automated segment descriptions using AI
- [ ] Export marketing campaign lists
- [ ] Add RFM score visualization

## Resources

- [scikit-learn Clustering](https://scikit-learn.org/stable/modules/clustering.html)
- [Plotly Scatter Plots](https://plotly.com/python/line-and-scatter/)
- [Customer Segmentation Guide](https://www.optimove.com/resources/learning-center/customer-segmentation)
- [UMAP Documentation](https://umap-learn.readthedocs.io/)

## Success Criteria

- Can load and preprocess customer data
- Clustering algorithms produce meaningful segments
- Visualizations clearly show cluster separation
- Cluster profiles are interpretable and actionable
- Interactive controls allow exploration
- Business insights are generated for each segment
- Dashboard is responsive and user-friendly
