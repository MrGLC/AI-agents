# Project 9: Neural Network Training Monitor

## Overview
Build a comprehensive visualization dashboard for monitoring deep learning model training in real-time, showing loss curves, accuracy metrics, learning rate schedules, gradient flows, and layer activation visualizations.

## Difficulty Level
Advanced

## Learning Objectives
- Monitor neural network training in real-time
- Visualize loss and accuracy curves
- Analyze gradient flow and vanishing gradients
- Display learning rate schedules
- Visualize layer activations and weights
- Detect training issues (overfitting, underfitting)
- Create confusion matrices and classification reports
- Implement early stopping visualization

## Technical Stack
- **Backend**: Python
- **Deep Learning**: TensorFlow/Keras or PyTorch
- **Visualization**: Plotly, Matplotlib, TensorBoard
- **Dashboard**: Streamlit or Dash
- **Data Processing**: Pandas, NumPy
- **Dataset**: MNIST, CIFAR-10, or custom dataset

## Project Requirements

### 1. Training Metrics Tracking
- Loss (training and validation)
- Accuracy (training and validation)
- Learning rate
- Batch metrics
- Epoch timing
- GPU utilization (if applicable)

### 2. Core Visualizations
- **Loss Curves**: Training vs validation loss over epochs
- **Accuracy Curves**: Training vs validation accuracy
- **Learning Rate Schedule**: LR over epochs/iterations
- **Gradient Histograms**: Distribution of gradients by layer
- **Weight Distributions**: Histograms of weights by layer
- **Confusion Matrix**: Final model predictions
- **ROC/PR Curves**: Classification performance
- **Sample Predictions**: Visualize predictions on test samples

### 3. Advanced Monitoring
- Gradient flow visualization
- Layer activation analysis
- Filter/kernel visualization (CNNs)
- Attention weights (Transformers)
- Overfitting detection
- Training stability indicators

### 4. Interactive Features
- Real-time updates during training
- Pause/resume training
- Adjust hyperparameters mid-training
- Layer selector for detailed analysis
- Epoch slider for historical view
- Export training logs

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas numpy tensorflow plotly matplotlib seaborn scikit-learn
# or for PyTorch: pip install torch torchvision
```

### Step 2: Create a Training Logger
```python
import numpy as np
import pandas as pd
from collections import defaultdict
import time

class TrainingMonitor:
    """Monitor and log training metrics"""

    def __init__(self):
        self.history = defaultdict(list)
        self.current_epoch = 0
        self.start_time = None

    def on_train_begin(self):
        """Called at the beginning of training"""
        self.start_time = time.time()
        print('Training started...')

    def on_epoch_end(self, epoch, logs=None):
        """Called at the end of each epoch"""
        self.current_epoch = epoch

        if logs:
            for key, value in logs.items():
                self.history[key].append(value)

            # Add timing
            self.history['epoch'].append(epoch)
            self.history['time'].append(time.time() - self.start_time)

    def on_train_end(self):
        """Called at the end of training"""
        print(f'Training completed in {time.time() - self.start_time:.2f}s')

    def get_history_df(self):
        """Get history as DataFrame"""
        return pd.DataFrame(self.history)
```

### Step 3: Build Neural Network (TensorFlow/Keras Example)
```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

