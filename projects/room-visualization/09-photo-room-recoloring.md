# Project 09: Photo-Based Room Recoloring with Computer Vision

## Overview
Build an intelligent photo recoloring system that uses computer vision and machine learning to automatically detect walls, furniture, and objects in room photos, then applies realistic color transformations while preserving textures, lighting, and shadows. Perfect for visualizing paint colors before committing to a purchase.

## Learning Objectives
- Master image segmentation with deep learning
- Implement color space transformations (RGB, HSV, LAB)
- Preserve image details during recoloring
- Build edge-preserving filters
- Create realistic lighting preservation algorithms
- Develop batch processing pipelines

## Difficulty Level
**Advanced** - Requires computer vision knowledge, Python/ML experience

## Technical Stack
- **Backend**: Python 3.9+
- **Computer Vision**: OpenCV, scikit-image
- **Deep Learning**:
  - PyTorch or TensorFlow
  - Mask R-CNN for segmentation
  - DeepLab v3+ for semantic segmentation
- **Image Processing**: PIL, NumPy, SciPy
- **Frontend**: Streamlit or Gradio for UI
- **Deployment**: Docker, FastAPI

## Room Specifications
- **Input**: Any room photo (JPG, PNG)
- **Output**: Recolored photo with natural appearance
- **Color Palettes**: Feng Shui (Warm, Soft, Calm)
- **Preservation**: Textures, shadows, highlights maintained

## Requirements

### 1. Image Analysis
- [x] Automatic wall detection
- [x] Floor segmentation
- [x] Furniture identification
- [x] Edge detection and refinement
- [x] Lighting analysis

### 2. Color Transformation
- [x] Selective region recoloring
- [x] Texture preservation
- [x] Shadow and highlight retention
- [x] Natural color blending
- [x] Edge anti-aliasing

### 3. AI Segmentation
- [x] Semantic segmentation model
- [x] Instance segmentation for furniture
- [x] Refinement with grabcut/graphcut
- [x] Post-processing filters
- [x] Confidence scoring

### 4. Batch Processing
- [x] Multiple images at once
- [x] Batch color application
- [x] Automated quality checks
- [x] Progress tracking
- [x] Error handling

### 5. Export Features
- [x] High-resolution output
- [x] Before/after comparison
- [x] Color palette reports
- [x] Paint quantity estimates
- [x] Shareable results

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create project directory
mkdir photo-room-recoloring
cd photo-room-recoloring

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install torch torchvision
pip install opencv-python opencv-contrib-python
pip install scikit-image scikit-learn
pip install Pillow numpy scipy
pip install detectron2 -f https://dl.fbaipublicfiles.com/detectron2/wheels/cu118/torch2.0/index.html
pip install streamlit
pip install matplotlib
```

**File: `requirements.txt`**
```txt
torch==2.1.0
torchvision==0.16.0
opencv-python==4.8.1.78
opencv-contrib-python==4.8.1.78
scikit-image==0.22.0
scikit-learn==1.3.2
Pillow==10.1.0
numpy==1.26.2
scipy==1.11.4
streamlit==1.28.2
matplotlib==3.8.2
detectron2
```

### Step 2: Segmentation Module

**File: `segmentation/wall_detector.py`**
```python
import cv2
import numpy as np
from detectron2 import model_zoo
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg
from detectron2.utils.visualizer import Visualizer
from detectron2.data import MetadataCatalog
import torch

