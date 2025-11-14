# Project 2: Convert PyTorch Text Model to TensorFlow.js via ONNX

## Overview
Master the multi-step conversion pipeline from PyTorch to TensorFlow.js using ONNX as an intermediate format. This project focuses on converting a text classification model (sentiment analysis) and deploying it for browser-based inference.

## Difficulty Level
Intermediate to Advanced

## Learning Objectives
- Train a PyTorch text classification model
- Export PyTorch models to ONNX format
- Convert ONNX models to TensorFlow SavedModel
- Convert TensorFlow models to TensorFlow.js
- Handle text preprocessing in JavaScript
- Implement tokenization in the browser
- Validate multi-stage conversion pipeline

## Technical Stack
- **Backend**: Python 3.8+, PyTorch 2.0+
- **Conversion Pipeline**: ONNX, onnx-tf, tensorflowjs_converter
- **Frontend**: HTML5, JavaScript ES6+, TensorFlow.js
- **Dataset**: IMDB sentiment analysis or AG News
- **Text Processing**: JavaScript tokenization
- **Server**: Node.js or Python http-server

## Project Requirements

### 1. PyTorch Model Training
- Build LSTM or Transformer-based text classifier
- Train on sentiment analysis dataset
- Achieve >80% validation accuracy
- Export vocabulary and tokenizer configuration

### 2. Multi-Stage Conversion
- PyTorch → ONNX export
- ONNX → TensorFlow conversion
- TensorFlow → TensorFlow.js conversion
- Validate outputs at each stage

### 3. Browser Implementation
- Implement JavaScript tokenizer matching Python version
- Handle variable-length sequences
- Load and run TFJS model
- Display sentiment predictions with confidence

### 4. Testing and Validation
- Compare predictions across all formats
- Ensure numerical consistency
- Test with various input lengths
- Handle edge cases gracefully

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
# Create conda environment
conda create -n pytorch-tfjs python=3.9
conda activate pytorch-tfjs

# Install PyTorch
pip install torch torchvision torchaudio

# Install conversion tools
pip install onnx onnx-tf tensorflow tensorflowjs

# Install additional dependencies
pip install numpy pandas scikit-learn tqdm
```

### Step 2: Prepare Dataset and Tokenizer
```python
# prepare_data.py
import torch
from torch.utils.data import Dataset, DataLoader
from collections import Counter
import json
import re

class SimpleTokenizer:
    def __init__(self, vocab_size=10000):
        self.vocab_size = vocab_size
        self.word2idx = {'<PAD>': 0, '<UNK>': 1}
        self.idx2word = {0: '<PAD>', 1: '<UNK>'}

    def fit(self, texts):
        # Count word frequencies
        word_counts = Counter()
        for text in texts:
            words = self.tokenize(text)
            word_counts.update(words)

        # Build vocabulary
        most_common = word_counts.most_common(self.vocab_size - 2)
        for idx, (word, _) in enumerate(most_common, start=2):
            self.word2idx[word] = idx
            self.idx2word[idx] = word

    def tokenize(self, text):
        # Simple whitespace tokenizer
        text = text.lower()
        text = re.sub(r'[^a-zA-Z0-9\s]', '', text)
        return text.split()

    def encode(self, text, max_length=100):
        tokens = self.tokenize(text)
        indices = [self.word2idx.get(token, 1) for token in tokens]

        # Pad or truncate
        if len(indices) < max_length:
            indices += [0] * (max_length - len(indices))
        else:
            indices = indices[:max_length]

        return indices

    def save(self, filepath):
        config = {
            'word2idx': self.word2idx,
            'idx2word': self.idx2word,
            'vocab_size': self.vocab_size
        }
        with open(filepath, 'w') as f:
            json.dump(config, f)

# Example: Load IMDB dataset
def load_imdb_data():
    """
    For demonstration - replace with actual IMDB loading
    """
    # This is a placeholder - use actual IMDB dataset
    texts = [
        "This movie was great! I loved it.",
        "Terrible film, waste of time.",
        # ... more examples
    ]
    labels = [1, 0]  # 1=positive, 0=negative
    return texts, labels

# Prepare data
texts, labels = load_imdb_data()
tokenizer = SimpleTokenizer(vocab_size=10000)
tokenizer.fit(texts)
tokenizer.save('tokenizer_config.json')

