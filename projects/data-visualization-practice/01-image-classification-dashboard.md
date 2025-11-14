# Project 1: Image Classification Visualization Dashboard

## Overview
Build an interactive dashboard that visualizes image classification results in real-time, showing predictions, confidence scores, and model explanations.

## Difficulty Level
Intermediate

## Learning Objectives
- Load and display images in a web interface
- Visualize classification probabilities
- Create confusion matrices and accuracy metrics
- Implement interactive filters and controls
- Display model attention/saliency maps

## Technical Stack
- **Backend**: Python, FastAPI or Flask
- **ML Framework**: TensorFlow/Keras or PyTorch
- **Visualization**: Plotly or Matplotlib
- **Dashboard**: Streamlit or Dash
- **Dataset**: CIFAR-10, Fashion-MNIST, or custom image dataset

## Project Requirements

### 1. Model Integration
- Load a pre-trained image classifier (ResNet, VGG, MobileNet, etc.)
- Or train a simple CNN on your chosen dataset
- Implement prediction endpoint

### 2. Core Visualizations
- **Image Display**: Show uploaded/selected image
- **Prediction Bar Chart**: Top-5 predictions with confidence scores
- **Confusion Matrix**: Overall model performance on test set
- **Accuracy Metrics**: Precision, recall, F1-score by class
- **Sample Gallery**: Grid of correctly/incorrectly classified images

### 3. Advanced Features
- **Saliency Maps**: Highlight which parts of image influenced prediction
- **Filter by Class**: Show only specific classes
- **Confidence Threshold**: Filter predictions by confidence level
- **Batch Processing**: Upload multiple images at once
- **Export Results**: Download predictions as CSV/JSON

### 4. UI Components
- File uploader for images
- Class selector dropdown
- Confidence threshold slider
- Tabs for different visualization views
- Real-time prediction updates

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit torch torchvision pillow plotly pandas numpy
```

### Step 2: Load Model
```python
import torch
import torchvision.models as models
import torchvision.transforms as transforms

model = models.resnet18(pretrained=True)
model.eval()
```

### Step 3: Create Prediction Function
```python
def predict_image(image):
    transform = transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406],
                           std=[0.229, 0.224, 0.225])
    ])

    img_tensor = transform(image).unsqueeze(0)

    with torch.no_grad():
        outputs = model(img_tensor)
        probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

    return probabilities.numpy()
```

### Step 4: Build Dashboard
- Create main Streamlit app
- Add image uploader
- Display predictions
- Add visualization charts

### Step 5: Add Advanced Visualizations
- Implement confusion matrix on test set
- Add saliency map generation
- Create performance metrics display

## Expected Outputs

1. **Interactive Dashboard** with:
   - Image upload functionality
   - Real-time prediction display
   - Multiple visualization views

2. **Visualizations**:
   - Bar chart of top predictions
   - Confusion matrix heatmap
   - Saliency/attention map overlay
   - Class-wise accuracy metrics

3. **User Controls**:
   - Class filter
   - Confidence threshold
   - Model selector (if multiple models)

## Bonus Challenges

- [ ] Add model comparison (compare 2+ models side-by-side)
- [ ] Implement grad-CAM for better interpretability
- [ ] Add data augmentation visualization
- [ ] Show misclassified examples for analysis
- [ ] Deploy to Streamlit Cloud or Heroku
- [ ] Add webcam integration for real-time classification
- [ ] Create downloadable PDF report of results

## Resources

- [Streamlit Documentation](https://docs.streamlit.io)
- [Plotly Python Graphing Library](https://plotly.com/python/)
- [PyTorch Vision Models](https://pytorch.org/vision/stable/models.html)
- [Grad-CAM Tutorial](https://jacobgil.github.io/pytorch-gradcam-book/)

## Success Criteria

- Dashboard loads and runs without errors
- Images can be uploaded and classified
- At least 3 different visualizations working
- UI is intuitive and responsive
- Predictions are accurate and fast (< 2 seconds)
