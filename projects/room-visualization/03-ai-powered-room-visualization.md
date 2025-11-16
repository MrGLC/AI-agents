# Project 03: AI-Powered Room Visualization with Stable Diffusion

## Overview
Build an AI-powered room design generator that uses Stable Diffusion or DALL-E to create photorealistic room visualizations from text descriptions. Users input room dimensions, color preferences, furniture requirements, and style choices to generate multiple design variations.

## Learning Objectives
- Integrate AI image generation APIs (Stable Diffusion, DALL-E, Midjourney)
- Master prompt engineering for interior design
- Build image-to-image transformation pipelines
- Implement ControlNet for precise spatial control
- Create variation generation systems
- Develop image upscaling and refinement workflows

## Difficulty Level
**Advanced** - Requires API integration, prompt engineering, and ML model understanding

## Technical Stack
- **Backend**: Python 3.9+, FastAPI
- **AI Models**:
  - Stable Diffusion (via Stability AI API or local)
  - ControlNet (for layout control)
  - DALL-E 3 (via OpenAI API)
  - Img2Img for variations
- **Frontend**: React, Next.js
- **Image Processing**: PIL, OpenCV, NumPy
- **Infrastructure**: Docker, CUDA (for local GPU)
- **Storage**: AWS S3 or Cloudinary for generated images

## Room Specifications
- **Dimensions**: 2.83m × 2.75m × 2.50m
- **Color Palettes**: Feng Shui-based (Warm, Soft, Calm)
- **Furniture**: Desk, piano, bed, storage
- **Styles**: Modern, Minimalist, Scandinavian, Industrial, Bohemian

## Requirements

### 1. Text-to-Image Generation
- [x] Accept detailed room description prompts
- [x] Generate multiple variations (4-8 images)
- [x] Support different artistic styles
- [x] Incorporate specific color palettes
- [x] Include furniture placement in prompts

### 2. ControlNet Integration
- [x] Use depth maps for spatial control
- [x] Implement edge detection for layout
- [x] Support custom floor plans as conditioning
- [x] Maintain room proportions

### 3. Image Refinement
- [x] Upscale generated images (2x, 4x)
- [x] Apply style transfer
- [x] Adjust lighting and color grading
- [x] Remove artifacts with inpainting

### 4. Variation Generation
- [x] Generate variations from seed image
- [x] Modify colors while keeping layout
- [x] Swap furniture styles
- [x] Change time of day/lighting

### 5. Export & Sharing
- [x] Download high-resolution images
- [x] Save prompt templates
- [x] Generate design mood boards
- [x] Create comparison galleries

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create project directory
mkdir ai-room-visualizer
cd ai-room-visualizer

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn python-multipart
pip install openai stability-sdk
pip install diffusers transformers accelerate
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
pip install opencv-python pillow numpy
pip install python-dotenv pydantic
```

**File: `requirements.txt`**
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6
openai==1.3.0
stability-sdk==0.8.4
diffusers==0.24.0
transformers==4.35.0
accelerate==0.24.0
torch==2.1.0
torchvision==0.16.0
opencv-python==4.8.1.78
Pillow==10.1.0
numpy==1.26.2
python-dotenv==1.0.0
pydantic==2.5.0
controlnet-aux==0.0.7
```

### Step 2: Backend API with FastAPI

