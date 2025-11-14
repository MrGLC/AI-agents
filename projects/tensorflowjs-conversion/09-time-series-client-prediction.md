# Project 9: Convert Time Series Model for Client-Side Prediction

## Overview
Build a browser-based time series forecasting application using TensorFlow.js. Convert LSTM or GRU models trained on sequential data to run in the browser, enabling real-time predictions for stock prices, weather forecasting, sensor data, or any time-dependent patterns without server dependency.

## Difficulty Level
Advanced

## Learning Objectives
- Train LSTM/GRU models for time series forecasting
- Handle sequential data preprocessing in JavaScript
- Convert recurrent neural networks to TensorFlow.js
- Implement sliding window prediction
- Visualize time series predictions with charts
- Handle streaming data updates
- Create interactive forecasting dashboards
- Optimize RNN inference for browser performance

## Technical Stack
- **Backend**: Python 3.8+, TensorFlow/Keras
- **Model**: LSTM, GRU, or Transformer for time series
- **Conversion**: tensorflowjs_converter
- **Frontend**: HTML5, JavaScript ES6+, TensorFlow.js
- **Visualization**: Chart.js or Plotly.js
- **Data**: Stock prices, weather data, or sensor readings
- **Real-time**: WebSocket (optional) for live data

## Project Requirements

### 1. Model Training
- Train LSTM/GRU on time series data
- Implement proper data normalization
- Create sliding window sequences
- Save model and preprocessing parameters

### 2. Data Preprocessing
- Implement data normalization in JavaScript
- Create sliding window generation
- Handle missing data points
- Match Python preprocessing exactly

### 3. Forecasting Application
- Load and run TFJS model
- Generate multi-step predictions
- Update predictions in real-time
- Visualize historical and predicted data
- Display confidence intervals

### 4. Interactive Features
- User input for forecast horizon
- Adjustable lookback window
- Multiple prediction scenarios
- Export predictions to CSV
- Comparison with actual values

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
mkdir timeseries-forecasting
cd timeseries-forecasting

# Setup Python environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install tensorflow numpy pandas matplotlib scikit-learn yfinance
pip install tensorflowjs
```

### Step 2: Prepare Time Series Data
```python
# prepare_data.py
import numpy as np
import pandas as pd
import yfinance as yf
from sklearn.preprocessing import MinMaxScaler
import json

# Download stock data (example: Apple stock)
print("Downloading stock data...")
data = yf.download('AAPL', start='2020-01-01', end='2024-01-01')

# Use closing price
prices = data['Close'].values.reshape(-1, 1)

print(f"Total data points: {len(prices)}")

# Normalize data
scaler = MinMaxScaler(feature_range=(0, 1))
scaled_data = scaler.fit_transform(prices)

# Save scaler parameters
scaler_params = {
    'min': float(scaler.data_min_[0]),
    'max': float(scaler.data_max_[0]),
    'feature_range': [0, 1]
}

with open('scaler_params.json', 'w') as f:
    json.dump(scaler_params, f)

print("Scaler parameters saved")

# Create sequences
def create_sequences(data, seq_length=60):
    X, y = [], []

    for i in range(seq_length, len(data)):
        X.append(data[i-seq_length:i, 0])
        y.append(data[i, 0])

    return np.array(X), np.array(y)

SEQ_LENGTH = 60  # Use 60 days to predict next day

X, y = create_sequences(scaled_data, SEQ_LENGTH)

print(f"Sequences created: {X.shape}, {y.shape}")

# Split data
train_size = int(len(X) * 0.8)
X_train, X_test = X[:train_size], X[train_size:]
y_train, y_test = y[:train_size], y[train_size:]

# Reshape for LSTM [samples, timesteps, features]
X_train = X_train.reshape(X_train.shape[0], X_train.shape[1], 1)
X_test = X_test.reshape(X_test.shape[0], X_test.shape[1], 1)

print(f"Train shape: {X_train.shape}, Test shape: {X_test.shape}")

# Save data configuration
config = {
    'sequence_length': SEQ_LENGTH,
    'features': 1,
    'train_size': train_size,
    'total_samples': len(X)
}

with open('model_config.json', 'w') as f:
    json.dump(config, f)

# Save some actual data for testing
test_data = {
    'dates': data.index[-100:].strftime('%Y-%m-%d').tolist(),
    'prices': prices[-100:].flatten().tolist()
}