print(f"Vocabulary size: {len(tokenizer.word2idx)}")
print("Sample encoding:", tokenizer.encode("This movie was great!"))
```

### Step 3: Build and Train PyTorch Model
```python
# train_pytorch_model.py
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
import json

class SentimentLSTM(nn.Module):
    def __init__(self, vocab_size, embedding_dim=128, hidden_dim=128, num_layers=2):
        super(SentimentLSTM, self).__init__()

        self.embedding = nn.Embedding(vocab_size, embedding_dim, padding_idx=0)
        self.lstm = nn.LSTM(
            embedding_dim,
            hidden_dim,
            num_layers=num_layers,
            batch_first=True,
            bidirectional=False
        )
        self.fc = nn.Linear(hidden_dim, 2)  # Binary classification

    def forward(self, x):
        # x shape: (batch_size, seq_length)
        embedded = self.embedding(x)  # (batch_size, seq_length, embedding_dim)

        # LSTM
        lstm_out, (hidden, cell) = self.lstm(embedded)

        # Use last hidden state
        output = self.fc(hidden[-1])  # (batch_size, 2)

        return output

# Initialize model
model = SentimentLSTM(vocab_size=10000, embedding_dim=128, hidden_dim=128)

# Training loop (simplified)
def train_model(model, train_loader, epochs=5):
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.Adam(model.parameters(), lr=0.001)

    model.train()
    for epoch in range(epochs):
        total_loss = 0
        for batch_idx, (data, target) in enumerate(train_loader):
            optimizer.zero_grad()
            output = model(data)
            loss = criterion(output, target)
            loss.backward()
            optimizer.step()
            total_loss += loss.item()

        avg_loss = total_loss / len(train_loader)
        print(f'Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}')

    return model

# After training, save model
# model = train_model(model, train_loader)
torch.save(model.state_dict(), 'sentiment_model.pth')
print("PyTorch model saved!")
```

### Step 4: Export PyTorch to ONNX
```python
# export_to_onnx.py
import torch
import torch.nn as nn

# Load trained model
model = SentimentLSTM(vocab_size=10000, embedding_dim=128, hidden_dim=128)
model.load_state_dict(torch.load('sentiment_model.pth'))
model.eval()

# Create dummy input (batch_size=1, seq_length=100)
dummy_input = torch.randint(0, 10000, (1, 100), dtype=torch.long)

# Export to ONNX
torch.onnx.export(
    model,
    dummy_input,
    'sentiment_model.onnx',
    export_params=True,
    opset_version=12,
    do_constant_folding=True,
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={
        'input': {0: 'batch_size'},
        'output': {0: 'batch_size'}
    }
)

print("ONNX model exported successfully!")

# Verify ONNX model
import onnx
onnx_model = onnx.load('sentiment_model.onnx')
onnx.checker.check_model(onnx_model)
print("ONNX model is valid!")
```

### Step 5: Convert ONNX to TensorFlow
```python
# onnx_to_tensorflow.py
import onnx
from onnx_tf.backend import prepare
import tensorflow as tf

# Load ONNX model
onnx_model = onnx.load('sentiment_model.onnx')

# Convert to TensorFlow
tf_rep = prepare(onnx_model)

# Export as SavedModel
tf_rep.export_graph('sentiment_tf_model')

print("TensorFlow SavedModel created!")

# Verify TensorFlow model
loaded_model = tf.saved_model.load('sentiment_tf_model')
print("TensorFlow model loaded successfully!")
```

### Step 6: Convert TensorFlow to TensorFlow.js
```bash
# Convert SavedModel to TFJS Graph Model
tensorflowjs_converter \
    --input_format=tf_saved_model \
    --output_format=tfjs_graph_model \
    --signature_name=serving_default \
    --saved_model_tags=serve \
    ./sentiment_tf_model \
    ./tfjs_sentiment_model

echo "TensorFlow.js model created!"
ls -lh tfjs_sentiment_model/
```

### Step 7: Validate Conversion Pipeline
```python
# validate_conversion.py
import torch
import tensorflow as tf
import numpy as np
import json

# Test input
test_input_np = np.random.randint(0, 10000, (1, 100), dtype=np.int32)

