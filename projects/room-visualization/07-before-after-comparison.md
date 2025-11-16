# Project 07: Before/After Room Comparison Tool

## Overview
Create a comprehensive before/after visualization tool for room designs with multiple comparison modes: side-by-side, split-screen slider, overlay fade, and animated transitions. Perfect for showcasing design transformations to clients or sharing renovation progress.

## Learning Objectives
- Implement multiple image comparison techniques
- Build smooth animation transitions
- Create interactive slider controls
- Develop responsive gallery layouts
- Master CSS blend modes and filters
- Build shareable comparison views

## Difficulty Level
**Beginner to Intermediate** - Great introduction to modern web development

## Technical Stack
- **Frontend**: HTML5, CSS3, JavaScript (Vanilla or React)
- **UI Framework**: Optional - React, Vue, or Svelte
- **Styling**: CSS Grid, Flexbox, CSS Animations
- **Libraries**:
  - React Compare Image (for slider)
  - GSAP (for animations)
  - html2canvas (for screenshots)
  - FileSaver.js (for downloads)
- **Build Tool**: Vite or Parcel

## Room Specifications
- **Before Images**: Original room photos
- **After Images**: Designed/rendered rooms
- **Dimensions**: Match exactly for proper comparison
- **Color Palettes**: Show transformation with Feng Shui palettes
- **Multiple Views**: Different angles of same room

## Requirements

### 1. Comparison Modes
- [x] Side-by-side view
- [x] Split-screen slider (vertical & horizontal)
- [x] Overlay fade transition
- [x] Animated flip transition
- [x] Grid view (multiple before/after pairs)

### 2. Interactive Controls
- [x] Smooth slider with handle
- [x] Keyboard navigation (arrow keys)
- [x] Touch support for mobile
- [x] Automatic transition timer
- [x] Zoom and pan

### 3. Annotations
- [x] Add labels and callouts
- [x] Highlight changes
- [x] Add measurements
- [x] Color palette indicators
- [x] Custom text overlays

### 4. Export Options
- [x] Download comparison image
- [x] Generate shareable link
- [x] Create presentation PDF
- [x] Export as video (GIF/MP4)
- [x] Social media formats

### 5. Gallery Management
- [x] Multiple comparison sets
- [x] Project organization
- [x] Image upload and management
- [x] Thumbnail previews
- [x] Metadata and tags

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create project directory
mkdir before-after-comparison
cd before-after-comparison

# Initialize with Vite
npm create vite@latest . -- --template vanilla
npm install

