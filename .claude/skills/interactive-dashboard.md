# Interactive Dashboard Skill

This skill helps you create interactive dashboards for machine learning models and data exploration.

## Dashboard Frameworks

- **Streamlit**: Quick prototyping, Python-focused
- **Dash (Plotly)**: Production-grade, highly customizable
- **Gradio**: Model demos and interfaces
- **Panel (HoloViz)**: Flexible, supports multiple viz libraries
- **Voila**: Turn Jupyter notebooks into dashboards
- **Custom (React/Vue)**: Full control, more development time

## When to Use This Skill

- Creating ML model demos
- Building internal tools for data scientists
- Visualizing model predictions in real-time
- Exploring datasets interactively
- Monitoring model performance
- A/B testing different models
- Sharing results with stakeholders

## Streamlit Dashboards

### Basic ML Dashboard

```python
import streamlit as st
import pandas as pd
import numpy as np
import plotly.express as px
import joblib

# Load model
@st.cache_resource
def load_model():
    return joblib.load('model.pkl')

model = load_model()

# Title and description
st.title('🤖 ML Model Dashboard')
st.write('Interactive predictions and visualizations')

# Sidebar for inputs
st.sidebar.header('Input Features')
feature1 = st.sidebar.slider('Feature 1', 0.0, 10.0, 5.0)
feature2 = st.sidebar.slider('Feature 2', 0.0, 10.0, 5.0)
feature3 = st.sidebar.slider('Feature 3', 0.0, 10.0, 5.0)

# Make prediction
features = np.array([[feature1, feature2, feature3]])
prediction = model.predict(features)[0]
probability = model.predict_proba(features)[0]

# Display results
col1, col2 = st.columns(2)
with col1:
    st.metric('Prediction', f'{prediction:.2f}')
with col2:
    st.metric('Confidence', f'{probability.max():.2%}')

# Visualization
fig = px.bar(
    x=['Class 0', 'Class 1'],
    y=probability,
    labels={'x': 'Class', 'y': 'Probability'},
    title='Prediction Probabilities'
)
st.plotly_chart(fig)

# Feature importance
if st.checkbox('Show Feature Importance'):
    importance = model.feature_importances_
    fig2 = px.bar(
        x=['Feature 1', 'Feature 2', 'Feature 3'],
        y=importance,
        title='Feature Importance'
    )
    st.plotly_chart(fig2)
```

### File Upload Dashboard

```python
import streamlit as st
import pandas as pd

st.title('📊 Data Analysis Dashboard')

uploaded_file = st.file_uploader('Upload CSV file', type=['csv'])

if uploaded_file is not None:
    df = pd.read_csv(uploaded_file)

    st.subheader('Data Preview')
    st.dataframe(df.head())

    st.subheader('Summary Statistics')
    st.write(df.describe())

    # Select columns for visualization
    numeric_cols = df.select_dtypes(include=['float64', 'int64']).columns
    col1 = st.selectbox('X-axis', numeric_cols)
    col2 = st.selectbox('Y-axis', numeric_cols)

    # Create scatter plot
    fig = px.scatter(df, x=col1, y=col2, title=f'{col1} vs {col2}')
    st.plotly_chart(fig)
```

## Dash Dashboards

### Production-Ready Dashboard

```python
import dash
from dash import dcc, html, Input, Output
import plotly.express as px
import pandas as pd
import numpy as np

app = dash.Dash(__name__)

app.layout = html.Div([
    html.H1('ML Model Dashboard'),

    html.Div([
        html.Label('Feature 1'),
        dcc.Slider(id='feature1', min=0, max=10, value=5, step=0.1),

        html.Label('Feature 2'),
        dcc.Slider(id='feature2', min=0, max=10, value=5, step=0.1),

        html.Label('Feature 3'),
        dcc.Slider(id='feature3', min=0, max=10, value=5, step=0.1),
    ], style={'width': '30%', 'display': 'inline-block'}),

    html.Div([
        html.H3(id='prediction-output'),
        dcc.Graph(id='probability-chart'),
        dcc.Graph(id='feature-importance'),
    ], style={'width': '65%', 'display': 'inline-block', 'float': 'right'}),
])

@app.callback(
    [Output('prediction-output', 'children'),
     Output('probability-chart', 'figure'),
     Output('feature-importance', 'figure')],
    [Input('feature1', 'value'),
     Input('feature2', 'value'),
     Input('feature3', 'value')]
)
def update_output(f1, f2, f3):
    # Make prediction
    features = np.array([[f1, f2, f3]])
    prediction = model.predict(features)[0]
    probability = model.predict_proba(features)[0]

    # Probability chart
    prob_fig = px.bar(
        x=['Class 0', 'Class 1'],
        y=probability,
        title='Prediction Probabilities'
    )

    # Feature importance
    importance_fig = px.bar(
        x=['Feature 1', 'Feature 2', 'Feature 3'],
        y=model.feature_importances_,
        title='Feature Importance'
    )

    return f'Prediction: {prediction:.2f}', prob_fig, importance_fig

if __name__ == '__main__':
    app.run_server(debug=True)
```

