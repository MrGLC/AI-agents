You are helping convert a machine learning model to a different format.

Follow these steps:

1. **Identify source and target**:
   - Ask: What is the current model format? (PyTorch, TensorFlow/Keras, scikit-learn, ONNX, etc.)
   - Ask: What is the target format? (TensorFlow.js, ONNX, TFLite, CoreML, etc.)
   - Verify the model file exists and location

2. **Determine conversion path**:
   Common conversions:
   - **PyTorch → TensorFlow.js**: PyTorch → ONNX → TensorFlow → TensorFlow.js
   - **Keras → TensorFlow.js**: Direct conversion with `tensorflowjs_converter`
   - **TensorFlow → ONNX**: Using `tf2onnx`
   - **PyTorch → ONNX**: Using `torch.onnx.export`
   - **Scikit-learn → ONNX**: Using `skl2onnx`
   - **TensorFlow → TFLite**: For mobile deployment

3. **Check for optimization needs**:
   - Ask: Should the model be optimized? (quantization, pruning)
   - Quantization options: float16 (~50% reduction), uint8 (~75% reduction)
   - Trade-off: Size vs accuracy

4. **Perform conversion**:
   - Install required conversion tools
   - Run conversion with appropriate settings
   - Apply optimizations if requested
   - Handle any conversion errors or unsupported operations

5. **Validate conversion**:
   - Load both original and converted models
   - Run same input through both
   - Compare outputs (should be very similar)
   - Document any differences

6. **Provide usage instructions**:
   - How to load the converted model
   - Example inference code
   - Any preprocessing requirements
   - Platform-specific considerations

7. **Document the process**:
   - Conversion commands used
   - Any modifications needed
   - Performance comparisons (size, speed)
   - Accuracy comparison results

Ask about the source model and target format to begin.