# Install dependencies
npm install gsap
npm install html2canvas
npm install file-saver
```

**File: `package.json`**
```json
{
  "name": "before-after-comparison",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "gsap": "^3.12.2",
    "html2canvas": "^1.4.1",
    "file-saver": "^2.0.5"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

### Step 2: HTML Structure

**File: `index.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Before/After Room Comparison</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="app-container">
        <!-- Header -->
        <header class="header">
            <h1>🏠 Room Transformation Comparison</h1>
            <p class="subtitle">Visualize your design changes</p>
        </header>

        <!-- Controls -->
        <div class="controls-bar">
            <div class="control-group">
                <label>Comparison Mode:</label>
                <select id="comparisonMode" class="mode-select">
                    <option value="slider">Split Slider</option>
                    <option value="sidebyside">Side by Side</option>
                    <option value="fade">Overlay Fade</option>
                    <option value="flip">Flip Animation</option>
                </select>
            </div>

            <div class="control-group">
                <label>Slider Direction:</label>
                <select id="sliderDirection" class="mode-select">
                    <option value="horizontal">Horizontal</option>
                    <option value="vertical">Vertical</option>
                </select>
            </div>

            <div class="control-group">
                <button id="autoPlayBtn" class="btn btn-secondary">
                    ▶️ Auto Play
                </button>
            </div>

            <div class="control-group">
                <button id="downloadBtn" class="btn btn-primary">
                    💾 Download
                </button>
            </div>
        </div>

        <!-- Main Comparison Area -->
        <div class="comparison-container">
            <!-- Slider Mode -->
            <div id="sliderMode" class="comparison-mode active">
                <div class="image-compare-slider horizontal">
                    <div class="image-wrapper before-wrapper">
                        <img src="assets/room-before.jpg" alt="Before" class="comparison-image">
                        <div class="image-label label-before">BEFORE</div>
                    </div>
                    <div class="image-wrapper after-wrapper">
                        <img src="assets/room-after.jpg" alt="After" class="comparison-image">
                        <div class="image-label label-after">AFTER</div>
                    </div>
                    <div class="slider-handle">
                        <div class="handle-line"></div>
                        <div class="handle-circle">
                            <div class="arrow-left">‹</div>
                            <div class="arrow-right">›</div>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Side by Side Mode -->
            <div id="sideBySideMode" class="comparison-mode">
                <div class="side-by-side-container">
                    <div class="image-panel">
                        <img src="assets/room-before.jpg" alt="Before">
                        <div class="panel-label">BEFORE</div>
                    </div>
                    <div class="image-panel">
                        <img src="assets/room-after.jpg" alt="After">
                        <div class="panel-label">AFTER</div>
                    </div>
                </div>
            </div>

            <!-- Fade Mode -->
            <div id="fadeMode" class="comparison-mode">
                <div class="fade-container">
                    <img src="assets/room-before.jpg" alt="Before" class="fade-before">
                    <img src="assets/room-after.jpg" alt="After" class="fade-after">
                </div>
                <div class="fade-controls">
                    <label>Opacity:</label>
                    <input type="range" id="fadeSlider" min="0" max="100" value="50" class="fade-slider">
                    <span id="fadeValue">50%</span>
                </div>
            </div>

            <!-- Flip Mode -->
            <div id="flipMode" class="comparison-mode">
                <div class="flip-container">
                    <div class="flip-card">
                        <div class="flip-card-inner">
                            <div class="flip-card-front">
                                <img src="assets/room-before.jpg" alt="Before">
                                <div class="flip-label">BEFORE</div>
                            </div>
                            <div class="flip-card-back">
                                <img src="assets/room-after.jpg" alt="After">
                                <div class="flip-label">AFTER</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Palette Comparison -->
        <div class="palette-comparison">
            <h3>Color Palette Transformation</h3>
            <div class="palette-row">
                <div class="palette-set">
                    <div class="palette-label">Before</div>
                    <div class="color-swatches">
                        <div class="swatch" style="background: #E8DCC8;"></div>
                        <div class="swatch" style="background: #A89B8C;"></div>
                        <div class="swatch" style="background: #6B6356;"></div>
                    </div>
                </div>
                <div class="arrow">→</div>
                <div class="palette-set">
                    <div class="palette-label">After (Warm Focus)</div>
                    <div class="color-swatches">
                        <div class="swatch" style="background: #D4735E;"></div>
                        <div class="swatch" style="background: #E8D5C4;"></div>
                        <div class="swatch" style="background: #4A3F35;"></div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Change Highlights -->
        <div class="changes-section">
            <h3>Key Improvements</h3>
            <div class="changes-grid">
                <div class="change-card">
                    <div class="change-icon">🎨</div>
                    <h4>Color Palette</h4>
                    <p>Warm, cohesive Feng Shui colors</p>
                </div>
                <div class="change-card">
                    <div class="change-icon">💡</div>
                    <h4>Lighting</h4>
                    <p>Enhanced natural light with warm accents</p>
                </div>
                <div class="change-card">
                    <div class="change-icon">🪑</div>
                    <h4>Furniture</h4>
                    <p>Optimized layout for 2.83m × 2.75m space</p>
                </div>
                <div class="change-card">
                    <div class="change-icon">✨</div>
                    <h4>Atmosphere</h4>
                    <p>Cozy, productive environment</p>
                </div>
            </div>
        </div>

        <!-- Gallery -->
        <div class="gallery-section">
            <h3>More Views</h3>
            <div class="thumbnail-gallery">
                <div class="thumbnail-item active">
                    <img src="assets/room-before.jpg" alt="View 1">
                    <span>Main View</span>
                </div>
                <div class="thumbnail-item">
                    <img src="assets/room-corner-before.jpg" alt="View 2">
                    <span>Corner View</span>
                </div>
                <div class="thumbnail-item">
                    <img src="assets/room-desk-before.jpg" alt="View 3">
                    <span>Desk Area</span>
                </div>
            </div>
        </div>
    </div>

    <script type="module" src="main.js"></script>
</body>
</html>
```

### Step 3: CSS Styling

**File: `style.css`**
```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: #333;
    line-height: 1.6;
}

.app-container {
    max-width: 1400px;
    margin: 0 auto;
    padding: 20px;
}

/* Header */
.header {
    text-align: center;
    color: white;
    padding: 40px 20px;
}

.header h1 {
    font-size: 2.5rem;
    margin-bottom: 10px;
}

.subtitle {
    font-size: 1.1rem;
    opacity: 0.9;
}

/* Controls */
.controls-bar {
    background: white;
    border-radius: 12px;
    padding: 20px;
    margin-bottom: 20px;
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    align-items: center;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.control-group {
    display: flex;
    align-items: center;
    gap: 10px;
}

.control-group label {
    font-weight: 600;
    color: #555;
}

.mode-select {
    padding: 8px 16px;
    border: 2px solid #e0e0e0;
    border-radius: 6px;
    font-size: 14px;
    cursor: pointer;
    transition: border-color 0.3s;
}

.mode-select:hover {
    border-color: #667eea;
}

.btn {
    padding: 10px 20px;
    border: none;
    border-radius: 6px;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s;
}

.btn-primary {
    background: #667eea;
    color: white;
}

.btn-primary:hover {
    background: #5568d3;
    transform: translateY(-2px);
}

.btn-secondary {
    background: #48bb78;
    color: white;
}

.btn-secondary:hover {
    background: #38a169;
    transform: translateY(-2px);
}

/* Comparison Container */
.comparison-container {
    background: white;
    border-radius: 12px;
    padding: 20px;
    margin-bottom: 20px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    min-height: 600px;
}

.comparison-mode {
    display: none;
}

.comparison-mode.active {
    display: block;
}

/* Slider Mode */
.image-compare-slider {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 9;
    overflow: hidden;
    border-radius: 8px;
    user-select: none;
}

.image-wrapper {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.after-wrapper {
    clip-path: inset(0 50% 0 0);
}

.image-compare-slider.vertical .after-wrapper {
    clip-path: inset(50% 0 0 0);
}

.comparison-image {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.image-label {
    position: absolute;
    padding: 8px 16px;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    font-weight: 600;
    font-size: 14px;
    border-radius: 4px;
}

.label-before {
    top: 20px;
    left: 20px;
}

.label-after {
    top: 20px;
    right: 20px;
}

.slider-handle {
    position: absolute;
    top: 0;
    left: 50%;
    width: 4px;
    height: 100%;
    background: white;
    cursor: ew-resize;
    transform: translateX(-50%);
    z-index: 10;
}

.image-compare-slider.vertical .slider-handle {
    top: 50%;
    left: 0;
    width: 100%;
    height: 4px;
    cursor: ns-resize;
    transform: translateY(-50%);
}

.handle-circle {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 48px;
    height: 48px;
    background: white;
    border: 3px solid #667eea;
    border-radius: 50%;
    transform: translate(-50%, -50%);
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.arrow-left,
.arrow-right {
    font-size: 18px;
    font-weight: bold;
    color: #667eea;
}

/* Side by Side Mode */
.side-by-side-container {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
}

.image-panel {
    position: relative;
    border-radius: 8px;
    overflow: hidden;
}

.image-panel img {
    width: 100%;
    height: auto;
    display: block;
}

.panel-label {
    position: absolute;
    top: 20px;
    left: 20px;
    padding: 8px 16px;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    font-weight: 600;
    border-radius: 4px;
}

/* Fade Mode */
.fade-container {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 9;
    border-radius: 8px;
    overflow: hidden;
}

.fade-before,
.fade-after {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.fade-after {
    opacity: 0.5;
    transition: opacity 0.3s;
}

.fade-controls {
    margin-top: 20px;
    display: flex;
    align-items: center;
    gap: 10px;
}

.fade-slider {
    flex: 1;
    height: 6px;
    border-radius: 3px;
    outline: none;
    background: linear-gradient(to right, #667eea, #764ba2);
}

/* Flip Mode */
.flip-container {
    perspective: 1000px;
    width: 100%;
    aspect-ratio: 16 / 9;
}

.flip-card {
    position: relative;
    width: 100%;
    height: 100%;
    cursor: pointer;
}

.flip-card-inner {
    position: relative;
    width: 100%;
    height: 100%;
    transition: transform 0.8s;
    transform-style: preserve-3d;
}

.flip-card:hover .flip-card-inner {
    transform: rotateY(180deg);
}

.flip-card-front,
.flip-card-back {
    position: absolute;
    width: 100%;
    height: 100%;
    backface-visibility: hidden;
    border-radius: 8px;
    overflow: hidden;
}

.flip-card-back {
    transform: rotateY(180deg);
}

.flip-card-front img,
.flip-card-back img {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

.flip-label {
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    padding: 8px 20px;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    font-weight: 600;
    border-radius: 4px;
}

/* Palette Comparison */
.palette-comparison {
    background: white;
    border-radius: 12px;
    padding: 30px;
    margin-bottom: 20px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.palette-comparison h3 {
    text-align: center;
    margin-bottom: 20px;
    color: #333;
}

.palette-row {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 40px;
}

.palette-set {
    text-align: center;
}

.palette-label {
    font-weight: 600;
    margin-bottom: 10px;
    color: #555;
}

.color-swatches {
    display: flex;
    gap: 10px;
}

.swatch {
    width: 60px;
    height: 60px;
    border-radius: 8px;
    border: 3px solid white;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.arrow {
    font-size: 32px;
    color: #667eea;
    font-weight: bold;
}

/* Changes Section */
.changes-section {
    background: white;
    border-radius: 12px;
    padding: 30px;
    margin-bottom: 20px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.changes-section h3 {
    text-align: center;
    margin-bottom: 30px;
    color: #333;
}

.changes-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 20px;
}

.change-card {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    padding: 30px;
    border-radius: 12px;
    text-align: center;
    color: white;
    transition: transform 0.3s;
}

.change-card:hover {
    transform: translateY(-5px);
}

.change-icon {
    font-size: 48px;
    margin-bottom: 15px;
}

.change-card h4 {
    margin-bottom: 10px;
    font-size: 1.2rem;
}

.change-card p {
    opacity: 0.9;
    font-size: 0.95rem;
}

/* Gallery */
.gallery-section {
    background: white;
    border-radius: 12px;
    padding: 30px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.gallery-section h3 {
    margin-bottom: 20px;
    color: #333;
}

.thumbnail-gallery {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 15px;
}

.thumbnail-item {
    position: relative;
    border-radius: 8px;
    overflow: hidden;
    cursor: pointer;
    border: 3px solid transparent;
    transition: all 0.3s;
}

.thumbnail-item:hover,
.thumbnail-item.active {
    border-color: #667eea;
    transform: scale(1.05);
}

.thumbnail-item img {
    width: 100%;
    height: 150px;
    object-fit: cover;
}

.thumbnail-item span {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 8px;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    font-size: 12px;
    text-align: center;
}

/* Responsive */
@media (max-width: 768px) {
    .header h1 {
        font-size: 1.8rem;
    }

    .controls-bar {
        flex-direction: column;
        align-items: stretch;
    }

    .side-by-side-container {
        grid-template-columns: 1fr;
    }

    .palette-row {
        flex-direction: column;
    }

    .changes-grid {
        grid-template-columns: 1fr;
    }
}
```

### Step 4: JavaScript Functionality

**File: `main.js`**
```javascript
import gsap from 'gsap';
import html2canvas from 'html2canvas';
import { saveAs } from 'file-saver';

class ComparisonTool {
    constructor() {
        this.currentMode = 'slider';
        this.sliderPosition = 50;
        this.isDragging = false;
        this.autoPlayInterval = null;

        this.init();
    }

    init() {
        this.setupEventListeners();
        this.setupSlider();
    }

    setupEventListeners() {
        // Mode selection
        document.getElementById('comparisonMode').addEventListener('change', (e) => {
            this.switchMode(e.target.value);
        });

        // Slider direction
        document.getElementById('sliderDirection').addEventListener('change', (e) => {
            this.setSliderDirection(e.target.value);
        });

        // Auto play
        document.getElementById('autoPlayBtn').addEventListener('click', () => {
            this.toggleAutoPlay();
        });

        // Download
        document.getElementById('downloadBtn').addEventListener('click', () => {
            this.downloadComparison();
        });

        // Fade slider
        document.getElementById('fadeSlider')?.addEventListener('input', (e) => {
            this.updateFadeOpacity(e.target.value);
        });

        // Thumbnail gallery
        document.querySelectorAll('.thumbnail-item').forEach(item => {
            item.addEventListener('click', (e) => {
                this.switchGalleryImage(e.currentTarget);
            });
        });
    }

    setupSlider() {
        const slider = document.querySelector('.image-compare-slider');
        const handle = slider.querySelector('.slider-handle');
        const afterWrapper = slider.querySelector('.after-wrapper');

        const updateSlider = (e) => {
            if (!this.isDragging && e.type === 'mousemove') return;

            const rect = slider.getBoundingClientRect();
            const direction = slider.classList.contains('vertical') ? 'vertical' : 'horizontal';

            let percentage;

            if (direction === 'horizontal') {
                const x = (e.touches ? e.touches[0].clientX : e.clientX) - rect.left;
                percentage = (x / rect.width) * 100;
                percentage = Math.max(0, Math.min(100, percentage));

                handle.style.left = percentage + '%';
                afterWrapper.style.clipPath = `inset(0 ${100 - percentage}% 0 0)`;
            } else {
                const y = (e.touches ? e.touches[0].clientY : e.clientY) - rect.top;
                percentage = (y / rect.height) * 100;
                percentage = Math.max(0, Math.min(100, percentage));

                handle.style.top = percentage + '%';
                afterWrapper.style.clipPath = `inset(${percentage}% 0 0 0)`;
            }

            this.sliderPosition = percentage;
        };

        // Mouse events
        handle.addEventListener('mousedown', () => {
            this.isDragging = true;
        });

        document.addEventListener('mousemove', updateSlider);

        document.addEventListener('mouseup', () => {
            this.isDragging = false;
        });

        // Touch events
        handle.addEventListener('touchstart', (e) => {
            this.isDragging = true;
            e.preventDefault();
        });

        document.addEventListener('touchmove', (e) => {
            if (this.isDragging) {
                updateSlider(e);
                e.preventDefault();
            }
        });

        document.addEventListener('touchend', () => {
            this.isDragging = false;
        });

        // Keyboard navigation
        document.addEventListener('keydown', (e) => {
            if (this.currentMode !== 'slider') return;

            const direction = slider.classList.contains('vertical') ? 'vertical' : 'horizontal';
            const step = 5;

            if ((direction === 'horizontal' && e.key === 'ArrowLeft') ||
                (direction === 'vertical' && e.key === 'ArrowUp')) {
                this.sliderPosition = Math.max(0, this.sliderPosition - step);
            } else if ((direction === 'horizontal' && e.key === 'ArrowRight') ||
                       (direction === 'vertical' && e.key === 'ArrowDown')) {
                this.sliderPosition = Math.min(100, this.sliderPosition + step);
            } else {
                return;
            }

            if (direction === 'horizontal') {
                handle.style.left = this.sliderPosition + '%';
                afterWrapper.style.clipPath = `inset(0 ${100 - this.sliderPosition}% 0 0)`;
            } else {
                handle.style.top = this.sliderPosition + '%';
                afterWrapper.style.clipPath = `inset(${this.sliderPosition}% 0 0 0)`;
            }
        });
    }

    switchMode(mode) {
        // Hide all modes
        document.querySelectorAll('.comparison-mode').forEach(el => {
            el.classList.remove('active');
        });

        // Show selected mode
        const modeMap = {
            'slider': 'sliderMode',
            'sidebyside': 'sideBySideMode',
            'fade': 'fadeMode',
            'flip': 'flipMode'
        };

        document.getElementById(modeMap[mode])?.classList.add('active');
        this.currentMode = mode;
    }

    setSliderDirection(direction) {
        const slider = document.querySelector('.image-compare-slider');
        const handle = slider.querySelector('.slider-handle');
        const afterWrapper = slider.querySelector('.after-wrapper');

        if (direction === 'vertical') {
            slider.classList.add('vertical');
            slider.classList.remove('horizontal');
            handle.style.left = '0';
            handle.style.top = '50%';
            afterWrapper.style.clipPath = 'inset(50% 0 0 0)';
        } else {
            slider.classList.add('horizontal');
            slider.classList.remove('vertical');
            handle.style.top = '0';
            handle.style.left = '50%';
            afterWrapper.style.clipPath = 'inset(0 50% 0 0)';
        }

        this.sliderPosition = 50;
    }

    toggleAutoPlay() {
        const btn = document.getElementById('autoPlayBtn');

        if (this.autoPlayInterval) {
            clearInterval(this.autoPlayInterval);
            this.autoPlayInterval = null;
            btn.textContent = '▶️ Auto Play';
            btn.classList.remove('active');
        } else {
            btn.textContent = '⏸️ Pause';
            btn.classList.add('active');

            this.autoPlayInterval = setInterval(() => {
                if (this.currentMode === 'slider') {
                    this.animateSlider();
                } else if (this.currentMode === 'fade') {
                    this.animateFade();
                }
            }, 3000);
        }
    }

    animateSlider() {
        const slider = document.querySelector('.image-compare-slider');
        const handle = slider.querySelector('.slider-handle');
        const afterWrapper = slider.querySelector('.after-wrapper');
        const direction = slider.classList.contains('vertical') ? 'vertical' : 'horizontal';

        const targetPosition = this.sliderPosition === 0 ? 100 : 0;

        gsap.to(this, {
            sliderPosition: targetPosition,
            duration: 2,
            ease: 'power2.inOut',
            onUpdate: () => {
                if (direction === 'horizontal') {
                    handle.style.left = this.sliderPosition + '%';
                    afterWrapper.style.clipPath = `inset(0 ${100 - this.sliderPosition}% 0 0)`;
                } else {
                    handle.style.top = this.sliderPosition + '%';
                    afterWrapper.style.clipPath = `inset(${this.sliderPosition}% 0 0 0)`;
                }
            }
        });
    }

    animateFade() {
        const afterImage = document.querySelector('.fade-after');
        const currentOpacity = parseFloat(afterImage.style.opacity || 0.5);
        const targetOpacity = currentOpacity === 0 ? 1 : 0;

        gsap.to(afterImage, {
            opacity: targetOpacity,
            duration: 2,
            ease: 'power2.inOut'
        });
    }

    updateFadeOpacity(value) {
        const afterImage = document.querySelector('.fade-after');
        const opacity = value / 100;
        afterImage.style.opacity = opacity;
        document.getElementById('fadeValue').textContent = value + '%';
    }

    async downloadComparison() {
        const btn = document.getElementById('downloadBtn');
        btn.disabled = true;
        btn.textContent = '⏳ Generating...';

        try {
            const container = document.querySelector('.comparison-container');
            const canvas = await html2canvas(container, {
                scale: 2,
                useCORS: true,
                backgroundColor: '#ffffff'
            });

            canvas.toBlob((blob) => {
                saveAs(blob, `room-comparison-${Date.now()}.png`);
                btn.disabled = false;
                btn.textContent = '💾 Download';
            });
        } catch (error) {
            console.error('Download failed:', error);
            btn.disabled = false;
            btn.textContent = '💾 Download';
        }
    }

    switchGalleryImage(thumbnail) {
        // Remove active class from all thumbnails
        document.querySelectorAll('.thumbnail-item').forEach(item => {
            item.classList.remove('active');
        });

        // Add active class to clicked thumbnail
        thumbnail.classList.add('active');

        // Update comparison images (would load different images here)
        console.log('Switched to:', thumbnail.querySelector('span').textContent);
    }
}

// Initialize the comparison tool
const tool = new ComparisonTool();
```

## Expected Outputs

1. **Interactive Slider**: Smooth before/after comparison
2. **Multiple Modes**: 4 different comparison views
3. **Animations**: Auto-play transitions
4. **Downloads**: High-quality comparison images
5. **Responsive Design**: Works on all devices
6. **Gallery**: Multiple view angles

## Bonus Challenges

1. **Video Export**: Generate MP4 comparison videos
2. **Annotations**: Add arrows and labels to highlight changes
3. **Timeline**: Show transformation progression
4. **Social Sharing**: One-click sharing to social media
5. **Measurement Tools**: Show dimension changes
6. **Cost Calculator**: Display renovation costs
7. **Print Layout**: PDF generation for presentations

## Resources

- [GSAP Animation](https://greensock.com/gsap/)
- [html2canvas](https://html2canvas.hertzen.com/)
- [CSS Clip Path](https://developer.mozilla.org/en-US/docs/Web/CSS/clip-path)
- [Before/After Patterns](https://www.smashingmagazine.com/2021/12/design-better-image-comparison-slider/)

## Success Criteria

- [ ] Slider moves smoothly in both directions
- [ ] All 4 comparison modes work correctly
- [ ] Keyboard navigation functional
- [ ] Touch support on mobile devices
- [ ] Auto-play transitions smoothly
- [ ] Downloads produce high-quality images
- [ ] Responsive on all screen sizes
- [ ] Gallery switches views correctly
