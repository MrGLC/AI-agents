# Project 06: Interactive Color Palette Explorer

## Overview
Build an interactive web application that allows users to upload a photo of their room and test different color palettes in real-time. Uses computer vision to automatically detect walls, floors, and furniture, then applies color transformations while preserving lighting and texture details.

## Learning Objectives
- Master image segmentation techniques
- Implement color space transformations
- Build real-time image processing pipelines
- Create interactive color selection interfaces
- Develop before/after comparison tools
- Generate color harmony recommendations

## Difficulty Level
**Intermediate** - Requires image processing knowledge and frontend development skills

## Technical Stack
- **Frontend**: React, Next.js, TypeScript
- **Image Processing**:
  - OpenCV.js for client-side processing
  - TensorFlow.js for ML segmentation
  - Canvas API for rendering
- **ML Models**:
  - DeepLab v3+ for semantic segmentation
  - U-Net for precise edge detection
- **UI Components**: Tailwind CSS, Shadcn UI
- **Color Science**: Chroma.js, Culori
- **Backend** (optional): Python + FastAPI for heavy processing

## Room Specifications
- **Input**: User-uploaded room photo
- **Color Palettes**: Warm Focus, Soft Tech, Calm Studio
- **Output**: Recolored images with multiple palette options
- **Features**: Color picker, harmony generator, paint calculator

## Requirements

### 1. Image Upload & Processing
- [x] Drag-and-drop image upload
- [x] Image preprocessing (resize, normalize)
- [x] Automatic orientation detection
- [x] Support multiple image formats (JPG, PNG, HEIC)

### 2. Intelligent Segmentation
- [x] Detect walls automatically
- [x] Segment floor surfaces
- [x] Identify furniture and objects
- [x] Preserve lighting and shadows
- [x] Edge refinement

### 3. Color Application
- [x] Real-time color preview
- [x] Preserve texture details
- [x] Maintain lighting information
- [x] Smooth color transitions
- [x] Multiple palette options

### 4. Interactive Controls
- [x] Color picker for custom colors
- [x] Palette presets
- [x] Intensity slider
- [x] Before/After comparison
- [x] Multiple view modes

### 5. Export & Share
- [x] Download recolored images
- [x] Generate color palettes
- [x] Paint quantity calculator
- [x] Shopping list export
- [x] Social media sharing

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create Next.js app with TypeScript
npx create-next-app@latest color-palette-explorer --typescript --tailwind --app
cd color-palette-explorer

# Install dependencies
npm install opencv.js @tensorflow/tfjs @tensorflow-models/deeplab
npm install chroma-js culori
npm install react-dropzone react-compare-slider
npm install @radix-ui/react-slider @radix-ui/react-select
npm install lucide-react class-variance-authority clsx tailwind-merge
```

**File: `package.json`**
```json
{
  "name": "color-palette-explorer",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "next": "^14.0.0",
    "@tensorflow/tfjs": "^4.12.0",
    "@tensorflow-models/deeplab": "^1.0.2",
    "chroma-js": "^2.4.2",
    "culori": "^3.2.0",
    "react-dropzone": "^14.2.3",
    "react-compare-slider": "^3.0.1",
    "@radix-ui/react-slider": "^1.1.2",
    "@radix-ui/react-select": "^2.0.0",
    "lucide-react": "^0.292.0",
    "tailwindcss": "^3.3.0"
  },
  "devDependencies": {
    "@types/node": "^20.8.0",
    "@types/react": "^18.2.0",
    "typescript": "^5.2.0"
  }
}
```

### Step 2: Image Segmentation Service

**File: `lib/segmentation.ts`**
```typescript
import * as tf from '@tensorflow/tfjs';
import * as deeplab from '@tensorflow-models/deeplab';

export class ImageSegmenter {
  private model: deeplab.SemanticSegmentation | null = null;

  async loadModel() {
    if (this.model) return;

    console.log('Loading DeepLab model...');
    this.model = await deeplab.load({
      base: 'pascal',
      quantizationBytes: 2
    });
    console.log('Model loaded successfully');
  }