class WallDetector:
    """Detect walls and surfaces in room photos using Mask R-CNN"""

    def __init__(self, model_type='mask_rcnn'):
        self.model_type = model_type
        self.predictor = self._load_model()

    def _load_model(self):
        """Load pre-trained segmentation model"""
        cfg = get_cfg()

        # Load Mask R-CNN config
        cfg.merge_from_file(
            model_zoo.get_config_file(
                "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"
            )
        )

        cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5
        cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
            "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"
        )

        # Use GPU if available
        cfg.MODEL.DEVICE = 'cuda' if torch.cuda.is_available() else 'cpu'

        predictor = DefaultPredictor(cfg)
        print(f"Model loaded on {cfg.MODEL.DEVICE}")

        return predictor

    def detect_surfaces(self, image):
        """
        Detect walls and floor in image

        Args:
            image: numpy array (BGR format)

        Returns:
            dict with masks for walls, floor, furniture
        """
        # Run inference
        outputs = self.predictor(image)

        # Get predictions
        instances = outputs["instances"].to("cpu")
        masks = instances.pred_masks.numpy()
        classes = instances.pred_classes.numpy()
        scores = instances.scores.numpy()

        # COCO class IDs: wall=73, floor=58, furniture=various
        wall_mask = self._combine_class_masks(masks, classes, [73], scores)
        floor_mask = self._combine_class_masks(masks, classes, [58], scores)
        furniture_masks = self._get_furniture_masks(masks, classes, scores)

        # Refine masks with morphological operations
        wall_mask = self._refine_mask(wall_mask)
        floor_mask = self._refine_mask(floor_mask)

        return {
            'wall': wall_mask,
            'floor': floor_mask,
            'furniture': furniture_masks,
            'instances': instances
        }

    def _combine_class_masks(self, masks, classes, target_classes, scores, min_score=0.5):
        """Combine masks from specific classes"""
        combined = np.zeros(masks[0].shape, dtype=np.uint8)

        for mask, cls, score in zip(masks, classes, scores):
            if cls in target_classes and score >= min_score:
                combined = np.logical_or(combined, mask).astype(np.uint8)

        return combined * 255

    def _get_furniture_masks(self, masks, classes, scores, min_score=0.5):
        """Get individual furniture masks"""
        # COCO furniture classes
        furniture_classes = [56, 57, 58, 59, 60, 61, 62]  # chair, couch, bed, etc.

        furniture_masks = []
        for mask, cls, score in zip(masks, classes, scores):
            if cls in furniture_classes and score >= min_score:
                furniture_masks.append(mask.astype(np.uint8) * 255)

        return furniture_masks

    def _refine_mask(self, mask):
        """Refine mask with morphological operations"""
        # Remove noise
        kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
        mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel)
        mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, kernel)

        # Smooth edges
        mask = cv2.GaussianBlur(mask, (5, 5), 0)

        return mask

    def detect_wall_simple(self, image):
        """
        Simplified wall detection using color and edge detection
        Fallback when ML model isn't available
        """
        # Convert to HSV
        hsv = cv2.cvtColor(image, cv2.COLOR_BGR2HSV)

        # Detect large flat areas (likely walls)
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        edges = cv2.Canny(gray, 50, 150)

        # Find contours
        contours, _ = cv2.findContours(
            edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )

        # Create mask for large contours (walls)
        wall_mask = np.zeros(gray.shape, dtype=np.uint8)

        for contour in contours:
            area = cv2.contourArea(contour)
            if area > 1000:  # Adjust threshold
                cv2.drawContours(wall_mask, [contour], -1, 255, -1)

        return wall_mask

    def refine_with_grabcut(self, image, mask):
        """Refine segmentation using GrabCut algorithm"""
        # Initialize GrabCut
        bgd_model = np.zeros((1, 65), np.float64)
        fgd_model = np.zeros((1, 65), np.float64)

        # Create trimap from mask
        trimap = np.where(mask > 128, cv2.GC_PR_FGD, cv2.GC_PR_BGD).astype(np.uint8)

        # Run GrabCut
        cv2.grabCut(
            image, trimap, None,
            bgd_model, fgd_model,
            5, cv2.GC_INIT_WITH_MASK
        )

        # Extract foreground
        refined_mask = np.where(
            (trimap == cv2.GC_FGD) | (trimap == cv2.GC_PR_FGD),
            255, 0
        ).astype(np.uint8)

        return refined_mask
```

### Step 3: Color Transformation Module

**File: `recoloring/color_transformer.py`**
```python
import cv2
import numpy as np
from sklearn.cluster import KMeans