# 1. PyTorch prediction
model_pytorch = SentimentLSTM(vocab_size=10000, embedding_dim=128, hidden_dim=128)
model_pytorch.load_state_dict(torch.load('sentiment_model.pth'))
model_pytorch.eval()

with torch.no_grad():
    test_input_torch = torch.from_numpy(test_input_np).long()
    pytorch_output = model_pytorch(test_input_torch).numpy()

print("PyTorch output:", pytorch_output)

# 2. TensorFlow prediction
tf_model = tf.saved_model.load('sentiment_tf_model')
tf_output = tf_model(**{'input': test_input_np}).numpy()

print("TensorFlow output:", tf_output)

# 3. Compare
difference = np.abs(pytorch_output - tf_output).max()
print(f"Maximum difference: {difference}")

if difference < 1e-3:
    print("✓ Conversion validated successfully!")
else:
    print("⚠ Significant difference detected!")
```

### Step 8: Create Browser Application
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sentiment Analysis - TensorFlow.js</title>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .container {
            background: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        textarea {
            width: 100%;
            height: 150px;
            padding: 10px;
            border: 2px solid #ddd;
            border-radius: 5px;
            font-size: 16px;
            resize: vertical;
        }
        button {
            background-color: #4CAF50;
            color: white;
            padding: 12px 30px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 16px;
            margin-top: 10px;
        }
        button:hover {
            background-color: #45a049;
        }
        button:disabled {
            background-color: #cccccc;
            cursor: not-allowed;
        }
        #result {
            margin-top: 20px;
            padding: 20px;
            border-radius: 5px;
            display: none;
        }
        .positive {
            background-color: #d4edda;
            border: 1px solid #c3e6cb;
        }
        .negative {
            background-color: #f8d7da;
            border: 1px solid #f5c6cb;
        }
        #loading {
            color: #666;
            font-style: italic;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Sentiment Analysis</h1>
        <p>Enter text to analyze sentiment (positive or negative)</p>

        <textarea id="textInput" placeholder="Type your review here... e.g., 'This movie was amazing!'"></textarea>

        <button id="analyzeBtn" onclick="analyzeSentiment()">Analyze Sentiment</button>

        <div id="loading" style="display:none;">Loading model...</div>

        <div id="result"></div>
    </div>

    <script src="sentiment.js"></script>
</body>
</html>
```

### Step 9: Implement JavaScript Inference
```javascript
// sentiment.js
let model = null;
let tokenizer = null;
const MAX_LENGTH = 100;

// Load tokenizer configuration
async function loadTokenizer() {
    const response = await fetch('tokenizer_config.json');
    const config = await response.json();
    tokenizer = config;
    console.log('Tokenizer loaded');
}

// Simple tokenization (matching Python version)
function tokenize(text) {
    text = text.toLowerCase();
    text = text.replace(/[^a-z0-9\s]/g, '');
    return text.split(/\s+/).filter(word => word.length > 0);
}

// Encode text to indices
function encodeText(text) {
    const tokens = tokenize(text);
    const indices = tokens.map(token =>
        tokenizer.word2idx[token] || tokenizer.word2idx['<UNK>']
    );

    // Pad or truncate
    const padded = new Array(MAX_LENGTH).fill(0);
    for (let i = 0; i < Math.min(indices.length, MAX_LENGTH); i++) {
        padded[i] = indices[i];
    }

    return padded;
}

// Load TensorFlow.js model
async function loadModel() {
    const loadingDiv = document.getElementById('loading');
    loadingDiv.style.display = 'block';

    try {
        await loadTokenizer();
        model = await tf.loadGraphModel('./tfjs_sentiment_model/model.json');
        console.log('Model loaded successfully');

        // Warm up
        tf.tidy(() => {
            const warmup = tf.zeros([1, MAX_LENGTH], 'int32');
            model.predict(warmup);
        });

        loadingDiv.textContent = 'Model ready!';
        setTimeout(() => loadingDiv.style.display = 'none', 1500);
        document.getElementById('analyzeBtn').disabled = false;
    } catch (error) {
        console.error('Error loading model:', error);
        loadingDiv.textContent = 'Error loading model: ' + error.message;
    }
}

// Analyze sentiment
async function analyzeSentiment() {
    const textInput = document.getElementById('textInput').value.trim();

    if (!textInput) {
        alert('Please enter some text to analyze');
        return;
    }

    if (!model) {
        alert('Model not loaded yet!');
        return;
    }

    const resultDiv = document.getElementById('result');
    resultDiv.style.display = 'none';

    try {
        // Encode text
        const encoded = encodeText(textInput);

        // Create tensor
        const inputTensor = tf.tensor2d([encoded], [1, MAX_LENGTH], 'int32');

        // Predict
        const prediction = model.predict(inputTensor);
        const probabilities = await tf.softmax(prediction).data();

        // Clean up
        inputTensor.dispose();
        prediction.dispose();

        // Display result
        const positiveProb = probabilities[1];
        const negativeProb = probabilities[0];

        const sentiment = positiveProb > negativeProb ? 'Positive' : 'Negative';
        const confidence = Math.max(positiveProb, negativeProb) * 100;

        resultDiv.innerHTML = `
            <h2>Result: ${sentiment}</h2>
            <p>Confidence: ${confidence.toFixed(2)}%</p>
            <p>Positive: ${(positiveProb * 100).toFixed(2)}%</p>
            <p>Negative: ${(negativeProb * 100).toFixed(2)}%</p>
        `;
        resultDiv.className = sentiment.toLowerCase();
        resultDiv.style.display = 'block';

    } catch (error) {
        console.error('Prediction error:', error);
        alert('Error during prediction: ' + error.message);
    }
}

// Initialize
document.getElementById('analyzeBtn').disabled = true;
loadModel();
```