**File: `backend/main.py`**
```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional, Dict
import os
from dotenv import load_dotenv

from generators.stable_diffusion import StableDiffusionGenerator
from generators.dalle import DalleGenerator
from utils.prompt_builder import PromptBuilder
from utils.image_processor import ImageProcessor

load_dotenv()

app = FastAPI(title="AI Room Visualizer API")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize generators
sd_generator = StableDiffusionGenerator()
dalle_generator = DalleGenerator(api_key=os.getenv("OPENAI_API_KEY"))
prompt_builder = PromptBuilder()
image_processor = ImageProcessor()

class RoomDesignRequest(BaseModel):
    width: float = 2.83
    depth: float = 2.75
    height: float = 2.50
    color_palette: str  # "warm", "soft", "calm"
    furniture: List[str]  # ["desk", "piano", "bed", "storage"]
    style: str  # "modern", "minimalist", "scandinavian", etc.
    lighting: str = "natural"  # "natural", "warm", "cool", "dramatic"
    additional_details: Optional[str] = None
    num_variations: int = 4
    model: str = "stable-diffusion"  # or "dalle"

class ImageVariationRequest(BaseModel):
    image_url: str
    modifications: Dict[str, str]  # {"color": "blue", "lighting": "warm"}
    num_variations: int = 4

@app.get("/")
def read_root():
    return {
        "message": "AI Room Visualizer API",
        "version": "1.0.0",
        "available_models": ["stable-diffusion", "dalle"],
        "color_palettes": ["warm", "soft", "calm"],
        "styles": ["modern", "minimalist", "scandinavian", "industrial", "bohemian"]
    }

@app.post("/generate")
async def generate_room_design(request: RoomDesignRequest):
    """Generate room design from description"""
    try:
        # Build prompt
        prompt = prompt_builder.build_room_prompt(
            dimensions=(request.width, request.depth, request.height),
            color_palette=request.color_palette,
            furniture=request.furniture,
            style=request.style,
            lighting=request.lighting,
            additional=request.additional_details
        )

        print(f"Generated prompt: {prompt}")

        # Generate images based on selected model
        if request.model == "dalle":
            images = await dalle_generator.generate(
                prompt=prompt,
                num_images=request.num_variations,
                size="1024x1024"
            )
        else:  # stable-diffusion
            images = await sd_generator.generate(
                prompt=prompt,
                num_images=request.num_variations,
                width=1024,
                height=1024,
                guidance_scale=7.5
            )

        return {
            "status": "success",
            "prompt": prompt,
            "images": images,
            "count": len(images)
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/generate-variations")
async def generate_variations(request: ImageVariationRequest):
    """Generate variations from existing image"""
    try:
        # Download and process source image
        source_image = image_processor.download_image(request.image_url)

        # Generate variations
        variations = await sd_generator.generate_img2img_variations(
            source_image=source_image,
            modifications=request.modifications,
            num_variations=request.num_variations
        )

        return {
            "status": "success",
            "variations": variations,
            "count": len(variations)
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/upscale")
async def upscale_image(image_url: str, scale: int = 2):
    """Upscale image using AI"""
    try:
        # Download image
        image = image_processor.download_image(image_url)

        # Upscale
        upscaled = image_processor.upscale(image, scale=scale)

        # Save and return URL
        upscaled_url = image_processor.save_to_storage(upscaled)

        return {
            "status": "success",
            "original_url": image_url,
            "upscaled_url": upscaled_url,
            "scale": scale
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

### Step 3: Prompt Builder

**File: `backend/utils/prompt_builder.py`**
```python
from typing import List, Tuple, Optional