class ColorTransformer:
    """Transform colors while preserving texture and lighting"""

    def __init__(self):
        self.palettes = {
            'warm': {
                'wall': (212, 115, 94),    # RGB: #D4735E
                'floor': (232, 213, 196),  # RGB: #E8D5C4
                'accent': (74, 63, 53)     # RGB: #4A3F35
            },
            'soft': {
                'wall': (184, 197, 214),   # RGB: #B8C5D6
                'floor': (245, 243, 239),  # RGB: #F5F3EF
                'accent': (60, 63, 65)     # RGB: #3C3F41
            },
            'calm': {
                'wall': (168, 181, 160),   # RGB: #A8B5A0
                'floor': (242, 235, 217),  # RGB: #F2EBD9
                'accent': (194, 197, 192)  # RGB: #C2C5C0
            }
        }

    def recolor_region(
        self,
        image,
        mask,
        target_color,
        preserve_lighting=True,
        preserve_texture=True
    ):
        """
        Recolor a specific region while preserving details

        Args:
            image: numpy array (BGR)
            mask: binary mask (0 or 255)
            target_color: tuple (B, G, R)
            preserve_lighting: keep shadows and highlights
            preserve_texture: maintain texture details

        Returns:
            recolored image
        """
        result = image.copy()

        # Convert to float for processing
        result = result.astype(np.float32) / 255.0
        mask_float = (mask > 128).astype(np.float32)

        if preserve_lighting:
            result = self._recolor_with_lighting(
                result, mask_float, target_color
            )
        else:
            result = self._simple_recolor(result, mask_float, target_color)

        if preserve_texture:
            result = self._preserve_texture(image, result, mask_float)

        # Convert back to uint8
        result = np.clip(result * 255, 0, 255).astype(np.uint8)

        return result

    def _recolor_with_lighting(self, image, mask, target_color):
        """Recolor while preserving lighting information"""
        # Convert to LAB color space for better color control
        lab = cv2.cvtColor((image * 255).astype(np.uint8), cv2.COLOR_BGR2LAB)
        lab = lab.astype(np.float32) / 255.0

        # Get target color in LAB
        target_bgr = np.array([[target_color]], dtype=np.uint8)
        target_lab = cv2.cvtColor(target_bgr, cv2.COLOR_BGR2LAB)
        target_lab = target_lab.astype(np.float32) / 255.0

        # Extract luminance (L channel)
        luminance = lab[:, :, 0]

        # Set new A and B channels
        new_lab = lab.copy()
        new_lab[:, :, 1] = target_lab[0, 0, 1]  # A channel
        new_lab[:, :, 2] = target_lab[0, 0, 2]  # B channel

        # Apply mask
        for c in range(3):
            new_lab[:, :, c] = (
                mask * new_lab[:, :, c] +
                (1 - mask) * lab[:, :, c]
            )

        # Convert back to BGR
        new_lab = (new_lab * 255).astype(np.uint8)
        result = cv2.cvtColor(new_lab, cv2.COLOR_LAB2BGR)
        result = result.astype(np.float32) / 255.0

        return result

    def _simple_recolor(self, image, mask, target_color):
        """Simple color replacement"""
        target = np.array(target_color, dtype=np.float32) / 255.0

        result = image.copy()
        for c in range(3):
            result[:, :, c] = (
                mask * target[c] +
                (1 - mask) * image[:, :, c]
            )

        return result

    def _preserve_texture(self, original, recolored, mask):
        """Preserve texture details from original image"""
        # Extract high-frequency details
        original_gray = cv2.cvtColor(
            (original * 255).astype(np.uint8) if original.dtype == np.float32 else original,
            cv2.COLOR_BGR2GRAY
        ).astype(np.float32) / 255.0

        recolored_gray = cv2.cvtColor(
            (recolored * 255).astype(np.uint8),
            cv2.COLOR_BGR2GRAY
        ).astype(np.float32) / 255.0

        # High-pass filter for texture
        original_blur = cv2.GaussianBlur(original_gray, (5, 5), 0)
        texture = original_gray - original_blur

        # Apply texture to recolored image
        result = recolored.copy()
        for c in range(3):
            result[:, :, c] = result[:, :, c] + texture * 0.3 * mask

        return result

    def detect_dominant_color(self, image, mask, n_colors=3):
        """Detect dominant colors in masked region"""
        # Extract pixels in mask
        pixels = image[mask > 128]

        if len(pixels) == 0:
            return None

        # Cluster colors
        kmeans = KMeans(n_clusters=n_colors, random_state=42, n_init=10)
        kmeans.fit(pixels)

        # Get dominant color (cluster center)
        dominant_color = kmeans.cluster_centers_[0].astype(int)

        return tuple(dominant_color)

    def blend_edges(self, image, mask, blend_radius=5):
        """Smooth edges between recolored and original regions"""
        # Create soft mask
        soft_mask = cv2.GaussianBlur(
            mask.astype(np.float32),
            (blend_radius * 2 + 1, blend_radius * 2 + 1),
            0
        ) / 255.0

        return soft_mask
```

### Step 4: Streamlit Web Interface

**File: `app.py`**
```python
import streamlit as st
import cv2
import numpy as np
from PIL import Image
from segmentation.wall_detector import WallDetector
from recoloring.color_transformer import ColorTransformer
import io

# Page config
st.set_page_config(
    page_title="Room Recoloring Tool",
    page_icon="🎨",
    layout="wide"
)

# Initialize models
@st.cache_resource
def load_models():
    detector = WallDetector()
    transformer = ColorTransformer()
    return detector, transformer

detector, transformer = load_models()

# Title
st.title("🎨 Photo-Based Room Recoloring")
st.markdown("Upload a room photo and visualize different color palettes")

# Sidebar controls
st.sidebar.header("Settings")

palette_name = st.sidebar.selectbox(
    "Color Palette",
    ["warm", "soft", "calm"],
    format_func=lambda x: {
        "warm": "Warm Focus",
        "soft": "Soft Tech",
        "calm": "Calm Studio"
    }[x]
)

preserve_lighting = st.sidebar.checkbox("Preserve Lighting", value=True)
preserve_texture = st.sidebar.checkbox("Preserve Texture", value=True)
edge_blend = st.sidebar.slider("Edge Blending", 1, 20, 5)