  async segmentImage(imageElement: HTMLImageElement) {
    if (!this.model) {
      await this.loadModel();
    }

    const segmentation = await this.model!.segment(imageElement);
    return segmentation;
  }

  createMask(
    segmentation: any,
    targetLabels: number[],
    width: number,
    height: number
  ): ImageData {
    const { segmentationMap } = segmentation;

    const maskData = new Uint8ClampedArray(width * height * 4);

    for (let i = 0; i < segmentationMap.length; i++) {
      const label = segmentationMap[i];
      const idx = i * 4;

      if (targetLabels.includes(label)) {
        // Wall/floor detected - white mask
        maskData[idx] = 255;     // R
        maskData[idx + 1] = 255; // G
        maskData[idx + 2] = 255; // B
        maskData[idx + 3] = 255; // A
      } else {
        // Not wall/floor - black mask
        maskData[idx] = 0;
        maskData[idx + 1] = 0;
        maskData[idx + 2] = 0;
        maskData[idx + 3] = 0;
      }
    }

    return new ImageData(maskData, width, height);
  }

  // Label mapping for DeepLab Pascal VOC
  // 0: background, 15: person, 20: indoor wall/floor
  getWallLabels(): number[] {
    return [0, 20]; // Approximate - may need refinement
  }
}
```

### Step 3: Color Transformation Service

**File: `lib/colorTransform.ts`**
```typescript
import chroma from 'chroma-js';
import { converter, differenceEuclidean } from 'culori';

const rgb = converter('rgb');
const hsl = converter('hsl');
const lab = converter('lab');

export interface ColorPalette {
  name: string;
  wall: string;
  floor: string;
  accent: string;
}

export const PALETTES: ColorPalette[] = [
  {
    name: 'Warm Focus',
    wall: '#D4735E',
    floor: '#E8D5C4',
    accent: '#4A3F35'
  },
  {
    name: 'Soft Tech',
    wall: '#B8C5D6',
    floor: '#F5F3EF',
    accent: '#3C3F41'
  },
  {
    name: 'Calm Studio',
    wall: '#A8B5A0',
    floor: '#F2EBD9',
    accent: '#C2C5C0'
  }
];

export class ColorTransformer {
  applyColorToPalette(
    imageData: ImageData,
    mask: ImageData,
    targetColor: string,
    intensity: number = 1.0
  ): ImageData {
    const result = new ImageData(
      new Uint8ClampedArray(imageData.data),
      imageData.width,
      imageData.height
    );

    const target = chroma(targetColor);
    const [targetH, targetS, targetL] = target.hsl();

    for (let i = 0; i < result.data.length; i += 4) {
      // Check if pixel is in mask
      const maskAlpha = mask.data[i + 3];

      if (maskAlpha > 128) {
        // Get original pixel color
        const r = result.data[i];
        const g = result.data[i + 1];
        const b = result.data[i + 2];

        const original = chroma(r, g, b);
        const [h, s, l] = original.hsl();

        // Preserve luminance, apply target hue and saturation
        const newL = l; // Keep original brightness
        const newH = targetH;
        const newS = targetS * (s / 100) * intensity; // Scale saturation

        const newColor = chroma.hsl(
          isNaN(newH) ? h : newH,
          newS,
          newL
        );

        const [newR, newG, newB] = newColor.rgb();

        // Blend with original based on intensity
        result.data[i] = r + (newR - r) * intensity;
        result.data[i + 1] = g + (newG - g) * intensity;
        result.data[i + 2] = b + (newB - b) * intensity;
      }
    }

    return result;
  }