with open('test_data.json', 'w') as f:
    json.dump(test_data, f)

print("Data preparation complete!")
```

### Step 3: Build and Train LSTM Model
```python
# train_lstm.py
import tensorflow as tf
from tensorflow import keras
import numpy as np
import json
from prepare_data import X_train, X_test, y_train, y_test, SEQ_LENGTH

print("Building LSTM model...")

model = keras.Sequential([
    keras.layers.LSTM(50, return_sequences=True, input_shape=(SEQ_LENGTH, 1)),
    keras.layers.Dropout(0.2),

    keras.layers.LSTM(50, return_sequences=True),
    keras.layers.Dropout(0.2),

    keras.layers.LSTM(50),
    keras.layers.Dropout(0.2),

    keras.layers.Dense(25, activation='relu'),
    keras.layers.Dense(1)
])

model.compile(
    optimizer='adam',
    loss='mean_squared_error',
    metrics=['mae']
)

print(model.summary())

# Train model
print("\nTraining model...")
history = model.fit(
    X_train, y_train,
    batch_size=32,
    epochs=50,
    validation_split=0.1,
    verbose=1
)

# Evaluate
test_loss, test_mae = model.evaluate(X_test, y_test, verbose=0)
print(f"\nTest Loss: {test_loss:.6f}")
print(f"Test MAE: {test_mae:.6f}")

# Save model
model.save('stock_lstm.h5')
print("Model saved as stock_lstm.h5")

# Save as SavedModel for conversion
tf.saved_model.save(model, 'stock_lstm_savedmodel')
print("SavedModel created")

# Save training history
training_history = {
    'loss': [float(x) for x in history.history['loss']],
    'val_loss': [float(x) for x in history.history['val_loss']],
    'mae': [float(x) for x in history.history['mae']],
    'val_mae': [float(x) for x in history.history['val_mae']]
}

with open('training_history.json', 'w') as f:
    json.dump(training_history, f)

# Make sample predictions
print("\nMaking sample predictions...")
predictions = model.predict(X_test[:10])
actuals = y_test[:10]

print("Sample predictions vs actuals:")
for i in range(5):
    print(f"  Predicted: {predictions[i][0]:.6f}, Actual: {actuals[i]:.6f}")
```

### Step 4: Convert Model to TensorFlow.js
```bash
# Convert LSTM model to TensorFlow.js
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --quantize_float16 \
    ./stock_lstm_savedmodel \
    ./tfjs_stock_model

