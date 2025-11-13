# TensorFlow.js Skill

This skill helps you work with TensorFlow.js for in-browser and Node.js machine learning.

## Capabilities

- Load pre-trained models in the browser
- Build and train models using TensorFlow.js APIs
- Optimize models for browser deployment
- Handle tensors and memory management
- Use WebGL and WASM backends
- Implement real-time inference in web apps
- Convert models from other frameworks

## When to Use This Skill

- Building ML features directly in the browser
- Creating privacy-preserving ML apps (data stays on device)
- Implementing real-time predictions without backend
- Converting Python models to run in JavaScript
- Building interactive ML demos and prototypes
- Edge deployment for offline-first applications

## Common Tasks

### Loading a Model
```javascript
// Load a pre-trained model
const model = await tf.loadLayersModel('/path/to/model.json');
// or
const model = await tf.loadGraphModel('/path/to/model.json');
```

### Making Predictions
```javascript
// Create input tensor
const input = tf.tensor2d([[1, 2, 3, 4]]);

// Make prediction
const prediction = model.predict(input);

// Get values
const values = await prediction.array();

// Clean up
input.dispose();
prediction.dispose();
```

### Building a Simple Model
```javascript
const model = tf.sequential({
  layers: [
    tf.layers.dense({ inputShape: [4], units: 16, activation: 'relu' }),
    tf.layers.dense({ units: 8, activation: 'relu' }),
    tf.layers.dense({ units: 1, activation: 'sigmoid' })
  ]
});

model.compile({
  optimizer: 'adam',
  loss: 'binaryCrossentropy',
  metrics: ['accuracy']
});
```

### Training a Model
```javascript
const xs = tf.tensor2d([[...]]);
const ys = tf.tensor2d([[...]]);

await model.fit(xs, ys, {
  epochs: 10,
  batchSize: 32,
  validationSplit: 0.2,
  callbacks: {
    onEpochEnd: (epoch, logs) => {
      console.log(`Epoch ${epoch}: loss = ${logs.loss}`);
    }
  }
});
```

## Memory Management

Always dispose of tensors to prevent memory leaks:

```javascript
// Manual disposal
const tensor = tf.tensor([1, 2, 3]);
tensor.dispose();

// Using tidy (automatic cleanup)
const result = tf.tidy(() => {
  const a = tf.tensor([1, 2, 3]);
  const b = tf.tensor([4, 5, 6]);
  return a.add(b);
});
// Only 'result' is kept in memory, a and b are disposed
```

## Performance Tips

- Use WebGL backend for GPU acceleration (default in browsers)
- Use WASM backend for CPU-only devices
- Batch predictions when possible
- Warm up the model with a dummy prediction
- Use quantized models for faster loading and inference
- Monitor memory usage with `tf.memory()`

## Best Practices

- Always call `dispose()` on tensors or use `tf.tidy()`
- Use `async/await` for model loading and predictions
- Implement error handling for model loading failures
- Test across different browsers and devices
- Consider model size and download time
- Provide loading indicators for model downloads
- Cache models in browser storage (IndexedDB)

## Troubleshooting

- **High memory usage**: Check for undisposed tensors
- **Slow inference**: Try WebGL backend, reduce input size
- **Model won't load**: Check CORS settings, file paths
- **Different results than Python**: Verify model conversion, preprocessing steps