## Gradio Dashboards

### Quick Model Demo

```python
import gradio as gr
import numpy as np

def predict(feature1, feature2, feature3):
    features = np.array([[feature1, feature2, feature3]])
    prediction = model.predict(features)[0]
    probability = model.predict_proba(features)[0]

    return {
        "prediction": prediction,
        "class_0_prob": probability[0],
        "class_1_prob": probability[1]
    }

demo = gr.Interface(
    fn=predict,
    inputs=[
        gr.Slider(0, 10, label="Feature 1"),
        gr.Slider(0, 10, label="Feature 2"),
        gr.Slider(0, 10, label="Feature 3")
    ],
    outputs=gr.JSON(label="Prediction Results"),
    title="ML Model Demo",
    description="Adjust the features to see predictions"
)

demo.launch()
```

### Image Classification Demo

```python
import gradio as gr
from PIL import Image

def classify_image(image):
    # Preprocess image
    img = preprocess(image)

    # Make prediction
    prediction = model.predict(img)
    class_names = ['Cat', 'Dog']

    return {class_names[i]: float(prediction[0][i]) for i in range(len(class_names))}

demo = gr.Interface(
    fn=classify_image,
    inputs=gr.Image(type="pil"),
    outputs=gr.Label(num_top_classes=2),
    examples=['cat1.jpg', 'dog1.jpg'],
    title="Image Classifier"
)

demo.launch()
```

## Dashboard Features

### Real-Time Updates

```python
import streamlit as st
import time

placeholder = st.empty()

for i in range(100):
    # Update prediction
    prediction = get_latest_prediction()

    with placeholder.container():
        st.metric('Latest Prediction', f'{prediction:.2f}')
        st.line_chart(get_recent_predictions())

    time.sleep(1)
```

### Model Comparison

```python
import streamlit as st

st.title('Model Comparison Dashboard')

# Load multiple models
models = {
    'Model A': load_model('model_a.pkl'),
    'Model B': load_model('model_b.pkl'),
    'Model C': load_model('model_c.pkl')
}

# Input features
features = get_user_input()

# Compare predictions
results = {}
for name, model in models.items():
    results[name] = model.predict([features])[0]

# Display comparison
df = pd.DataFrame.from_dict(results, orient='index', columns=['Prediction'])
st.bar_chart(df)
```

### Interactive Data Exploration

```python
import streamlit as st
import pandas as pd
import plotly.express as px

df = pd.read_csv('data.csv')

st.title('Interactive Data Explorer')

# Filters
selected_columns = st.multiselect('Select columns', df.columns.tolist())

if selected_columns:
    filtered_df = df[selected_columns]

    # Plot type selection
    plot_type = st.selectbox('Plot type', ['scatter', 'bar', 'line', 'histogram'])

    if plot_type == 'scatter':
        x = st.selectbox('X-axis', selected_columns)
        y = st.selectbox('Y-axis', selected_columns)
        fig = px.scatter(filtered_df, x=x, y=y)
        st.plotly_chart(fig)
```

## Best Practices

### Performance
- Use `@st.cache_resource` for loading models
- Use `@st.cache_data` for loading data
- Limit data size in visualizations
- Use pagination for large tables
- Optimize model inference time

### User Experience
- Provide clear instructions
- Show loading indicators
- Display error messages gracefully
- Add examples and defaults
- Make it mobile-responsive

### Design
- Use consistent color schemes
- Organize with tabs or pages
- Add meaningful titles and labels
- Use appropriate chart types
- Include legends and axis labels

### Deployment
- Use Docker for consistency
- Set up HTTPS for production
- Implement authentication if needed
- Monitor usage and performance
- Add logging and error tracking

## Deployment Options

### Streamlit Cloud
```bash
# Deploy to Streamlit Cloud (free)
# Just push to GitHub and connect repository
```

### Heroku
```bash
# Procfile
web: streamlit run app.py --server.port=$PORT
```

### Docker
```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8501
CMD ["streamlit", "run", "app.py"]
```

### AWS/GCP/Azure
Deploy using container services (ECS, Cloud Run, App Service)