class PromptBuilder:
    """Build optimized prompts for room visualization"""

    # Color palette definitions
    COLOR_PALETTES = {
        "warm": {
            "primary": "terracotta",
            "secondary": "warm beige",
            "accent": "deep brown",
            "description": "warm, cozy, earth-toned color scheme"
        },
        "soft": {
            "primary": "soft blue-gray",
            "secondary": "warm white",
            "accent": "charcoal",
            "description": "soft, modern, minimalist color palette"
        },
        "calm": {
            "primary": "sage green",
            "secondary": "cream",
            "accent": "soft gray",
            "description": "calm, nature-inspired, serene colors"
        }
    }

    # Style keywords
    STYLE_KEYWORDS = {
        "modern": "modern, contemporary, clean lines, minimalist furniture",
        "minimalist": "minimalist, sparse, functional, simple geometric forms",
        "scandinavian": "scandinavian, hygge, natural wood, bright and airy",
        "industrial": "industrial, exposed brick, metal accents, urban loft",
        "bohemian": "bohemian, eclectic, colorful textiles, plants, layered textures"
    }

    # Lighting descriptions
    LIGHTING_STYLES = {
        "natural": "soft natural daylight streaming through window",
        "warm": "warm ambient lighting, golden hour glow",
        "cool": "cool white lighting, bright and energizing",
        "dramatic": "dramatic directional lighting with shadows"
    }

    def build_room_prompt(
        self,
        dimensions: Tuple[float, float, float],
        color_palette: str,
        furniture: List[str],
        style: str,
        lighting: str,
        additional: Optional[str] = None
    ) -> str:
        """Build comprehensive prompt for room generation"""

        width, depth, height = dimensions
        palette = self.COLOR_PALETTES.get(color_palette, self.COLOR_PALETTES["warm"])

        # Base prompt
        prompt_parts = [
            "A professional interior design photograph of a small bedroom studio",
            f"Room dimensions approximately {width}m by {depth}m",
        ]

        # Style
        if style in self.STYLE_KEYWORDS:
            prompt_parts.append(self.STYLE_KEYWORDS[style])

        # Color palette
        prompt_parts.append(f"Color scheme: {palette['description']}")
        prompt_parts.append(
            f"Walls in {palette['primary']}, "
            f"accents in {palette['secondary']}, "
            f"details in {palette['accent']}"
        )

        # Furniture
        furniture_desc = self._build_furniture_description(furniture)
        prompt_parts.append(furniture_desc)

        # Lighting
        if lighting in self.LIGHTING_STYLES:
            prompt_parts.append(self.LIGHTING_STYLES[lighting])

        # Additional details
        if additional:
            prompt_parts.append(additional)

        # Quality modifiers
        prompt_parts.append(
            "high quality, professional photography, "
            "architectural digest style, 8k resolution, "
            "realistic materials and textures"
        )

        # Negative prompt elements
        negative = (
            "blurry, distorted, cartoon, unrealistic proportions, "
            "oversaturated, cluttered, messy"
        )

        full_prompt = ", ".join(prompt_parts)

        return {
            "prompt": full_prompt,
            "negative_prompt": negative
        }

    def _build_furniture_description(self, furniture: List[str]) -> str:
        """Build furniture description from list"""

        furniture_map = {
            "desk": "wooden desk with modern design",
            "piano": "upright piano in dark wood finish",
            "bed": "comfortable single bed with minimalist frame",
            "storage": "floating shelves or storage unit",
            "chair": "ergonomic office chair"
        }

        descriptions = [
            furniture_map.get(item, item)
            for item in furniture
            if item in furniture_map
        ]

        if descriptions:
            return f"Furniture includes: {', '.join(descriptions)}"
        return "Sparsely furnished"

    def build_variation_prompt(
        self,
        base_prompt: str,
        modifications: dict
    ) -> str:
        """Modify prompt for variations"""

        variation_prompt = base_prompt

        # Apply modifications
        if "color" in modifications:
            variation_prompt += f", with {modifications['color']} color accents"

        if "lighting" in modifications:
            lighting = self.LIGHTING_STYLES.get(
                modifications['lighting'],
                modifications['lighting']
            )
            variation_prompt += f", {lighting}"

        if "style" in modifications:
            style = self.STYLE_KEYWORDS.get(
                modifications['style'],
                modifications['style']
            )
            variation_prompt += f", {style}"

        return variation_prompt
```

### Step 4: Stable Diffusion Generator

**File: `backend/generators/stable_diffusion.py`**
```python
import torch
from diffusers import (
    StableDiffusionPipeline,
    StableDiffusionImg2ImgPipeline,
    ControlNetModel,
    StableDiffusionControlNetPipeline,
    DPMSolverMultistepScheduler
)
from PIL import Image
import io
import base64
from typing import List, Optional, Dict
import numpy as np

