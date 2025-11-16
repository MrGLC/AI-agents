# Project 02: 2D Canvas Room Planner with Color Palette Tester

## Overview
Create a top-down 2D room planning application using HTML5 Canvas that allows users to design a 2.83m × 2.75m room layout with drag-and-drop furniture placement, real-time color palette testing, and measurement tools. This project focuses on practical space planning with an intuitive interface.

## Learning Objectives
- Master HTML5 Canvas API for 2D rendering
- Implement drag-and-drop interactions with collision detection
- Build a responsive grid system with snap-to-grid functionality
- Create efficient redraw algorithms for canvas performance
- Develop export functionality for floor plans and shopping lists

## Difficulty Level
**Beginner to Intermediate** - Excellent introduction to Canvas API and UI interactions

## Technical Stack
- **Frontend**: HTML5, CSS3, Vanilla JavaScript (ES6+)
- **Canvas**: HTML5 Canvas 2D Context
- **UI Framework**: Optional - Bootstrap or Tailwind for controls
- **Export**: jsPDF for PDF generation, Canvas toBlob for images
- **Additional Libraries**:
  - Fabric.js (optional, for easier object manipulation)
  - Paper.js (optional, for vector graphics)
  - Pica.js for high-quality image resizing

## Room Specifications
- **Dimensions**: 2.83m (width) × 2.75m (depth)
- **Scale**: 100 pixels = 1 meter (adjustable zoom)
- **Grid Size**: 10cm increments (10 pixels at base scale)
- **Color Palettes**:
  - Warm Focus: Terracotta (#D4735E), Warm Beige (#E8D5C4), Deep Brown (#4A3F35)
  - Soft Tech: Soft Blue-Gray (#B8C5D6), Warm White (#F5F3EF), Charcoal (#3C3F41)
  - Calm Studio: Sage Green (#A8B5A0), Cream (#F2EBD9), Soft Gray (#C2C5C0)

## Requirements

### 1. Core Canvas Rendering
- [x] Draw room outline with accurate dimensions
- [x] Render measurement rulers and grid
- [x] Display furniture as 2D shapes (top-down view)
- [x] Apply color schemes to wall representations
- [x] Zoom and pan controls

### 2. Color Palette System
- [x] Visual palette swatches
- [x] Real-time color preview on room layout
- [x] Color contrast checker
- [x] Export color codes and paint quantities

### 3. Furniture Management
- [x] Draggable furniture objects
- [x] Furniture library (desk, piano, bed, storage)
- [x] Rotation controls (90° increments)
- [x] Dimension labels on furniture
- [x] Collision detection with walls and other furniture

### 4. Measurement Tools
- [x] Ruler along edges
- [x] Distance measurement tool
- [x] Area calculator
- [x] Clearance checker

### 5. Export Features
- [x] Export as PNG/SVG
- [x] Generate PDF floor plan
- [x] Create shopping list
- [x] Save/load project JSON

## Step-by-Step Implementation

### Step 1: HTML Structure

**File: `index.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>2D Room Planner</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
            background: #f5f5f5;
            display: flex;
            height: 100vh;
            overflow: hidden;
        }

        #sidebar {
            width: 300px;
            background: white;
            box-shadow: 2px 0 8px rgba(0,0,0,0.1);
            overflow-y: auto;
            padding: 20px;
        }

        #main-canvas-area {
            flex: 1;
            display: flex;
            flex-direction: column;
            background: #e8e8e8;
        }

        #toolbar {
            background: white;
            padding: 12px 20px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            display: flex;
            gap: 12px;
            align-items: center;
        }

        #canvas-container {
            flex: 1;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 20px;
            overflow: auto;
        }

        #room-canvas {
            background: white;
            box-shadow: 0 4px 16px rgba(0,0,0,0.15);
            cursor: crosshair;
        }

        .section {
            margin-bottom: 24px;
        }

        .section h3 {
            font-size: 14px;
            font-weight: 600;
            text-transform: uppercase;
            color: #555;
            margin-bottom: 12px;
            padding-bottom: 8px;
            border-bottom: 2px solid #e0e0e0;
        }

        .palette-option {
            padding: 12px;
            margin: 8px 0;
            border: 2px solid #ddd;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            align-items: center;
            gap: 10px;
        }

        .palette-option:hover {
            border-color: #999;
            transform: translateX(4px);
        }

        .palette-option.active {
            border-color: #4CAF50;
            background: #f1f8f4;
        }

        .color-swatch {
            width: 30px;
            height: 30px;
            border-radius: 4px;
            border: 1px solid rgba(0,0,0,0.1);
        }

        .furniture-item {
            padding: 10px;
            margin: 6px 0;
            background: #f9f9f9;
            border: 1px solid #ddd;
            border-radius: 6px;
            cursor: grab;
            transition: all 0.2s;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .furniture-item:hover {
            background: #f0f0f0;
            transform: translateX(4px);
        }

        .furniture-item:active {
            cursor: grabbing;
        }

        .btn {
            padding: 10px 16px;
            background: #2196F3;
            color: white;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-size: 14px;
            font-weight: 500;
            transition: all 0.2s;
        }

        .btn:hover {
            background: #1976D2;
            transform: translateY(-1px);
        }

        .btn-secondary {
            background: #757575;
        }

        .btn-secondary:hover {
            background: #616161;
        }

        .btn-success {
            background: #4CAF50;
        }

        .btn-success:hover {
            background: #45a049;
        }

        .tool-btn {
            padding: 8px 12px;
            background: white;
            border: 2px solid #ddd;
            border-radius: 6px;
            cursor: pointer;
            font-size: 13px;
            transition: all 0.2s;
        }

        .tool-btn:hover {
            border-color: #2196F3;
            color: #2196F3;
        }

        .tool-btn.active {
            background: #2196F3;
            color: white;
            border-color: #2196F3;
        }

        .dimensions-display {
            background: #f0f7ff;
            padding: 12px;
            border-radius: 6px;
            margin-top: 8px;
            font-size: 13px;
            color: #1565C0;
        }

        .info-box {
            background: #fff3cd;
            border-left: 4px solid #ffc107;
            padding: 12px;
            margin-top: 12px;
            border-radius: 4px;
            font-size: 12px;
            color: #856404;
        }
    </style>
</head>
<body>
    <div id="sidebar">
        <h2 style="margin-bottom: 20px;">Room Planner</h2>

        <div class="section">
            <h3>Color Palettes</h3>
            <div class="palette-option" data-palette="warm">
                <div style="display: flex; gap: 4px;">
                    <div class="color-swatch" style="background: #D4735E;"></div>
                    <div class="color-swatch" style="background: #E8D5C4;"></div>
                    <div class="color-swatch" style="background: #4A3F35;"></div>
                </div>
                <span>Warm Focus</span>
            </div>
            <div class="palette-option" data-palette="soft">
                <div style="display: flex; gap: 4px;">
                    <div class="color-swatch" style="background: #B8C5D6;"></div>
                    <div class="color-swatch" style="background: #F5F3EF;"></div>
                    <div class="color-swatch" style="background: #3C3F41;"></div>
                </div>
                <span>Soft Tech</span>
            </div>
            <div class="palette-option" data-palette="calm">
                <div style="display: flex; gap: 4px;">
                    <div class="color-swatch" style="background: #A8B5A0;"></div>
                    <div class="color-swatch" style="background: #F2EBD9;"></div>
                    <div class="color-swatch" style="background: #C2C5C0;"></div>
                </div>
                <span>Calm Studio</span>
            </div>
        </div>

        <div class="section">
            <h3>Furniture Library</h3>
            <div class="furniture-item" draggable="true" data-furniture="desk">
                <span>📚 Desk</span>
                <span style="font-size: 11px; color: #999;">120×60cm</span>
            </div>
            <div class="furniture-item" draggable="true" data-furniture="piano">
                <span>🎹 Piano</span>
                <span style="font-size: 11px; color: #999;">140×60cm</span>
            </div>
            <div class="furniture-item" draggable="true" data-furniture="bed">
                <span>🛏️ Bed</span>
                <span style="font-size: 11px; color: #999;">100×200cm</span>
            </div>
            <div class="furniture-item" draggable="true" data-furniture="storage">
                <span>📦 Storage</span>
                <span style="font-size: 11px; color: #999;">80×30cm</span>
            </div>
            <div class="furniture-item" draggable="true" data-furniture="chair">
                <span>🪑 Chair</span>
                <span style="font-size: 11px; color: #999;">50×50cm</span>
            </div>
        </div>

        <div class="section">
            <h3>Actions</h3>
            <button class="btn btn-success" style="width: 100%; margin-bottom: 8px;" id="exportPDF">
                📄 Export PDF
            </button>
            <button class="btn" style="width: 100%; margin-bottom: 8px;" id="exportPNG">
                🖼️ Export PNG
            </button>
            <button class="btn btn-secondary" style="width: 100%; margin-bottom: 8px;" id="exportList">
                📋 Shopping List
            </button>
            <button class="btn btn-secondary" style="width: 100%;" id="clearAll">
                🗑️ Clear All
            </button>
        </div>

        <div class="dimensions-display">
            <strong>Room Dimensions</strong><br>
            Width: 2.83m<br>
            Depth: 2.75m<br>
            Area: 7.78m²
        </div>

        <div class="info-box">
            💡 Drag furniture from the library onto the canvas. Click to select, drag to move, rotate with R key.
        </div>
    </div>

    <div id="main-canvas-area">
        <div id="toolbar">
            <button class="tool-btn active" data-tool="select">✋ Select</button>
            <button class="tool-btn" data-tool="measure">📏 Measure</button>
            <button class="tool-btn" data-tool="rotate">🔄 Rotate</button>
            <div style="flex: 1;"></div>
            <label>
                Zoom: <input type="range" id="zoom-slider" min="50" max="200" value="100" style="width: 120px;">
                <span id="zoom-value">100%</span>
            </label>
            <button class="tool-btn" id="reset-view">🎯 Reset View</button>
        </div>

        <div id="canvas-container">
            <canvas id="room-canvas" width="800" height="800"></canvas>
        </div>
    </div>

    <script src="app.js"></script>
</body>
</html>
```

### Step 2: Core Canvas Application

**File: `app.js`**
```javascript
class RoomPlanner {
    constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');

        // Room dimensions in meters
        this.roomWidth = 2.83;
        this.roomDepth = 2.75;

        // Scale: pixels per meter
        this.baseScale = 200;
        this.scale = this.baseScale;
        this.zoom = 1.0;

        // Canvas dimensions
        this.canvasWidth = 800;
        this.canvasHeight = 800;

        // Offset for centering
        this.offsetX = (this.canvasWidth - this.roomWidth * this.scale) / 2;
        this.offsetY = (this.canvasHeight - this.roomDepth * this.scale) / 2;

        // Current state
        this.currentPalette = null;
        this.furniture = [];
        this.selectedFurniture = null;
        this.isDragging = false;
        this.dragStartX = 0;
        this.dragStartY = 0;
        this.currentTool = 'select';

        // Furniture templates (dimensions in meters)
        this.furnitureTemplates = {
            desk: { width: 1.2, depth: 0.6, color: '#8B6F47', label: 'Desk' },
            piano: { width: 1.4, depth: 0.6, color: '#2C2416', label: 'Piano' },
            bed: { width: 1.0, depth: 2.0, color: '#A0826D', label: 'Bed' },
            storage: { width: 0.8, depth: 0.3, color: '#9C826B', label: 'Storage' },
            chair: { width: 0.5, depth: 0.5, color: '#654321', label: 'Chair' }
        };

        // Color palettes
        this.palettes = {
            warm: {
                wall: '#D4735E',
                floor: '#E8D5C4',
                accent: '#4A3F35',
                name: 'Warm Focus'
            },
            soft: {
                wall: '#B8C5D6',
                floor: '#F5F3EF',
                accent: '#3C3F41',
                name: 'Soft Tech'
            },
            calm: {
                wall: '#A8B5A0',
                floor: '#F2EBD9',
                accent: '#C2C5C0',
                name: 'Calm Studio'
            }
        };

        this.init();
    }

    init() {
        this.setupEventListeners();
        this.applyPalette('warm');
        this.draw();
    }

    setupEventListeners() {
        // Canvas interactions
        this.canvas.addEventListener('mousedown', (e) => this.onMouseDown(e));
        this.canvas.addEventListener('mousemove', (e) => this.onMouseMove(e));
        this.canvas.addEventListener('mouseup', (e) => this.onMouseUp(e));

        // Keyboard shortcuts
        document.addEventListener('keydown', (e) => {
            if (e.key === 'r' || e.key === 'R') {
                this.rotateSelected();
            } else if (e.key === 'Delete' || e.key === 'Backspace') {
                this.deleteSelected();
            }
        });

        // Palette selection
        document.querySelectorAll('.palette-option').forEach(option => {
            option.addEventListener('click', (e) => {
                const palette = e.currentTarget.dataset.palette;
                document.querySelectorAll('.palette-option').forEach(opt =>
                    opt.classList.remove('active'));
                e.currentTarget.classList.add('active');
                this.applyPalette(palette);
            });
        });

        // Furniture drag from library
        document.querySelectorAll('.furniture-item').forEach(item => {
            item.addEventListener('dragstart', (e) => {
                e.dataTransfer.setData('furniture', e.currentTarget.dataset.furniture);
            });
        });

        this.canvas.addEventListener('dragover', (e) => e.preventDefault());
        this.canvas.addEventListener('drop', (e) => {
            e.preventDefault();
            const furnitureType = e.dataTransfer.getData('furniture');
            this.addFurniture(furnitureType, e.offsetX, e.offsetY);
        });

        // Toolbar
        document.querySelectorAll('.tool-btn[data-tool]').forEach(btn => {
            btn.addEventListener('click', (e) => {
                document.querySelectorAll('.tool-btn[data-tool]').forEach(b =>
                    b.classList.remove('active'));
                e.currentTarget.classList.add('active');
                this.currentTool = e.currentTarget.dataset.tool;
            });
        });

        // Zoom
        document.getElementById('zoom-slider').addEventListener('input', (e) => {
            this.zoom = e.target.value / 100;
            this.scale = this.baseScale * this.zoom;
            document.getElementById('zoom-value').textContent = e.target.value + '%';
            this.draw();
        });

        document.getElementById('reset-view').addEventListener('click', () => {
            this.zoom = 1.0;
            this.scale = this.baseScale;
            document.getElementById('zoom-slider').value = 100;
            document.getElementById('zoom-value').textContent = '100%';
            this.draw();
        });

        // Export buttons
        document.getElementById('exportPNG').addEventListener('click', () => this.exportPNG());
        document.getElementById('exportPDF').addEventListener('click', () => this.exportPDF());
        document.getElementById('exportList').addEventListener('click', () => this.exportShoppingList());
        document.getElementById('clearAll').addEventListener('click', () => this.clearAll());
    }

    applyPalette(paletteName) {
        this.currentPalette = this.palettes[paletteName];
        this.draw();
    }

    draw() {
        // Clear canvas
        this.ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight);

        // Draw grid
        this.drawGrid();

        // Draw room
        this.drawRoom();

        // Draw rulers
        this.drawRulers();

        // Draw furniture
        this.furniture.forEach(item => this.drawFurniture(item));

        // Draw selection
        if (this.selectedFurniture) {
            this.drawSelection(this.selectedFurniture);
        }
    }

    drawGrid() {
        this.ctx.strokeStyle = '#e0e0e0';
        this.ctx.lineWidth = 1;

        const gridSize = 0.1 * this.scale; // 10cm grid
        const roomWidthPx = this.roomWidth * this.scale;
        const roomDepthPx = this.roomDepth * this.scale;

        // Vertical lines
        for (let x = 0; x <= roomWidthPx; x += gridSize) {
            this.ctx.beginPath();
            this.ctx.moveTo(this.offsetX + x, this.offsetY);
            this.ctx.lineTo(this.offsetX + x, this.offsetY + roomDepthPx);
            this.ctx.stroke();
        }

        // Horizontal lines
        for (let y = 0; y <= roomDepthPx; y += gridSize) {
            this.ctx.beginPath();
            this.ctx.moveTo(this.offsetX, this.offsetY + y);
            this.ctx.lineTo(this.offsetX + roomWidthPx, this.offsetY + y);
            this.ctx.stroke();
        }
    }

    drawRoom() {
        const roomWidthPx = this.roomWidth * this.scale;
        const roomDepthPx = this.roomDepth * this.scale;

        // Floor
        this.ctx.fillStyle = this.currentPalette.floor;
        this.ctx.fillRect(this.offsetX, this.offsetY, roomWidthPx, roomDepthPx);

        // Walls (border)
        this.ctx.strokeStyle = this.currentPalette.wall;
        this.ctx.lineWidth = 8;
        this.ctx.strokeRect(this.offsetX, this.offsetY, roomWidthPx, roomDepthPx);

        // Wall indicators (small colored rectangles at edges)
        this.ctx.fillStyle = this.currentPalette.wall;
        this.ctx.globalAlpha = 0.3;

        // Top wall indicator
        this.ctx.fillRect(this.offsetX, this.offsetY - 20, roomWidthPx, 15);

        // Left wall indicator
        this.ctx.fillRect(this.offsetX - 20, this.offsetY, 15, roomDepthPx);

        this.ctx.globalAlpha = 1.0;
    }

    drawRulers() {
        this.ctx.fillStyle = '#333';
        this.ctx.font = '12px Arial';
        this.ctx.textAlign = 'center';

        const roomWidthPx = this.roomWidth * this.scale;
        const roomDepthPx = this.roomDepth * this.scale;

        // Top ruler (width)
        this.ctx.fillText(
            `${this.roomWidth}m`,
            this.offsetX + roomWidthPx / 2,
            this.offsetY - 30
        );

        // Left ruler (depth)
        this.ctx.save();
        this.ctx.translate(this.offsetX - 30, this.offsetY + roomDepthPx / 2);
        this.ctx.rotate(-Math.PI / 2);
        this.ctx.fillText(`${this.roomDepth}m`, 0, 0);
        this.ctx.restore();

        // Draw tick marks
        this.ctx.strokeStyle = '#666';
        this.ctx.lineWidth = 1;

        // Width ticks (every 0.5m)
        for (let x = 0; x <= this.roomWidth; x += 0.5) {
            const px = this.offsetX + x * this.scale;
            this.ctx.beginPath();
            this.ctx.moveTo(px, this.offsetY - 10);
            this.ctx.lineTo(px, this.offsetY - 5);
            this.ctx.stroke();
        }

        // Depth ticks
        for (let y = 0; y <= this.roomDepth; y += 0.5) {
            const py = this.offsetY + y * this.scale;
            this.ctx.beginPath();
            this.ctx.moveTo(this.offsetX - 10, py);
            this.ctx.lineTo(this.offsetX - 5, py);
            this.ctx.stroke();
        }
    }

    drawFurniture(item) {
        this.ctx.save();

        // Translate to furniture position
        this.ctx.translate(item.x, item.y);

        // Rotate if necessary
        if (item.rotation) {
            this.ctx.rotate(item.rotation * Math.PI / 180);
        }

        // Draw furniture rectangle
        const width = item.width * this.scale;
        const depth = item.depth * this.scale;

        this.ctx.fillStyle = item.color;
        this.ctx.fillRect(-width / 2, -depth / 2, width, depth);

        // Draw border
        this.ctx.strokeStyle = '#333';
        this.ctx.lineWidth = 2;
        this.ctx.strokeRect(-width / 2, -depth / 2, width, depth);

        // Draw label
        this.ctx.fillStyle = '#fff';
        this.ctx.font = 'bold 12px Arial';
        this.ctx.textAlign = 'center';
        this.ctx.textBaseline = 'middle';
        this.ctx.fillText(item.label, 0, 0);

        // Draw dimensions
        this.ctx.font = '10px Arial';
        this.ctx.fillStyle = '#000';
        this.ctx.fillText(
            `${(item.width * 100).toFixed(0)}×${(item.depth * 100).toFixed(0)}cm`,
            0,
            depth / 2 + 15
        );

        this.ctx.restore();
    }

    drawSelection(item) {
        this.ctx.save();
        this.ctx.translate(item.x, item.y);
        if (item.rotation) {
            this.ctx.rotate(item.rotation * Math.PI / 180);
        }

        const width = item.width * this.scale;
        const depth = item.depth * this.scale;

        // Selection outline
        this.ctx.strokeStyle = '#2196F3';
        this.ctx.lineWidth = 3;
        this.ctx.setLineDash([5, 5]);
        this.ctx.strokeRect(-width / 2 - 5, -depth / 2 - 5, width + 10, depth + 10);
        this.ctx.setLineDash([]);

        // Corner handles
        this.ctx.fillStyle = '#2196F3';
        const handleSize = 8;
        const corners = [
            [-width / 2, -depth / 2],
            [width / 2, -depth / 2],
            [width / 2, depth / 2],
            [-width / 2, depth / 2]
        ];

        corners.forEach(([cx, cy]) => {
            this.ctx.fillRect(cx - handleSize / 2, cy - handleSize / 2, handleSize, handleSize);
        });

        this.ctx.restore();
    }

    addFurniture(type, x, y) {
        const template = this.furnitureTemplates[type];
        if (!template) return;

        const furniture = {
            id: Date.now(),
            type: type,
            x: x,
            y: y,
            width: template.width,
            depth: template.depth,
            color: template.color,
            label: template.label,
            rotation: 0
        };

        this.furniture.push(furniture);
        this.selectedFurniture = furniture;
        this.draw();
    }

    onMouseDown(e) {
        const rect = this.canvas.getBoundingClientRect();
        const x = e.clientX - rect.left;
        const y = e.clientY - rect.top;

        // Check if clicking on furniture
        const clicked = this.getFurnitureAtPoint(x, y);

        if (clicked) {
            this.selectedFurniture = clicked;
            this.isDragging = true;
            this.dragStartX = x - clicked.x;
            this.dragStartY = y - clicked.y;
        } else {
            this.selectedFurniture = null;
        }

        this.draw();
    }

    onMouseMove(e) {
        if (!this.isDragging || !this.selectedFurniture) return;

        const rect = this.canvas.getBoundingClientRect();
        const x = e.clientX - rect.left;
        const y = e.clientY - rect.top;

        this.selectedFurniture.x = x - this.dragStartX;
        this.selectedFurniture.y = y - this.dragStartY;

        // Snap to grid (optional)
        const gridSize = 0.1 * this.scale;
        this.selectedFurniture.x = Math.round(this.selectedFurniture.x / gridSize) * gridSize;
        this.selectedFurniture.y = Math.round(this.selectedFurniture.y / gridSize) * gridSize;

        this.draw();
    }

    onMouseUp(e) {
        this.isDragging = false;
    }

    getFurnitureAtPoint(x, y) {
        // Check in reverse order (top to bottom)
        for (let i = this.furniture.length - 1; i >= 0; i--) {
            const item = this.furniture[i];
            const width = item.width * this.scale;
            const depth = item.depth * this.scale;

            // Simple bounding box check (doesn't account for rotation)
            if (
                x >= item.x - width / 2 &&
                x <= item.x + width / 2 &&
                y >= item.y - depth / 2 &&
                y <= item.y + depth / 2
            ) {
                return item;
            }
        }
        return null;
    }

    rotateSelected() {
        if (!this.selectedFurniture) return;
        this.selectedFurniture.rotation = (this.selectedFurniture.rotation + 90) % 360;
        this.draw();
    }

    deleteSelected() {
        if (!this.selectedFurniture) return;
        this.furniture = this.furniture.filter(f => f.id !== this.selectedFurniture.id);
        this.selectedFurniture = null;
        this.draw();
    }

    clearAll() {
        if (confirm('Clear all furniture? This cannot be undone.')) {
            this.furniture = [];
            this.selectedFurniture = null;
            this.draw();
        }
    }

    exportPNG() {
        const link = document.createElement('a');
        link.download = `room-plan-${Date.now()}.png`;
        link.href = this.canvas.toDataURL('image/png');
        link.click();
    }

    exportPDF() {
        // Simple implementation - in production, use jsPDF
        alert('PDF export would use jsPDF library. For now, use PNG export.');
    }

    exportShoppingList() {
        const wallArea = 2 * (this.roomWidth + this.roomDepth) * 2.5; // Assuming 2.5m height
        const paintLiters = Math.ceil((wallArea * 2) / 10); // 2 coats, 1L per 10m²

        const shoppingList = {
            room: {
                dimensions: `${this.roomWidth}m × ${this.roomDepth}m`,
                area: (this.roomWidth * this.roomDepth).toFixed(2) + 'm²'
            },
            palette: {
                name: this.currentPalette.name,
                colors: this.currentPalette
            },
            paint: {
                wall_color: this.currentPalette.wall,
                floor_color: this.currentPalette.floor,
                liters_needed: paintLiters
            },
            furniture: this.furniture.map(f => ({
                type: f.label,
                dimensions: `${(f.width * 100).toFixed(0)}cm × ${(f.depth * 100).toFixed(0)}cm`,
                position: `(${((f.x - this.offsetX) / this.scale).toFixed(2)}m, ${((f.y - this.offsetY) / this.scale).toFixed(2)}m)`,
                rotation: f.rotation + '°'
            }))
        };

        const blob = new Blob([JSON.stringify(shoppingList, null, 2)],
            { type: 'application/json' });
        const link = document.createElement('a');
        link.download = `shopping-list-${Date.now()}.json`;
        link.href = URL.createObjectURL(blob);
        link.click();

        console.log('Shopping list:', shoppingList);
    }
}

// Initialize the planner
const planner = new RoomPlanner('room-canvas');

// Set default palette
document.querySelector('[data-palette="warm"]').classList.add('active');
```

### Step 3: Advanced Features (Optional)

**File: `advanced-features.js`** (Add to app.js)
```javascript
// Collision detection
checkCollision(item1, item2) {
    // Simple AABB collision (doesn't handle rotation)
    const margin = 0.05; // 5cm clearance

    const item1Left = item1.x - (item1.width * this.scale) / 2 - margin * this.scale;
    const item1Right = item1.x + (item1.width * this.scale) / 2 + margin * this.scale;
    const item1Top = item1.y - (item1.depth * this.scale) / 2 - margin * this.scale;
    const item1Bottom = item1.y + (item1.depth * this.scale) / 2 + margin * this.scale;

    const item2Left = item2.x - (item2.width * this.scale) / 2;
    const item2Right = item2.x + (item2.width * this.scale) / 2;
    const item2Top = item2.y - (item2.depth * this.scale) / 2;
    const item2Bottom = item2.y + (item2.depth * this.scale) / 2;

    return !(
        item1Right < item2Left ||
        item1Left > item2Right ||
        item1Bottom < item2Top ||
        item1Top > item2Bottom
    );
}

// Wall collision detection
isInsideRoom(item) {
    const roomLeft = this.offsetX;
    const roomRight = this.offsetX + this.roomWidth * this.scale;
    const roomTop = this.offsetY;
    const roomBottom = this.offsetY + this.roomDepth * this.scale;

    const itemLeft = item.x - (item.width * this.scale) / 2;
    const itemRight = item.x + (item.width * this.scale) / 2;
    const itemTop = item.y - (item.depth * this.scale) / 2;
    const itemBottom = item.y + (item.depth * this.scale) / 2;

    return (
        itemLeft >= roomLeft &&
        itemRight <= roomRight &&
        itemTop >= roomTop &&
        itemBottom <= roomBottom
    );
}

// Measurement tool
drawMeasurement(x1, y1, x2, y2) {
    const distance = Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2) / this.scale;

    this.ctx.strokeStyle = '#FF5722';
    this.ctx.lineWidth = 2;
    this.ctx.setLineDash([5, 5]);

    this.ctx.beginPath();
    this.ctx.moveTo(x1, y1);
    this.ctx.lineTo(x2, y2);
    this.ctx.stroke();

    this.ctx.setLineDash([]);

    // Draw distance label
    const midX = (x1 + x2) / 2;
    const midY = (y1 + y2) / 2;

    this.ctx.fillStyle = '#FF5722';
    this.ctx.font = 'bold 14px Arial';
    this.ctx.textAlign = 'center';
    this.ctx.fillText(`${distance.toFixed(2)}m`, midX, midY - 10);
}
```

## Expected Outputs

1. **Interactive Floor Plan**: Top-down view of room with grid overlay
2. **Color Visualization**: Real-time preview of wall/floor colors
3. **Furniture Layout**: Draggable, rotatable furniture pieces
4. **Measurements**: Rulers, grid, and distance tools
5. **Exports**: PNG images, JSON shopping lists, PDF floor plans

## Bonus Challenges

1. Add undo/redo functionality
2. Implement furniture library with custom items
3. Add wall thickness visualization
4. Create multiple room templates
5. Add door and window placement
6. Implement print-to-scale functionality
7. Create 3D preview from 2D layout
8. Add furniture dimension customization

## Resources

- [Canvas API Tutorial](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial)
- [Fabric.js Documentation](http://fabricjs.com/)
- [jsPDF Library](https://github.com/parallax/jsPDF)
- [Color Theory for Interiors](https://www.canva.com/colors/color-theory/)

## Success Criteria

- [ ] Room renders at correct scale with measurements
- [ ] All three color palettes apply correctly
- [ ] Furniture can be dragged, rotated, and deleted
- [ ] Grid and snap-to-grid work properly
- [ ] PNG export captures full canvas
- [ ] Shopping list includes furniture and paint calculations
- [ ] No furniture overlap when placing items
- [ ] Responsive design works on different screen sizes