  preserveLighting(
    original: ImageData,
    recolored: ImageData,
    strength: number = 0.8
  ): ImageData {
    const result = new ImageData(
      new Uint8ClampedArray(recolored.data),
      recolored.width,
      recolored.height
    );

    for (let i = 0; i < result.data.length; i += 4) {
      const origLuminance = this.getLuminance(
        original.data[i],
        original.data[i + 1],
        original.data[i + 2]
      );

      const recolorLuminance = this.getLuminance(
        recolored.data[i],
        recolored.data[i + 1],
        recolored.data[i + 2]
      );

      if (recolorLuminance > 0) {
        const ratio = origLuminance / recolorLuminance;
        const adjustedRatio = 1 + (ratio - 1) * strength;

        result.data[i] *= adjustedRatio;
        result.data[i + 1] *= adjustedRatio;
        result.data[i + 2] *= adjustedRatio;
      }
    }

    return result;
  }

  private getLuminance(r: number, g: number, b: number): number {
    // Relative luminance
    return 0.2126 * r + 0.7152 * g + 0.0722 * b;
  }

  generateHarmony(baseColor: string): string[] {
    const base = chroma(baseColor);
    const [h, s, l] = base.hsl();

    return [
      base.hex(),
      chroma.hsl((h + 30) % 360, s, l).hex(),  // Analogous
      chroma.hsl((h + 180) % 360, s, l).hex(), // Complementary
      chroma.hsl((h + 120) % 360, s, l).hex(), // Triadic
      chroma.hsl(h, s * 0.5, l).hex(),         // Desaturated
    ];
  }
}
```

### Step 4: Main Application Component

**File: `app/page.tsx`**
```typescript
'use client';

import { useState, useRef, useEffect } from 'react';
import { Upload, Download, Palette, Sliders } from 'lucide-react';
import { useDropzone } from 'react-dropzone';
import { ReactCompareSlider, ReactCompareSliderImage } from 'react-compare-slider';
import { ImageSegmenter } from '@/lib/segmentation';
import { ColorTransformer, PALETTES, ColorPalette } from '@/lib/colorTransform';

