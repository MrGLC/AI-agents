# Model Conversion Skill

This skill helps you convert machine learning models between different frameworks and formats.

## Supported Conversions

- **PyTorch** → ONNX → TensorFlow → TensorFlow.js
- **TensorFlow/Keras** → TensorFlow.js
- **TensorFlow/Keras** → ONNX
- **Scikit-learn** → ONNX
- **ONNX** → TensorFlow
- **SavedModel** → TFLite (mobile)
- **Any** → ONNX (universal interchange format)

## When to Use This Skill

- Deploying models to different platforms (web, mobile, embedded)
- Migrating between ML frameworks
- Optimizing models for specific hardware
- Creating cross-platform ML applications
- Reducing model size for edge deployment
- Ensuring model compatibility with production systems

## Conversion Workflows

### PyTorch → TensorFlow.js

```bash
# Step 1: PyTorch to ONNX
import torch
model = torch.load('model.pth')
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(model, dummy_input, 'model.onnx')

# Step 2: ONNX to TensorFlow
onnx-tf convert -i model.onnx -o tf_model

# Step 3: TensorFlow to TensorFlow.js
tensorflowjs_converter \
  --input_format=tf_saved_model \
  --output_format=tfjs_graph_model \
  tf_model \
  tfjs_model
```

### Keras → TensorFlow.js

```bash
# Direct conversion
tensorflowjs_converter \
  --input_format=keras \
  model.h5 \
  tfjs_model/

# With quantization
tensorflowjs_converter \
  --input_format=keras \
  --quantize_float16 \
  model.h5 \
  tfjs_model/
```

### TensorFlow → ONNX

```bash
python -m tf2onnx.convert \
  --saved-model /path/to/saved_model \
  --output model.onnx \
  --opset 13
```

### Scikit-learn → ONNX

```python
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType

initial_type = [('float_input', FloatTensorType([None, 4]))]
onx = convert_sklearn(sklearn_model, initial_types=initial_type)

with open("model.onnx", "wb") as f:
    f.write(onx.SerializeToString())
```

### ONNX → TensorFlow

```bash
onnx-tf convert -i model.onnx -o tf_model/
```

## Optimization Techniques

### Quantization
Reduce model size and improve inference speed:

```bash
# Float16 quantization (50% size reduction)
tensorflowjs_converter \
  --quantize_float16 \
  input_model \
  output_model

# Uint8 quantization (75% size reduction)
tensorflowjs_converter \
  --quantize_uint8 \
  input_model \
  output_model
```

### Pruning
Remove unnecessary weights before conversion:

```python
import tensorflow as tf
import tensorflow_model_optimization as tfmot

# Define pruning schedule
pruning_schedule = tfmot.sparsity.keras.PolynomialDecay(
    initial_sparsity=0.0,
    final_sparsity=0.5,
    begin_step=0,
    end_step=1000
)

# Apply pruning
model_for_pruning = tfmot.sparsity.keras.prune_low_magnitude(
    model,
    pruning_schedule=pruning_schedule
)
```

## Validation

Always validate that converted models produce similar outputs:

```python
import numpy as np

# Create test input
test_input = np.random.randn(1, 224, 224, 3).astype(np.float32)

# Get original model output
original_output = original_model.predict(test_input)

# Get converted model output
converted_output = converted_model.predict(test_input)

# Compare outputs
difference = np.abs(original_output - converted_output)
print(f"Max difference: {np.max(difference)}")
print(f"Mean difference: {np.mean(difference)}")
```

## Common Issues and Solutions

### Unsupported Operations
- Check framework compatibility matrices
- Replace unsupported ops with equivalents
- Use fallback implementations
- Consider exporting a subgraph

### Shape Mismatches
- Verify input shapes match between frameworks
- Check for differences in channel ordering (NCHW vs NHWC)
- Ensure batch dimensions are handled correctly

### Accuracy Degradation
- Test with representative data
- Check preprocessing steps are identical
- Verify quantization didn't hurt accuracy too much
- Compare layer-by-layer outputs

### Large Model Size
- Apply quantization
- Prune unnecessary weights
- Remove unused operations
- Split model into smaller pieces

## Best Practices

- Always test converted models thoroughly
- Document the conversion process and any modifications
- Keep the original model for reference
- Version converted models alongside originals
- Maintain consistent preprocessing across platforms
- Use virtual environments to manage conversion tool dependencies
- Automate conversion with scripts for reproducibility

## Tools Reference

- **tensorflowjs_converter**: TensorFlow/Keras → TensorFlow.js
- **onnx-tf**: ONNX ↔ TensorFlow
- **torch.onnx**: PyTorch → ONNX
- **tf2onnx**: TensorFlow → ONNX
- **skl2onnx**: Scikit-learn → ONNX
- **onnxruntime**: Universal ONNX inference