class StableDiffusionGenerator:
    """Generate room designs using Stable Diffusion"""

    def __init__(
        self,
        model_id: str = "stabilityai/stable-diffusion-2-1",
        device: str = "cuda" if torch.cuda.is_available() else "cpu"
    ):
        self.device = device
        self.model_id = model_id

        # Load text-to-image pipeline
        self.txt2img_pipe = StableDiffusionPipeline.from_pretrained(
            model_id,
            torch_dtype=torch.float16 if device == "cuda" else torch.float32,
            safety_checker=None
        )
        self.txt2img_pipe.scheduler = DPMSolverMultistepScheduler.from_config(
            self.txt2img_pipe.scheduler.config
        )
        self.txt2img_pipe = self.txt2img_pipe.to(device)

        # Enable memory optimizations
        if device == "cuda":
            self.txt2img_pipe.enable_attention_slicing()
            self.txt2img_pipe.enable_vae_slicing()

        print(f"Stable Diffusion loaded on {device}")

    async def generate(
        self,
        prompt: Dict[str, str],
        num_images: int = 4,
        width: int = 1024,
        height: int = 1024,
        guidance_scale: float = 7.5,
        num_inference_steps: int = 50,
        seed: Optional[int] = None
    ) -> List[str]:
        """Generate images from prompt"""

        images = []

        for i in range(num_images):
            # Set seed for reproducibility
            generator = torch.Generator(device=self.device)
            if seed is not None:
                generator.manual_seed(seed + i)

            # Generate image
            result = self.txt2img_pipe(
                prompt=prompt["prompt"],
                negative_prompt=prompt.get("negative_prompt", ""),
                width=width,
                height=height,
                guidance_scale=guidance_scale,
                num_inference_steps=num_inference_steps,
                generator=generator
            )

            # Convert to base64
            image = result.images[0]
            image_base64 = self._image_to_base64(image)
            images.append(image_base64)

        return images

    async def generate_img2img_variations(
        self,
        source_image: Image.Image,
        modifications: Dict[str, str],
        num_variations: int = 4,
        strength: float = 0.7
    ) -> List[str]:
        """Generate variations from source image"""

        # Load img2img pipeline if not loaded
        if not hasattr(self, 'img2img_pipe'):
            self.img2img_pipe = StableDiffusionImg2ImgPipeline.from_pretrained(
                self.model_id,
                torch_dtype=torch.float16 if self.device == "cuda" else torch.float32,
                safety_checker=None
            )
            self.img2img_pipe = self.img2img_pipe.to(self.device)

        # Build modification prompt
        modification_prompt = self._build_modification_prompt(modifications)

        variations = []

        for i in range(num_variations):
            generator = torch.Generator(device=self.device).manual_seed(42 + i)

            result = self.img2img_pipe(
                prompt=modification_prompt,
                image=source_image,
                strength=strength,
                guidance_scale=7.5,
                generator=generator
            )

            variation = result.images[0]
            variation_base64 = self._image_to_base64(variation)
            variations.append(variation_base64)

        return variations

    def _build_modification_prompt(self, modifications: Dict[str, str]) -> str:
        """Build prompt from modifications"""
        parts = ["Interior design, room,"]

        if "color" in modifications:
            parts.append(f"{modifications['color']} color scheme,")
        if "lighting" in modifications:
            parts.append(f"{modifications['lighting']} lighting,")
        if "style" in modifications:
            parts.append(f"{modifications['style']} style,")

        parts.append("professional photography, high quality")

        return " ".join(parts)

    def _image_to_base64(self, image: Image.Image) -> str:
        """Convert PIL Image to base64 string"""
        buffered = io.BytesIO()
        image.save(buffered, format="PNG")
        img_str = base64.b64encode(buffered.getvalue()).decode()
        return f"data:image/png;base64,{img_str}"

    def cleanup(self):
        """Clean up GPU memory"""
        if hasattr(self, 'txt2img_pipe'):
            del self.txt2img_pipe
        if hasattr(self, 'img2img_pipe'):
            del self.img2img_pipe
        if self.device == "cuda":
            torch.cuda.empty_cache()
```

### Step 5: DALL-E Generator

**File: `backend/generators/dalle.py`**
```python
from openai import AsyncOpenAI
from typing import List, Optional
import httpx