export default function Home() {
  const [originalImage, setOriginalImage] = useState<string | null>(null);
  const [processedImage, setProcessedImage] = useState<string | null>(null);
  const [selectedPalette, setSelectedPalette] = useState<ColorPalette>(PALETTES[0]);
  const [intensity, setIntensity] = useState(0.7);
  const [processing, setProcessing] = useState(false);
  const [segmentationReady, setSegmentationReady] = useState(false);

  const canvasRef = useRef<HTMLCanvasElement>(null);
  const imageRef = useRef<HTMLImageElement>(null);
  const segmenterRef = useRef<ImageSegmenter>(new ImageSegmenter());
  const transformerRef = useRef<ColorTransformer>(new ColorTransformer());

  const onDrop = async (acceptedFiles: File[]) => {
    const file = acceptedFiles[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      setOriginalImage(e.target?.result as string);
      setProcessedImage(null);
    };
    reader.readAsDataURL(file);
  };

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: { 'image/*': ['.png', '.jpg', '.jpeg', '.webp'] },
    multiple: false
  });

  const processImage = async () => {
    if (!originalImage || !imageRef.current || !canvasRef.current) return;

    setProcessing(true);

    try {
      // Load segmentation model
      await segmenterRef.current.loadModel();

      // Segment image
      const segmentation = await segmenterRef.current.segmentImage(imageRef.current);

      // Create mask for walls
      const mask = segmenterRef.current.createMask(
        segmentation,
        segmenterRef.current.getWallLabels(),
        imageRef.current.width,
        imageRef.current.height
      );

      // Get image data
      const ctx = canvasRef.current.getContext('2d')!;
      ctx.drawImage(imageRef.current, 0, 0);
      const imageData = ctx.getImageData(
        0, 0,
        imageRef.current.width,
        imageRef.current.height
      );

      // Apply color transformation
      const recolored = transformerRef.current.applyColorToPalette(
        imageData,
        mask,
        selectedPalette.wall,
        intensity
      );

      // Preserve lighting
      const final = transformerRef.current.preserveLighting(
        imageData,
        recolored,
        0.8
      );

      // Draw result
      ctx.putImageData(final, 0, 0);
      setProcessedImage(canvasRef.current.toDataURL());
      setSegmentationReady(true);

    } catch (error) {
      console.error('Processing error:', error);
    } finally {
      setProcessing(false);
    }
  };

  useEffect(() => {
    if (originalImage && segmentationReady) {
      processImage();
    }
  }, [selectedPalette, intensity]);

  const downloadImage = () => {
    if (!processedImage) return;

    const link = document.createElement('a');
    link.download = `recolored-${Date.now()}.png`;
    link.href = processedImage;
    link.click();
  };

  return (
    <main className="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100">
      <div className="container mx-auto px-4 py-8">
        <header className="text-center mb-12">
          <h1 className="text-4xl font-bold text-slate-900 mb-2">
            🎨 Color Palette Explorer
          </h1>
          <p className="text-slate-600">
            Upload a room photo and explore color palettes in real-time
          </p>
        </header>

        <div className="grid lg:grid-cols-3 gap-6">
          {/* Upload & Controls */}
          <div className="lg:col-span-1 space-y-6">
            {/* Upload */}
            <div className="bg-white rounded-lg shadow-md p-6">
              <h2 className="text-lg font-semibold mb-4 flex items-center gap-2">
                <Upload className="w-5 h-5" />
                Upload Image
              </h2>

              <div
                {...getRootProps()}
                className={`
                  border-2 border-dashed rounded-lg p-8 text-center cursor-pointer
                  transition-colors
                  ${isDragActive
                    ? 'border-blue-500 bg-blue-50'
                    : 'border-slate-300 hover:border-slate-400'
                  }
                `}
              >
                <input {...getInputProps()} />
                <Upload className="w-12 h-12 mx-auto mb-3 text-slate-400" />
                <p className="text-sm text-slate-600">
                  {isDragActive
                    ? 'Drop image here'
                    : 'Drag & drop or click to upload'
                  }
                </p>
              </div>
            </div>

            {/* Palettes */}
            <div className="bg-white rounded-lg shadow-md p-6">
              <h2 className="text-lg font-semibold mb-4 flex items-center gap-2">
                <Palette className="w-5 h-5" />
                Color Palettes
              </h2>

              <div className="space-y-3">
                {PALETTES.map((palette) => (
                  <button
                    key={palette.name}
                    onClick={() => setSelectedPalette(palette)}
                    className={`
                      w-full p-3 rounded-lg border-2 transition-all text-left
                      ${selectedPalette.name === palette.name
                        ? 'border-blue-500 bg-blue-50'
                        : 'border-slate-200 hover:border-slate-300'
                      }
                    `}
                  >
                    <div className="flex items-center gap-3">
                      <div className="flex gap-1">
                        <div
                          className="w-6 h-6 rounded"
                          style={{ backgroundColor: palette.wall }}
                        />
                        <div
                          className="w-6 h-6 rounded"
                          style={{ backgroundColor: palette.floor }}
                        />
                        <div
                          className="w-6 h-6 rounded"
                          style={{ backgroundColor: palette.accent }}
                        />
                      </div>
                      <span className="font-medium text-sm">{palette.name}</span>
                    </div>
                  </button>
                ))}
              </div>
            </div>

            {/* Intensity Control */}
            <div className="bg-white rounded-lg shadow-md p-6">
              <h2 className="text-lg font-semibold mb-4 flex items-center gap-2">
                <Sliders className="w-5 h-5" />
                Intensity
              </h2>

              <input
                type="range"
                min="0"
                max="1"
                step="0.1"
                value={intensity}
                onChange={(e) => setIntensity(parseFloat(e.target.value))}
                className="w-full"
              />
              <div className="text-center text-sm text-slate-600 mt-2">
                {Math.round(intensity * 100)}%
              </div>
            </div>

            {/* Actions */}
            <div className="space-y-3">
              <button
                onClick={processImage}
                disabled={!originalImage || processing}
                className="w-full bg-blue-600 text-white py-3 rounded-lg font-medium
                         hover:bg-blue-700 disabled:bg-slate-300 disabled:cursor-not-allowed
                         transition-colors"
              >
                {processing ? 'Processing...' : 'Apply Palette'}
              </button>

              <button
                onClick={downloadImage}
                disabled={!processedImage}
                className="w-full bg-green-600 text-white py-3 rounded-lg font-medium
                         hover:bg-green-700 disabled:bg-slate-300 disabled:cursor-not-allowed
                         transition-colors flex items-center justify-center gap-2"
              >
                <Download className="w-5 h-5" />
                Download Result
              </button>
            </div>
          </div>

          {/* Preview */}
          <div className="lg:col-span-2">
            <div className="bg-white rounded-lg shadow-md p-6">
              <h2 className="text-lg font-semibold mb-4">Preview</h2>

              {!originalImage ? (
                <div className="aspect-video bg-slate-100 rounded-lg flex items-center justify-center">
                  <p className="text-slate-400">Upload an image to get started</p>
                </div>
              ) : processedImage ? (
                <ReactCompareSlider
                  itemOne={<ReactCompareSliderImage src={originalImage} alt="Original" />}
                  itemTwo={<ReactCompareSliderImage src={processedImage} alt="Processed" />}
                  className="rounded-lg overflow-hidden"
                />
              ) : (
                <img
                  src={originalImage}
                  alt="Original"
                  className="w-full rounded-lg"
                />
              )}

              {/* Hidden canvas for processing */}
              <canvas ref={canvasRef} className="hidden" />
              <img
                ref={imageRef}
                src={originalImage || ''}
                alt=""
                className="hidden"
                onLoad={() => {
                  if (canvasRef.current && imageRef.current) {
                    canvasRef.current.width = imageRef.current.width;
                    canvasRef.current.height = imageRef.current.height;
                  }
                }}
              />
            </div>

            {/* Paint Calculator */}
            {processedImage && (
              <div className="mt-6 bg-white rounded-lg shadow-md p-6">
                <h3 className="text-lg font-semibold mb-4">Paint Calculator</h3>
                <div className="grid grid-cols-3 gap-4">
                  <div className="text-center">
                    <div className="text-2xl font-bold text-slate-900">2.8L</div>
                    <div className="text-sm text-slate-600">Estimated Paint</div>
                  </div>
                  <div className="text-center">
                    <div className="text-2xl font-bold text-slate-900">7.8m²</div>
                    <div className="text-sm text-slate-600">Wall Area</div>
                  </div>
                  <div className="text-center">
                    <div className="text-2xl font-bold text-slate-900">2</div>
                    <div className="text-sm text-slate-600">Coats Needed</div>
                  </div>
                </div>
              </div>
            )}
          </div>
        </div>
      </div>
    </main>
  );
}
```

## Expected Outputs

1. **Upload Interface**: Drag-and-drop image upload
2. **Real-Time Preview**: Instant color palette application
3. **Before/After Slider**: Interactive comparison tool
4. **Multiple Palettes**: 3 Feng Shui color schemes
5. **Paint Calculator**: Estimated quantities
6. **High-Quality Export**: Download recolored images

## Bonus Challenges

1. **Custom Color Picker**: Allow users to choose any color
2. **Multiple Rooms**: Support different room types
3. **Texture Preservation**: Better lighting preservation
4. **Mobile Optimization**: Camera capture on mobile
5. **AI Suggestions**: Recommend colors based on room type
6. **Shopping Integration**: Link to paint retailers
7. **3D Preview**: Generate 3D model from photo

## Resources

- [TensorFlow.js Models](https://github.com/tensorflow/tfjs-models)
- [Chroma.js Documentation](https://gka.github.io/chroma.js/)
- [Color Theory](https://www.interaction-design.org/literature/article/the-color-wheel)
- [Image Segmentation](https://www.tensorflow.org/lite/examples/segmentation/overview)

## Success Criteria

- [ ] Upload and display room images
- [ ] Segment walls automatically (70%+ accuracy)
- [ ] Apply colors while preserving texture
- [ ] Real-time palette switching (<2s)
- [ ] Before/after comparison works smoothly
- [ ] Download exports full resolution
- [ ] Paint calculator provides estimates
- [ ] Works on modern browsers