# File upload
uploaded_file = st.file_uploader(
    "Upload room photo",
    type=['jpg', 'jpeg', 'png']
)

if uploaded_file is not None:
    # Read image
    file_bytes = np.asarray(bytearray(uploaded_file.read()), dtype=np.uint8)
    image = cv2.imdecode(file_bytes, cv2.IMREAD_COLOR)

    # Resize if too large
    max_size = 1024
    h, w = image.shape[:2]
    if max(h, w) > max_size:
        scale = max_size / max(h, w)
        new_w = int(w * scale)
        new_h = int(h * scale)
        image = cv2.resize(image, (new_w, new_h))

    col1, col2 = st.columns(2)

    with col1:
        st.subheader("Original")
        st.image(cv2.cvtColor(image, cv2.COLOR_BGR2RGB))

    # Process button
    if st.sidebar.button("🎨 Apply Color Palette", type="primary"):
        with st.spinner("Processing..."):
            # Detect walls
            surfaces = detector.detect_surfaces(image)
            wall_mask = surfaces['wall']

            # Get target color
            palette = transformer.palettes[palette_name]
            target_color = palette['wall']

            # Recolor
            result = transformer.recolor_region(
                image,
                wall_mask,
                target_color,
                preserve_lighting=preserve_lighting,
                preserve_texture=preserve_texture
            )

            # Display result
            with col2:
                st.subheader("Recolored")
                st.image(cv2.cvtColor(result, cv2.COLOR_BGR2RGB))

            # Download button
            result_rgb = cv2.cvtColor(result, cv2.COLOR_BGR2RGB)
            result_pil = Image.fromarray(result_rgb)

            buf = io.BytesIO()
            result_pil.save(buf, format='PNG')
            byte_im = buf.getvalue()

            st.sidebar.download_button(
                label="💾 Download Result",
                data=byte_im,
                file_name=f"recolored_{palette_name}.png",
                mime="image/png"
            )

            # Show color palette
            st.sidebar.markdown("### Applied Colors")
            st.sidebar.color_picker(
                "Wall Color",
                f"#{target_color[2]:02x}{target_color[1]:02x}{target_color[0]:02x}",
                disabled=True
            )

            # Paint calculator
            st.sidebar.markdown("### Paint Estimate")
            wall_area = np.sum(wall_mask > 128) / (image.shape[0] * image.shape[1]) * 7.8
            paint_liters = max(1, int(np.ceil(wall_area * 2 / 10)))

            st.sidebar.metric("Estimated Paint", f"{paint_liters}L")
            st.sidebar.caption("Based on 2 coats, 1L per 10m²")

else:
    st.info("👆 Upload a room photo to get started")

    # Show example
    st.markdown("### Example Results")
    col1, col2, col3 = st.columns(3)

    with col1:
        st.markdown("**Original**")
        st.image("examples/room-before.jpg", use_column_width=True)

    with col2:
        st.markdown("**Warm Focus**")
        st.image("examples/room-warm.jpg", use_column_width=True)

    with col3:
        st.markdown("**Soft Tech**")
        st.image("examples/room-soft.jpg", use_column_width=True)
```

### Step 5: Run the Application

```bash
# Start Streamlit app
streamlit run app.py

# Or use FastAPI for production
uvicorn api:app --reload
```

## Expected Outputs

1. **Accurate Segmentation**: 85%+ wall detection accuracy
2. **Natural Recoloring**: Preserves texture and lighting
3. **Multiple Palettes**: 3 Feng Shui color schemes
4. **High Quality**: No artifacts or color bleeding
5. **Fast Processing**: <5 seconds per image
6. **Before/After View**: Side-by-side comparison

## Bonus Challenges

1. **Custom Colors**: User-defined color picker
2. **Multi-Room**: Process entire house photo sets
3. **Style Transfer**: Apply artistic painting styles
4. **Material Change**: Change wall materials (brick, wood)
5. **Furniture Recolor**: Change furniture colors too
6. **AR Preview**: Export to AR-compatible format
7. **API Service**: Deploy as REST API

## Resources

- [Detectron2 Documentation](https://detectron2.readthedocs.io/)
- [OpenCV Color Spaces](https://docs.opencv.org/master/df/d9d/tutorial_py_colorspaces.html)
- [GrabCut Algorithm](https://docs.opencv.org/master/d8/d83/tutorial_py_grabcut.html)
- [Color Theory](https://programmingdesignsystems.com/color/)

## Success Criteria

- [ ] Wall detection accuracy >80%
- [ ] Natural color application
- [ ] Texture preservation visible
- [ ] Shadows and highlights maintained
- [ ] No color bleeding at edges
- [ ] Processing time <5 seconds
- [ ] Works with various room types
- [ ] Batch processing functional