# Load dataset
(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()

# Preprocess
x_train = x_train.reshape(-1, 28, 28, 1).astype('float32') / 255
x_test = x_test.reshape(-1, 28, 28, 1).astype('float32') / 255

y_train = keras.utils.to_categorical(y_train, 10)
y_test = keras.utils.to_categorical(y_test, 10)

# Build model
def create_model(learning_rate=0.001):
    model = keras.Sequential([
        layers.Conv2D(32, 3, activation='relu', input_shape=(28, 28, 1), name='conv1'),
        layers.MaxPooling2D(2, name='pool1'),
        layers.Conv2D(64, 3, activation='relu', name='conv2'),
        layers.MaxPooling2D(2, name='pool2'),
        layers.Flatten(name='flatten'),
        layers.Dense(128, activation='relu', name='dense1'),
        layers.Dropout(0.5, name='dropout'),
        layers.Dense(10, activation='softmax', name='output')
    ])

    model.compile(
        optimizer=keras.optimizers.Adam(learning_rate),
        loss='categorical_crossentropy',
        metrics=['accuracy']
    )

    return model

model = create_model()
```

### Step 4: Custom Callback for Detailed Logging
```python
class DetailedMonitor(keras.callbacks.Callback):
    """Custom callback to log detailed metrics"""

    def __init__(self):
        super().__init__()
        self.epoch_logs = []
        self.batch_logs = []
        self.gradients = []
        self.weights = []

    def on_epoch_end(self, epoch, logs=None):
        """Log epoch metrics"""
        logs['epoch'] = epoch
        logs['learning_rate'] = float(keras.backend.get_value(self.model.optimizer.lr))
        self.epoch_logs.append(logs.copy())

    def on_batch_end(self, batch, logs=None):
        """Log batch metrics (sample every N batches)"""
        if batch % 100 == 0:
            self.batch_logs.append({
                'batch': batch,
                'loss': logs.get('loss'),
                'accuracy': logs.get('accuracy')
            })

    def log_gradients(self):
        """Log gradient statistics"""
        gradients = {}
        for layer in self.model.layers:
            if hasattr(layer, 'kernel'):
                weights = layer.kernel
                grads = keras.backend.gradients(self.model.total_loss, weights)[0]
                if grads is not None:
                    grad_values = keras.backend.get_value(grads)
                    gradients[layer.name] = {
                        'mean': np.mean(np.abs(grad_values)),
                        'std': np.std(grad_values),
                        'max': np.max(np.abs(grad_values))
                    }
        return gradients
```

### Step 5: Create Real-Time Dashboard
```python
import streamlit as st
import plotly.graph_objects as go
from plotly.subplots import make_subplots

st.set_page_config(page_title='NN Training Monitor', layout='wide')

st.title('🧠 Neural Network Training Monitor')

# Sidebar
st.sidebar.header('Training Configuration')
learning_rate = st.sidebar.slider('Learning Rate', 0.0001, 0.01, 0.001, 0.0001, format='%.4f')
epochs = st.sidebar.slider('Epochs', 1, 50, 10)
batch_size = st.sidebar.slider('Batch Size', 16, 256, 32)

# Initialize session state
if 'training_complete' not in st.session_state:
    st.session_state.training_complete = False
    st.session_state.history = None
    st.session_state.model = None
    st.session_state.monitor = None

# Training button
if st.sidebar.button('Start Training'):
    st.session_state.training_complete = False

    # Create model
    model = create_model(learning_rate)
    monitor = DetailedMonitor()

    # Progress tracking
    progress_bar = st.progress(0)
    status_text = st.empty()
    metrics_placeholder = st.empty()
    chart_placeholder = st.empty()

    # Training with callbacks
    history = model.fit(
        x_train, y_train,
        batch_size=batch_size,
        epochs=epochs,
        validation_split=0.2,
        callbacks=[
            monitor,
            keras.callbacks.LambdaCallback(
                on_epoch_end=lambda epoch, logs: (
                    progress_bar.progress((epoch + 1) / epochs),
                    status_text.text(f'Epoch {epoch + 1}/{epochs}'),
                    display_metrics(metrics_placeholder, pd.DataFrame(monitor.epoch_logs)),
                    display_charts(chart_placeholder, pd.DataFrame(monitor.epoch_logs))
                )
            )
        ],
        verbose=0
    )

    st.session_state.training_complete = True
    st.session_state.history = pd.DataFrame(monitor.epoch_logs)
    st.session_state.model = model
    st.session_state.monitor = monitor

    st.success('Training completed!')

def display_metrics(placeholder, history_df):
    """Display current metrics"""
    if len(history_df) == 0:
        return

    latest = history_df.iloc[-1]

    with placeholder.container():
        col1, col2, col3, col4 = st.columns(4)
        col1.metric('Train Loss', f"{latest['loss']:.4f}")
        col2.metric('Train Acc', f"{latest['accuracy']:.4f}")
        col3.metric('Val Loss', f"{latest['val_loss']:.4f}")
        col4.metric('Val Acc', f"{latest['val_accuracy']:.4f}")

def display_charts(placeholder, history_df):
    """Display training charts"""
    if len(history_df) == 0:
        return

    with placeholder.container():
        # Create subplots
        fig = make_subplots(
            rows=1, cols=2,
            subplot_titles=('Loss', 'Accuracy')
        )

        # Loss plot
        fig.add_trace(
            go.Scatter(x=history_df['epoch'], y=history_df['loss'],
                      name='Train Loss', line=dict(color='blue')),
            row=1, col=1
        )
        fig.add_trace(
            go.Scatter(x=history_df['epoch'], y=history_df['val_loss'],
                      name='Val Loss', line=dict(color='red')),
            row=1, col=1
        )

        # Accuracy plot
        fig.add_trace(
            go.Scatter(x=history_df['epoch'], y=history_df['accuracy'],
                      name='Train Acc', line=dict(color='blue')),
            row=1, col=2
        )
        fig.add_trace(
            go.Scatter(x=history_df['epoch'], y=history_df['val_accuracy'],
                      name='Val Acc', line=dict(color='red')),
            row=1, col=2
        )

        fig.update_xaxes(title_text='Epoch')
        fig.update_layout(height=400, showlegend=True)

        st.plotly_chart(fig, use_container_width=True)
```

### Step 6: Post-Training Analysis
```python
if st.session_state.training_complete:
    st.subheader('📊 Training Analysis')

    history_df = st.session_state.history
    model = st.session_state.model

    # Detailed charts
    tab1, tab2, tab3, tab4 = st.tabs([
        'Loss & Accuracy',
        'Learning Rate',
        'Model Architecture',
        'Predictions'
    ])

    with tab1:
        col1, col2 = st.columns(2)

        with col1:
            fig = go.Figure()
            fig.add_trace(go.Scatter(
                x=history_df['epoch'],
                y=history_df['loss'],
                name='Training Loss',
                line=dict(color='blue')
            ))
            fig.add_trace(go.Scatter(
                x=history_df['epoch'],
                y=history_df['val_loss'],
                name='Validation Loss',
                line=dict(color='red')
            ))

            # Highlight minimum validation loss
            min_val_loss_idx = history_df['val_loss'].idxmin()
            fig.add_vline(
                x=history_df.loc[min_val_loss_idx, 'epoch'],
                line_dash='dash',
                line_color='green',
                annotation_text='Best Val Loss'
            )

            fig.update_layout(
                title='Loss Over Epochs',
                xaxis_title='Epoch',
                yaxis_title='Loss'
            )
            st.plotly_chart(fig, use_container_width=True)

        with col2:
            fig = go.Figure()
            fig.add_trace(go.Scatter(
                x=history_df['epoch'],
                y=history_df['accuracy'],
                name='Training Accuracy',
                line=dict(color='blue')
            ))
            fig.add_trace(go.Scatter(
                x=history_df['epoch'],
                y=history_df['val_accuracy'],
                name='Validation Accuracy',
                line=dict(color='red')
            ))

            fig.update_layout(
                title='Accuracy Over Epochs',
                xaxis_title='Epoch',
                yaxis_title='Accuracy'
            )
            st.plotly_chart(fig, use_container_width=True)

        # Overfitting detection
        final_train_loss = history_df['loss'].iloc[-1]
        final_val_loss = history_df['val_loss'].iloc[-1]

        if final_val_loss > final_train_loss * 1.2:
            st.warning('⚠️ Potential overfitting detected! Validation loss is significantly higher than training loss.')
        elif final_train_loss > 0.5:
            st.warning('⚠️ Potential underfitting! Training loss is still high.')
        else:
            st.success('✅ Training looks good!')

    with tab2:
        if 'learning_rate' in history_df.columns:
            fig = go.Figure()
            fig.add_trace(go.Scatter(
                x=history_df['epoch'],
                y=history_df['learning_rate'],
                name='Learning Rate',
                line=dict(color='purple')
            ))

            fig.update_layout(
                title='Learning Rate Schedule',
                xaxis_title='Epoch',
                yaxis_title='Learning Rate',
                yaxis_type='log'
            )
            st.plotly_chart(fig, use_container_width=True)

    with tab3:
        st.write('**Model Architecture:**')

        # Display model summary
        stringlist = []
        model.summary(print_fn=lambda x: stringlist.append(x))
        model_summary = '\n'.join(stringlist)
        st.code(model_summary, language='text')

        # Layer weights visualization
        st.write('**Layer Weights Distribution:**')

        selected_layer = st.selectbox(
            'Select Layer',
            [layer.name for layer in model.layers if len(layer.get_weights()) > 0]
        )

        for layer in model.layers:
            if layer.name == selected_layer:
                weights = layer.get_weights()[0].flatten()

                fig = go.Figure()
                fig.add_trace(go.Histogram(x=weights, nbinsx=50))
                fig.update_layout(
                    title=f'Weight Distribution - {layer.name}',
                    xaxis_title='Weight Value',
                    yaxis_title='Count'
                )
                st.plotly_chart(fig, use_container_width=True)

                # Weight statistics
                col1, col2, col3 = st.columns(3)
                col1.metric('Mean', f'{weights.mean():.4f}')
                col2.metric('Std', f'{weights.std():.4f}')
                col3.metric('Range', f'[{weights.min():.4f}, {weights.max():.4f}]')

    with tab4:
        st.write('**Model Predictions on Test Set:**')

        # Make predictions
        y_pred = model.predict(x_test[:1000])
        y_pred_classes = np.argmax(y_pred, axis=1)
        y_true_classes = np.argmax(y_test[:1000], axis=1)

        # Confusion matrix
        from sklearn.metrics import confusion_matrix, classification_report

        cm = confusion_matrix(y_true_classes, y_pred_classes)

        fig = go.Figure(data=go.Heatmap(
            z=cm,
            x=[str(i) for i in range(10)],
            y=[str(i) for i in range(10)],
            colorscale='Blues',
            text=cm,
            texttemplate='%{text}',
            textfont={"size": 10}
        ))

        fig.update_layout(
            title='Confusion Matrix',
            xaxis_title='Predicted',
            yaxis_title='Actual',
            width=600,
            height=600
        )

        st.plotly_chart(fig, use_container_width=True)

        # Classification report
        st.write('**Classification Report:**')
        report = classification_report(y_true_classes, y_pred_classes, output_dict=True)
        st.dataframe(pd.DataFrame(report).transpose())

        # Sample predictions
        st.write('**Sample Predictions:**')

        n_samples = st.slider('Number of samples to show', 5, 20, 10)

        cols = st.columns(5)
        for i in range(n_samples):
            with cols[i % 5]:
                st.image(x_test[i].reshape(28, 28), width=100, caption=f'True: {y_true_classes[i]}, Pred: {y_pred_classes[i]}')
```

## Expected Outputs

1. **Real-Time Training Monitor**:
   - Live loss and accuracy curves
   - Current metrics display
   - Progress bar

2. **Post-Training Analysis**:
   - Detailed loss/accuracy plots
   - Best epoch indicator
   - Overfitting/underfitting detection

3. **Model Inspection**:
   - Architecture summary
   - Weight distributions by layer
   - Learning rate schedule

4. **Prediction Analysis**:
   - Confusion matrix
   - Classification report
   - Sample predictions visualization

## Bonus Challenges

- [ ] Add TensorBoard integration
- [ ] Implement gradient flow visualization
- [ ] Add layer activation visualization
- [ ] Create filter/kernel visualization for CNNs
- [ ] Add early stopping with visual indicator
- [ ] Implement learning rate finder
- [ ] Add model checkpointing visualization
- [ ] Create attention heatmaps (for attention models)
- [ ] Add batch normalization statistics
- [ ] Implement custom metrics tracking

## Resources

- [TensorFlow Callbacks](https://www.tensorflow.org/guide/keras/custom_callback)
- [PyTorch Hooks](https://pytorch.org/tutorials/beginner/former_torchies/nnft_tutorial.html#forward-and-backward-function-hooks)
- [TensorBoard Documentation](https://www.tensorflow.org/tensorboard)
- [Visualizing Neural Networks](https://distill.pub/)

## Success Criteria

- Training metrics update in real-time
- Loss and accuracy curves display correctly
- Overfitting is detected when present
- Weight distributions are meaningful
- Confusion matrix matches model performance
- Dashboard is responsive during training
- All visualizations render correctly