## Expected Outputs

1. **Conversion Pipeline Artifacts**:
   - `sentiment_model.pth` (PyTorch weights)
   - `sentiment_model.onnx` (ONNX model)
   - `sentiment_tf_model/` (TensorFlow SavedModel)
   - `tfjs_sentiment_model/` (TFJS Graph Model)
   - `tokenizer_config.json` (vocabulary)

2. **Model Validation**:
   - Predictions match across all formats (difference < 0.001)
   - No conversion errors or warnings

3. **Web Application**:
   - Model loads successfully in browser
   - Text input processed correctly
   - Sentiment predictions displayed with confidence
   - Response time <1 second per prediction

4. **Console Output**:
   ```
   Tokenizer loaded
   Model loaded successfully
   Prediction: [positive_prob, negative_prob]
   ```

## Bonus Challenges

- [ ] Add support for neutral sentiment (3-class classification)
- [ ] Implement attention visualization showing important words
- [ ] Add multi-language support with different tokenizers
- [ ] Create a chrome extension for sentiment analysis
- [ ] Implement model quantization to reduce size
- [ ] Add autocomplete suggestions based on positive/negative words
- [ ] Build a comparison tool for multiple models
- [ ] Implement incremental text analysis (analyze while typing)
- [ ] Add export functionality for batch predictions
- [ ] Create API endpoint wrapper for the browser model

## Resources

- [PyTorch ONNX Export](https://pytorch.org/docs/stable/onnx.html)
- [ONNX-TensorFlow Converter](https://github.com/onnx/onnx-tensorflow)
- [TensorFlow.js Converter](https://www.tensorflow.org/js/guide/conversion)
- [ONNX Model Zoo](https://github.com/onnx/models)
- [IMDB Dataset](https://ai.stanford.edu/~amaas/data/sentiment/)
- [TensorFlow.js Graph Model API](https://js.tensorflow.org/api/latest/#loadGraphModel)
- [PyTorch Text Classification Tutorial](https://pytorch.org/tutorials/beginner/text_sentiment_ngrams_tutorial.html)

## Success Criteria

- PyTorch model trains with >80% validation accuracy
- ONNX export completes without errors
- ONNX model validates successfully
- TensorFlow conversion produces valid SavedModel
- TFJS conversion creates model.json and weight files
- Predictions match across all formats (max difference <0.001)
- Browser application loads and runs without errors
- Tokenization in JavaScript matches Python implementation
- Sentiment predictions are accurate and fast (<1 second)
- Memory usage is stable (no tensor leaks)
- Works across different browsers (Chrome, Firefox, Safari)