class DalleGenerator:
    """Generate room designs using DALL-E 3"""

    def __init__(self, api_key: str):
        self.client = AsyncOpenAI(api_key=api_key)

    async def generate(
        self,
        prompt: str,
        num_images: int = 1,  # DALL-E 3 generates 1 at a time
        size: str = "1024x1024",
        quality: str = "standard"  # or "hd"
    ) -> List[str]:
        """Generate images using DALL-E 3"""

        images = []

        # DALL-E 3 API generates one image at a time
        for _ in range(num_images):
            response = await self.client.images.generate(
                model="dall-e-3",
                prompt=prompt,
                size=size,
                quality=quality,
                n=1
            )

            image_url = response.data[0].url
            images.append(image_url)

        return images

    async def generate_variation(
        self,
        image_url: str,
        num_variations: int = 1
    ) -> List[str]:
        """Generate variations (DALL-E 2 feature)"""

        # Note: DALL-E 3 doesn't support variations directly
        # This would use DALL-E 2 or img2img approach

        variations = []

        # Download image
        async with httpx.AsyncClient() as client:
            response = await client.get(image_url)
            image_bytes = response.content

        # Use DALL-E 2 for variations
        response = await self.client.images.create_variation(
            image=image_bytes,
            n=num_variations,
            size="1024x1024"
        )

        for data in response.data:
            variations.append(data.url)

        return variations
```

### Step 6: Frontend React App

**File: `frontend/src/App.jsx`**
```javascript
import React, { useState } from 'react';
import axios from 'axios';
import './App.css';

const API_URL = 'http://localhost:8000';