echo "Model converted to TensorFlow.js!"
ls -lh tfjs_stock_model/
```

### Step 5: Create HTML Interface
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Time Series Forecasting - TensorFlow.js</title>

    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0"></script>

    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }

        .container {
            max-width: 1400px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            padding: 30px;
            box-shadow: 0 10px 50px rgba(0,0,0,0.3);
        }

        h1 {
            color: #667eea;
            text-align: center;
            margin-bottom: 10px;
        }

        .subtitle {
            text-align: center;
            color: #666;
            margin-bottom: 30px;
        }

        .status {
            text-align: center;
            padding: 15px;
            background: #fff3cd;
            border-radius: 10px;
            margin-bottom: 20px;
            font-weight: 600;
        }

        .controls {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            margin: 20px 0;
            padding: 20px;
            background: #f5f5f5;
            border-radius: 10px;
        }

        .control-group {
            display: flex;
            flex-direction: column;
        }

        label {
            font-weight: 600;
            margin-bottom: 8px;
            color: #333;
        }

        input[type="number"],
        select {
            padding: 10px;
            border: 2px solid #ddd;
            border-radius: 5px;
            font-size: 16px;
        }

        button {
            padding: 12px 25px;
            font-size: 16px;
            font-weight: 600;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.3s;
        }

        .btn-primary {
            background: #4CAF50;
            color: white;
            grid-column: span 2;
        }

        .btn-primary:hover {
            background: #45a049;
        }

        .btn-secondary {
            background: #2196F3;
            color: white;
        }

        button:disabled {
            background: #ccc;
            cursor: not-allowed;
        }

        .chart-container {
            margin: 30px 0;
            padding: 20px;
            background: #f9f9f9;
            border-radius: 10px;
        }

        .predictions-summary {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 20px;
            margin: 20px 0;
        }

        .summary-card {
            background: #f5f5f5;
            padding: 20px;
            border-radius: 10px;
            text-align: center;
        }

        .summary-card h3 {
            color: #666;
            font-size: 14px;
            margin-bottom: 10px;
        }

        .summary-card .value {
            font-size: 28px;
            font-weight: 700;
            color: #667eea;
        }

        .predictions-table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
        }

        .predictions-table th,
        .predictions-table td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }

        .predictions-table th {
            background: #667eea;
            color: white;
            font-weight: 600;
        }

        .predictions-table tr:hover {
            background: #f5f5f5;
        }

        .trend-up {
            color: #4CAF50;
        }

        .trend-down {
            color: #f44336;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📈 Time Series Forecasting</h1>
        <p class="subtitle">LSTM-based stock price prediction with TensorFlow.js</p>

        <div id="status" class="status">Loading model...</div>

        <div class="controls">
            <div class="control-group">
                <label for="forecastDays">Forecast Days:</label>
                <input type="number" id="forecastDays" min="1" max="30" value="7">
            </div>

            <div class="control-group">
                <label for="lookbackDays">Lookback Period:</label>
                <input type="number" id="lookbackDays" min="30" max="120" value="60" disabled>
            </div>

            <button class="btn-primary" onclick="generateForecast()" id="forecastBtn" disabled>
                🔮 Generate Forecast
            </button>

            <button class="btn-secondary" onclick="exportPredictions()" id="exportBtn" disabled>
                📥 Export Data
            </button>
        </div>

        <div class="chart-container">
            <canvas id="forecastChart"></canvas>
        </div>

        <div class="predictions-summary" id="summary" style="display: none;">
            <div class="summary-card">
                <h3>Current Price</h3>
                <div class="value" id="currentPrice">-</div>
            </div>
            <div class="summary-card">
                <h3>Predicted (7 days)</h3>
                <div class="value" id="predictedPrice">-</div>
            </div>
            <div class="summary-card">
                <h3>Expected Change</h3>
                <div class="value" id="priceChange">-</div>
            </div>
            <div class="summary-card">
                <h3>Trend Direction</h3>
                <div class="value" id="trendDirection">-</div>
            </div>
        </div>

        <div style="overflow-x: auto;">
            <table class="predictions-table" id="predictionsTable" style="display: none;">
                <thead>
                    <tr>
                        <th>Day</th>
                        <th>Date</th>
                        <th>Predicted Price</th>
                        <th>Change from Previous</th>
                        <th>% Change</th>
                    </tr>
                </thead>
                <tbody id="predictionsBody"></tbody>
            </table>
        </div>
    </div>

    <script src="forecaster.js"></script>
</body>
</html>
```