function App() {
  const [formData, setFormData] = useState({
    color_palette: 'warm',
    furniture: ['desk', 'piano', 'bed', 'storage'],
    style: 'modern',
    lighting: 'natural',
    additional_details: '',
    num_variations: 4,
    model: 'stable-diffusion'
  });

  const [loading, setLoading] = useState(false);
  const [images, setImages] = useState([]);
  const [error, setError] = useState(null);
  const [prompt, setPrompt] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError(null);
    setImages([]);

    try {
      const response = await axios.post(`${API_URL}/generate`, formData);
      setImages(response.data.images);
      setPrompt(response.data.prompt);
    } catch (err) {
      setError(err.response?.data?.detail || 'Generation failed');
    } finally {
      setLoading(false);
    }
  };

  const handleFurnitureToggle = (item) => {
    setFormData(prev => ({
      ...prev,
      furniture: prev.furniture.includes(item)
        ? prev.furniture.filter(f => f !== item)
        : [...prev.furniture, item]
    }));
  };

  return (
    <div className="App">
      <header>
        <h1>🎨 AI Room Visualizer</h1>
        <p>Generate photorealistic room designs with AI</p>
      </header>

      <div className="container">
        <aside className="sidebar">
          <form onSubmit={handleSubmit}>
            <section>
              <h3>Color Palette</h3>
              <select
                value={formData.color_palette}
                onChange={(e) => setFormData({...formData, color_palette: e.target.value})}
              >
                <option value="warm">Warm Focus</option>
                <option value="soft">Soft Tech</option>
                <option value="calm">Calm Studio</option>
              </select>
            </section>

            <section>
              <h3>Furniture</h3>
              {['desk', 'piano', 'bed', 'storage', 'chair'].map(item => (
                <label key={item}>
                  <input
                    type="checkbox"
                    checked={formData.furniture.includes(item)}
                    onChange={() => handleFurnitureToggle(item)}
                  />
                  {item.charAt(0).toUpperCase() + item.slice(1)}
                </label>
              ))}
            </section>

            <section>
              <h3>Style</h3>
              <select
                value={formData.style}
                onChange={(e) => setFormData({...formData, style: e.target.value})}
              >
                <option value="modern">Modern</option>
                <option value="minimalist">Minimalist</option>
                <option value="scandinavian">Scandinavian</option>
                <option value="industrial">Industrial</option>
                <option value="bohemian">Bohemian</option>
              </select>
            </section>

            <section>
              <h3>Lighting</h3>
              <select
                value={formData.lighting}
                onChange={(e) => setFormData({...formData, lighting: e.target.value})}
              >
                <option value="natural">Natural Daylight</option>
                <option value="warm">Warm & Cozy</option>
                <option value="cool">Cool & Bright</option>
                <option value="dramatic">Dramatic</option>
              </select>
            </section>

            <section>
              <h3>Additional Details</h3>
              <textarea
                value={formData.additional_details}
                onChange={(e) => setFormData({...formData, additional_details: e.target.value})}
                placeholder="Add specific details..."
                rows="3"
              />
            </section>

            <section>
              <h3>Settings</h3>
              <label>
                Variations:
                <input
                  type="number"
                  min="1"
                  max="8"
                  value={formData.num_variations}
                  onChange={(e) => setFormData({...formData, num_variations: parseInt(e.target.value)})}
                />
              </label>

              <label>
                Model:
                <select
                  value={formData.model}
                  onChange={(e) => setFormData({...formData, model: e.target.value})}
                >
                  <option value="stable-diffusion">Stable Diffusion</option>
                  <option value="dalle">DALL-E 3</option>
                </select>
              </label>
            </section>

            <button type="submit" disabled={loading}>
              {loading ? 'Generating...' : 'Generate Designs'}
            </button>
          </form>
        </aside>

        <main className="content">
          {error && (
            <div className="error">
              ❌ {error}
            </div>
          )}

          {prompt && (
            <div className="prompt-display">
              <strong>Generated Prompt:</strong>
              <p>{prompt}</p>
            </div>
          )}

          {loading && (
            <div className="loading">
              <div className="spinner"></div>
              <p>Generating your room designs... This may take 30-60 seconds</p>
            </div>
          )}

          {images.length > 0 && (
            <div className="gallery">
              {images.map((img, idx) => (
                <div key={idx} className="image-card">
                  <img src={img} alt={`Design variation ${idx + 1}`} />
                  <div className="image-actions">
                    <button onClick={() => window.open(img, '_blank')}>View Full</button>
                    <button onClick={() => {/* upscale */}}>Upscale</button>
                    <button onClick={() => {/* download */}}>Download</button>
                  </div>
                </div>
              ))}
            </div>
          )}
        </main>
      </div>
    </div>
  );
}

export default App;
```

## Expected Outputs

1. **Photorealistic Room Renders**: 4-8 AI-generated images per request
2. **Multiple Variations**: Different angles, lighting, furniture arrangements
3. **Style Consistency**: Matches specified Feng Shui palette and design style
4. **High Resolution**: 1024x1024 base, upscalable to 4K
5. **Fast Generation**: 30-60 seconds for batch of 4 images

## Bonus Challenges

1. **ControlNet Integration**: Use depth maps for precise spatial control
2. **Inpainting**: Edit specific areas (change wall color, swap furniture)
3. **Style Transfer**: Apply artistic styles to generated rooms
4. **360° Panoramas**: Generate panoramic room views
5. **Animation**: Create walk-through videos
6. **Virtual Staging**: Add furniture to empty room photos
7. **Cost Calculator**: Estimate renovation costs from generated designs

## Resources

- [Stable Diffusion Documentation](https://github.com/Stability-AI/stablediffusion)
- [DALL-E API Guide](https://platform.openai.com/docs/guides/images)
- [ControlNet GitHub](https://github.com/lllyasviel/ControlNet)
- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [Interior Design Prompts Database](https://prompthero.com/)

## Success Criteria

- [ ] Generate 4 variations in under 60 seconds
- [ ] Images match specified color palette
- [ ] Room proportions appear accurate
- [ ] Furniture placement looks realistic
- [ ] No major artifacts or distortions
- [ ] Upscaling produces sharp 4K images
- [ ] API handles errors gracefully
- [ ] Frontend provides smooth UX