### Step 6: Implement Forecasting Logic
```javascript
// forecaster.js
let model = null;
let scalerParams = null;
let modelConfig = null;
let historicalData = null;
let chart = null;

// Load configuration and model
async function init() {
    document.getElementById('status').textContent = 'Loading model and configuration...';

    try {
        // Load scaler parameters
        const scalerResponse = await fetch('scaler_params.json');
        scalerParams = await scalerResponse.json();

        // Load model configuration
        const configResponse = await fetch('model_config.json');
        modelConfig = await configResponse.json();

        // Load test data
        const dataResponse = await fetch('test_data.json');
        historicalData = await dataResponse.json();

        // Load TensorFlow.js model
        model = await tf.loadGraphModel('./tfjs_stock_model/model.json');

        console.log('Model loaded successfully');

        // Warm up model
        await warmUpModel();

        document.getElementById('status').textContent = 'Model ready! Click "Generate Forecast"';
        document.getElementById('status').style.background = '#d4edda';
        document.getElementById('forecastBtn').disabled = false;

        // Initialize chart
        initializeChart();

    } catch (error) {
        console.error('Initialization error:', error);
        document.getElementById('status').textContent = 'Error loading model: ' + error.message;
        document.getElementById('status').style.background = '#f8d7da';
    }
}

// Warm up model
async function warmUpModel() {
    const warmupInput = tf.zeros([1, modelConfig.sequence_length, 1]);
    await model.predict(warmupInput);
    warmupInput.dispose();
}

// Normalize data
function normalize(value) {
    const min = scalerParams.min;
    const max = scalerParams.max;
    return (value - min) / (max - min);
}

// Denormalize data
function denormalize(value) {
    const min = scalerParams.min;
    const max = scalerParams.max;
    return value * (max - min) + min;
}

// Generate forecast
async function generateForecast() {
    const forecastDays = parseInt(document.getElementById('forecastDays').value);

    document.getElementById('forecastBtn').disabled = true;
    document.getElementById('status').textContent = 'Generating forecast...';

    try {
        // Get last N days of data
        const lookbackPeriod = modelConfig.sequence_length;
        const lastPrices = historicalData.prices.slice(-lookbackPeriod);

        // Normalize
        const normalizedPrices = lastPrices.map(p => normalize(p));

        // Make predictions
        const predictions = [];
        let currentSequence = [...normalizedPrices];

        for (let i = 0; i < forecastDays; i++) {
            // Create input tensor
            const inputTensor = tf.tensor3d([currentSequence], [1, lookbackPeriod, 1]);

            // Predict
            const prediction = await model.predict(inputTensor).data();

            // Add prediction to results
            predictions.push(denormalize(prediction[0]));

            // Update sequence (sliding window)
            currentSequence.shift();
            currentSequence.push(prediction[0]);

            // Clean up
            inputTensor.dispose();
        }

        // Display results
        displayForecast(predictions, lastPrices);

        document.getElementById('status').textContent = 'Forecast complete!';
        document.getElementById('exportBtn').disabled = false;

    } catch (error) {
        console.error('Forecast error:', error);
        document.getElementById('status').textContent = 'Error generating forecast';
        document.getElementById('status').style.background = '#f8d7da';
    }

    document.getElementById('forecastBtn').disabled = false;
}

// Display forecast results
function displayForecast(predictions, historical) {
    // Update chart
    updateChart(predictions, historical);

    // Update summary
    updateSummary(predictions, historical);

    // Update table
    updateTable(predictions);

    document.getElementById('summary').style.display = 'grid';
    document.getElementById('predictionsTable').style.display = 'table';
}

// Initialize chart
function initializeChart() {
    const ctx = document.getElementById('forecastChart').getContext('2d');

    chart = new Chart(ctx, {
        type: 'line',
        data: {
            labels: [],
            datasets: [
                {
                    label: 'Historical Prices',
                    data: [],
                    borderColor: '#667eea',
                    backgroundColor: 'rgba(102, 126, 234, 0.1)',
                    tension: 0.4
                },
                {
                    label: 'Forecasted Prices',
                    data: [],
                    borderColor: '#4CAF50',
                    backgroundColor: 'rgba(76, 175, 80, 0.1)',
                    borderDash: [5, 5],
                    tension: 0.4
                }
            ]
        },
        options: {
            responsive: true,
            maintainAspectRatio: true,
            plugins: {
                legend: {
                    display: true,
                    position: 'top'
                },
                title: {
                    display: true,
                    text: 'Stock Price Forecast',
                    font: {
                        size: 18
                    }
                }
            },
            scales: {
                y: {
                    beginAtZero: false,
                    title: {
                        display: true,
                        text: 'Price ($)'
                    }
                },
                x: {
                    title: {
                        display: true,
                        text: 'Date'
                    }
                }
            }
        }
    });
}

// Update chart with new data
function updateChart(predictions, historical) {
    const historicalLength = 30;
    const recentHistorical = historical.slice(-historicalLength);
    const recentDates = historicalData.dates.slice(-historicalLength);

    // Generate future dates
    const lastDate = new Date(recentDates[recentDates.length - 1]);
    const futureDates = [];

    for (let i = 1; i <= predictions.length; i++) {
        const futureDate = new Date(lastDate);
        futureDate.setDate(lastDate.getDate() + i);
        futureDates.push(futureDate.toISOString().split('T')[0]);
    }

    // Update chart data
    chart.data.labels = [...recentDates, ...futureDates];

    chart.data.datasets[0].data = [...recentHistorical, ...Array(predictions.length).fill(null)];
    chart.data.datasets[1].data = [...Array(historicalLength).fill(null), ...predictions];

    chart.update();
}

// Update summary cards
function updateSummary(predictions, historical) {
    const currentPrice = historical[historical.length - 1];
    const predictedPrice = predictions[predictions.length - 1];
    const change = predictedPrice - currentPrice;
    const changePercent = (change / currentPrice) * 100;

    document.getElementById('currentPrice').textContent = `$${currentPrice.toFixed(2)}`;
    document.getElementById('predictedPrice').textContent = `$${predictedPrice.toFixed(2)}`;

    const changeElement = document.getElementById('priceChange');
    changeElement.textContent = `$${change.toFixed(2)} (${changePercent.toFixed(2)}%)`;
    changeElement.className = 'value ' + (change >= 0 ? 'trend-up' : 'trend-down');

    const trendElement = document.getElementById('trendDirection');
    trendElement.textContent = change >= 0 ? '📈 Bullish' : '📉 Bearish';
    trendElement.className = 'value ' + (change >= 0 ? 'trend-up' : 'trend-down');
}

// Update predictions table
function updateTable(predictions) {
    const tbody = document.getElementById('predictionsBody');
    tbody.innerHTML = '';

    const lastDate = new Date(historicalData.dates[historicalData.dates.length - 1]);
    const lastPrice = historicalData.prices[historicalData.prices.length - 1];

    predictions.forEach((price, index) => {
        const futureDate = new Date(lastDate);
        futureDate.setDate(lastDate.getDate() + index + 1);

        const previousPrice = index === 0 ? lastPrice : predictions[index - 1];
        const change = price - previousPrice;
        const changePercent = (change / previousPrice) * 100;

        const row = tbody.insertRow();
        row.innerHTML = `
            <td>${index + 1}</td>
            <td>${futureDate.toISOString().split('T')[0]}</td>
            <td>$${price.toFixed(2)}</td>
            <td class="${change >= 0 ? 'trend-up' : 'trend-down'}">
                ${change >= 0 ? '↑' : '↓'} $${Math.abs(change).toFixed(2)}
            </td>
            <td class="${change >= 0 ? 'trend-up' : 'trend-down'}">
                ${changePercent >= 0 ? '+' : ''}${changePercent.toFixed(2)}%
            </td>
        `;
    });
}

// Export predictions to CSV
function exportPredictions() {
    const forecastDays = parseInt(document.getElementById('forecastDays').value);
    const tbody = document.getElementById('predictionsBody');

    if (!tbody.rows.length) {
        alert('No predictions to export. Generate a forecast first.');
        return;
    }

    let csv = 'Day,Date,Predicted Price,Change,% Change\n';

    for (let row of tbody.rows) {
        const cells = row.cells;
        csv += `${cells[0].textContent},${cells[1].textContent},${cells[2].textContent},${cells[3].textContent},${cells[4].textContent}\n`;
    }

    // Create download link
    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `forecast-${new Date().toISOString().split('T')[0]}.csv`;
    a.click();
    URL.revokeObjectURL(url);
}

// Initialize
init();
```

## Expected Outputs

1. **Model Performance**:
   - Train MAE: <0.02 (normalized)
   - Test MAE: <0.03 (normalized)
   - Forecast accuracy within 5-10%

2. **Predictions**:
   - Multi-step forecasts (1-30 days)
   - Smooth prediction curves
   - Reasonable price trends

3. **Visualization**:
   - Interactive chart with historical/predicted data
   - Summary statistics
   - Detailed predictions table

4. **Performance**:
   - Model load: <5 seconds
   - Prediction generation: <1 second per day
   - Smooth UI updates

## Bonus Challenges

- [ ] Add confidence intervals for predictions
- [ ] Implement multiple models ensemble
- [ ] Add real-time data updates via WebSocket
- [ ] Create scenario analysis (best/worst case)
- [ ] Implement technical indicators (RSI, MACD)
- [ ] Add anomaly detection
- [ ] Create backtesting functionality
- [ ] Implement multi-variate forecasting
- [ ] Add seasonal decomposition
- [ ] Create automated trading signals

## Resources

- [Time Series Forecasting with TensorFlow](https://www.tensorflow.org/tutorials/structured_data/time_series)
- [LSTM Networks](https://colah.github.io/posts/2015-08-Understanding-LSTMs/)
- [Chart.js Documentation](https://www.chartjs.org/docs/latest/)
- [Yahoo Finance API](https://pypi.org/project/yfinance/)
- [Time Series Analysis](https://otexts.com/fpp3/)

## Success Criteria

- LSTM model trains successfully
- Model converts to TensorFlow.js without errors
- Preprocessing matches Python implementation
- Forecasts are generated correctly
- Multi-step predictions work
- Chart displays historical and predicted data
- Summary statistics are accurate
- Export functionality works
- Predictions update in real-time
- Works across different browsers
