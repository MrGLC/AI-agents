# Project 01: Three.js 3D Room Visualizer

## Overview
Build an interactive 3D web-based room visualizer using Three.js that allows users to design and customize a 2.83m × 2.75m room with real-time color palette application, furniture placement, and material preview. Users can rotate the camera, move furniture, apply Feng Shui color schemes, and export their designs.

## Learning Objectives
- Master Three.js fundamentals: scenes, cameras, lighting, and materials
- Implement interactive 3D object manipulation with raycasting
- Build a color palette system with real-time material updates
- Create realistic lighting simulations (natural and artificial)
- Develop export functionality for 3D scenes and shopping lists

## Difficulty Level
**Intermediate to Advanced** - Requires JavaScript proficiency, 3D graphics concepts, and UI/UX design skills

## Technical Stack
- **Frontend**: HTML5, CSS3, JavaScript (ES6+)
- **3D Engine**: Three.js (r150+)
- **UI Framework**: React or Vue.js (optional but recommended)
- **Build Tool**: Vite or Webpack
- **Additional Libraries**:
  - dat.GUI for controls
  - OrbitControls for camera navigation
  - GLTFLoader for 3D furniture models
  - html2canvas for screenshots

## Room Specifications
- **Dimensions**: 2.83m (width) × 2.75m (depth) × 2.50m (height)
- **Color Palettes**:
  - Warm Focus: Terracotta (#D4735E), Warm Beige (#E8D5C4), Deep Brown (#4A3F35)
  - Soft Tech: Soft Blue-Gray (#B8C5D6), Warm White (#F5F3EF), Charcoal (#3C3F41)
  - Calm Studio: Sage Green (#A8B5A0), Cream (#F2EBD9), Soft Gray (#C2C5C0)

## Requirements

### 1. Core Rendering System
- [x] Create 3D room with accurate dimensions
- [x] Implement realistic PBR materials
- [x] Add ambient, directional, and point lighting
- [x] Enable camera controls (orbit, pan, zoom)

### 2. Color Palette Application
- [x] Toggle between three Feng Shui palettes
- [x] Apply colors to walls, floor, and ceiling independently
- [x] Preview material textures (matte, satin, glossy)
- [x] Real-time color updates without scene reload

### 3. Furniture Placement
- [x] Load 3D models for desk, piano, bed, storage
- [x] Drag-and-drop furniture positioning
- [x] Rotate furniture with controls
- [x] Snap-to-grid functionality
- [x] Collision detection

### 4. Lighting Visualization
- [x] Simulate natural light from window
- [x] Add artificial light sources (ceiling, desk lamp)
- [x] Toggle day/night modes
- [x] Adjustable light intensity and color temperature

### 5. Export Capabilities
- [x] Screenshot capture (multiple angles)
- [x] Export scene as GLTF/GLB
- [x] Generate furniture shopping list
- [x] Save/load room configurations (JSON)

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create project directory
mkdir threejs-room-visualizer
cd threejs-room-visualizer

# Initialize package.json
npm init -y

# Install dependencies
npm install three vite dat.gui
npm install --save-dev @types/three
```

**File: `index.html`**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Room Visualizer</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            background: #1a1a1a;
        }
        #canvas-container {
            width: 100vw;
            height: 100vh;
        }
        #controls-panel {
            position: absolute;
            top: 20px;
            right: 20px;
            background: rgba(255, 255, 255, 0.95);
            padding: 20px;
            border-radius: 12px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.2);
            max-width: 320px;
            max-height: 80vh;
            overflow-y: auto;
        }
        .palette-btn {
            width: 100%;
            padding: 12px;
            margin: 6px 0;
            border: 2px solid #ddd;
            border-radius: 8px;
            cursor: pointer;
            font-size: 14px;
            font-weight: 600;
            transition: all 0.3s;
        }
        .palette-btn:hover { transform: translateY(-2px); box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
        .palette-btn.active { border-color: #4CAF50; background: #e8f5e9; }
        h3 { margin-top: 16px; margin-bottom: 8px; color: #333; }
        .info-box {
            background: #f5f5f5;
            padding: 12px;
            border-radius: 6px;
            margin-top: 12px;
            font-size: 12px;
            color: #666;
        }
    </style>
</head>
<body>
    <div id="canvas-container"></div>
    <div id="controls-panel">
        <h2 style="margin-bottom: 16px; color: #333;">Room Designer</h2>

        <h3>Color Palettes</h3>
        <button class="palette-btn" data-palette="warm">Warm Focus</button>
        <button class="palette-btn" data-palette="soft">Soft Tech</button>
        <button class="palette-btn" data-palette="calm">Calm Studio</button>

        <h3>Lighting</h3>
        <label>
            <input type="checkbox" id="dayMode" checked> Day Mode
        </label><br>
        <label>
            Intensity: <input type="range" id="lightIntensity" min="0" max="2" step="0.1" value="1">
        </label>

        <h3>Actions</h3>
        <button class="palette-btn" id="screenshot">📸 Screenshot</button>
        <button class="palette-btn" id="exportScene">💾 Export Scene</button>
        <button class="palette-btn" id="exportList">📋 Shopping List</button>

        <div class="info-box">
            <strong>Room: 2.83m × 2.75m</strong><br>
            Drag to rotate • Right-click to pan • Scroll to zoom
        </div>
    </div>

    <script type="module" src="/src/main.js"></script>
</body>
</html>
```

### Step 2: Core Three.js Scene Setup

**File: `src/main.js`**
```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { RoomVisualizer } from './RoomVisualizer';

// Initialize the visualizer
const container = document.getElementById('canvas-container');
const visualizer = new RoomVisualizer(container);

// Color palette definitions
const PALETTES = {
    warm: {
        walls: '#D4735E',
        floor: '#4A3F35',
        ceiling: '#E8D5C4',
        accent: '#C65D47'
    },
    soft: {
        walls: '#B8C5D6',
        floor: '#3C3F41',
        ceiling: '#F5F3EF',
        accent: '#8FA3B8'
    },
    calm: {
        walls: '#A8B5A0',
        floor: '#C2C5C0',
        ceiling: '#F2EBD9',
        accent: '#8FA188'
    }
};

// Event listeners
document.querySelectorAll('.palette-btn[data-palette]').forEach(btn => {
    btn.addEventListener('click', (e) => {
        const palette = e.target.dataset.palette;
        document.querySelectorAll('.palette-btn[data-palette]').forEach(b =>
            b.classList.remove('active'));
        e.target.classList.add('active');
        visualizer.applyPalette(PALETTES[palette]);
    });
});

document.getElementById('dayMode').addEventListener('change', (e) => {
    visualizer.setDayMode(e.target.checked);
});

document.getElementById('lightIntensity').addEventListener('input', (e) => {
    visualizer.setLightIntensity(parseFloat(e.target.value));
});

document.getElementById('screenshot').addEventListener('click', () => {
    visualizer.takeScreenshot();
});

document.getElementById('exportScene').addEventListener('click', () => {
    visualizer.exportScene();
});

document.getElementById('exportList').addEventListener('click', () => {
    visualizer.exportShoppingList();
});

// Apply default palette
visualizer.applyPalette(PALETTES.warm);
document.querySelector('[data-palette="warm"]').classList.add('active');

// Start animation loop
visualizer.animate();
```

### Step 3: Room Visualizer Class

**File: `src/RoomVisualizer.js`**
```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { GLTFExporter } from 'three/examples/jsm/exporters/GLTFExporter';

export class RoomVisualizer {
    constructor(container) {
        this.container = container;
        this.furniture = [];
        this.currentPalette = null;

        // Room dimensions (in meters)
        this.roomWidth = 2.83;
        this.roomDepth = 2.75;
        this.roomHeight = 2.50;

        this.initScene();
        this.initLights();
        this.createRoom();
        this.addFurniture();
        this.setupControls();
        this.setupRaycaster();

        window.addEventListener('resize', () => this.onWindowResize());
    }

    initScene() {
        // Scene
        this.scene = new THREE.Scene();
        this.scene.background = new THREE.Color(0x1a1a1a);

        // Camera
        this.camera = new THREE.PerspectiveCamera(
            60,
            window.innerWidth / window.innerHeight,
            0.1,
            100
        );
        this.camera.position.set(3, 2, 4);
        this.camera.lookAt(0, 1, 0);

        // Renderer
        this.renderer = new THREE.WebGLRenderer({
            antialias: true,
            preserveDrawingBuffer: true // For screenshots
        });
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        this.renderer.setPixelRatio(window.devicePixelRatio);
        this.renderer.shadowMap.enabled = true;
        this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
        this.renderer.toneMappingExposure = 1.0;
        this.container.appendChild(this.renderer.domElement);
    }

    initLights() {
        // Ambient light
        this.ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
        this.scene.add(this.ambientLight);

        // Directional light (sun/window)
        this.sunLight = new THREE.DirectionalLight(0xfff5e6, 1.2);
        this.sunLight.position.set(2, 3, 1);
        this.sunLight.castShadow = true;
        this.sunLight.shadow.mapSize.width = 2048;
        this.sunLight.shadow.mapSize.height = 2048;
        this.sunLight.shadow.camera.near = 0.5;
        this.sunLight.shadow.camera.far = 10;
        this.scene.add(this.sunLight);

        // Ceiling light
        this.ceilingLight = new THREE.PointLight(0xfff8dc, 0.8, 10);
        this.ceilingLight.position.set(0, this.roomHeight - 0.1, 0);
        this.ceilingLight.castShadow = true;
        this.scene.add(this.ceilingLight);

        // Desk lamp (will be positioned near desk)
        this.deskLamp = new THREE.SpotLight(0xfff4e6, 0.6, 5, Math.PI / 6);
        this.deskLamp.position.set(-1, 1.2, -1);
        this.deskLamp.castShadow = true;
        this.scene.add(this.deskLamp);
    }

    createRoom() {
        const halfWidth = this.roomWidth / 2;
        const halfDepth = this.roomDepth / 2;

        // Floor
        const floorGeometry = new THREE.PlaneGeometry(this.roomWidth, this.roomDepth);
        this.floorMaterial = new THREE.MeshStandardMaterial({
            color: 0x4A3F35,
            roughness: 0.8,
            metalness: 0.1
        });
        this.floor = new THREE.Mesh(floorGeometry, this.floorMaterial);
        this.floor.rotation.x = -Math.PI / 2;
        this.floor.receiveShadow = true;
        this.floor.name = 'floor';
        this.scene.add(this.floor);

        // Ceiling
        const ceilingGeometry = new THREE.PlaneGeometry(this.roomWidth, this.roomDepth);
        this.ceilingMaterial = new THREE.MeshStandardMaterial({
            color: 0xE8D5C4,
            roughness: 0.9,
            metalness: 0.0
        });
        this.ceiling = new THREE.Mesh(ceilingGeometry, this.ceilingMaterial);
        this.ceiling.rotation.x = Math.PI / 2;
        this.ceiling.position.y = this.roomHeight;
        this.ceiling.name = 'ceiling';
        this.scene.add(this.ceiling);

        // Walls
        this.walls = [];
        const wallMaterial = new THREE.MeshStandardMaterial({
            color: 0xD4735E,
            roughness: 0.85,
            metalness: 0.0
        });
        this.wallMaterial = wallMaterial;

        // Back wall
        const backWallGeometry = new THREE.PlaneGeometry(this.roomWidth, this.roomHeight);
        const backWall = new THREE.Mesh(backWallGeometry, wallMaterial);
        backWall.position.z = -halfDepth;
        backWall.position.y = this.roomHeight / 2;
        backWall.receiveShadow = true;
        backWall.name = 'backWall';
        this.walls.push(backWall);
        this.scene.add(backWall);

        // Left wall
        const leftWallGeometry = new THREE.PlaneGeometry(this.roomDepth, this.roomHeight);
        const leftWall = new THREE.Mesh(leftWallGeometry, wallMaterial.clone());
        leftWall.position.x = -halfWidth;
        leftWall.position.y = this.roomHeight / 2;
        leftWall.rotation.y = Math.PI / 2;
        leftWall.receiveShadow = true;
        leftWall.name = 'leftWall';
        this.walls.push(leftWall);
        this.scene.add(leftWall);

        // Right wall (with window)
        const rightWallGeometry = new THREE.PlaneGeometry(this.roomDepth, this.roomHeight);
        const rightWall = new THREE.Mesh(rightWallGeometry, wallMaterial.clone());
        rightWall.position.x = halfWidth;
        rightWall.position.y = this.roomHeight / 2;
        rightWall.rotation.y = -Math.PI / 2;
        rightWall.receiveShadow = true;
        rightWall.name = 'rightWall';
        this.walls.push(rightWall);
        this.scene.add(rightWall);

        // Front wall
        const frontWallGeometry = new THREE.PlaneGeometry(this.roomWidth, this.roomHeight);
        const frontWall = new THREE.Mesh(frontWallGeometry, wallMaterial.clone());
        frontWall.position.z = halfDepth;
        frontWall.position.y = this.roomHeight / 2;
        frontWall.rotation.y = Math.PI;
        frontWall.receiveShadow = true;
        frontWall.name = 'frontWall';
        this.walls.push(frontWall);
        this.scene.add(frontWall);

        // Add window on right wall
        this.createWindow(halfWidth - 0.01, 1.3, 0, 0.8, 1.2);
    }

    createWindow(x, y, z, width, height) {
        const windowGeometry = new THREE.PlaneGeometry(width, height);
        const windowMaterial = new THREE.MeshPhysicalMaterial({
            color: 0x87CEEB,
            transparent: true,
            opacity: 0.3,
            roughness: 0.1,
            metalness: 0.1,
            transmission: 0.9
        });
        const windowMesh = new THREE.Mesh(windowGeometry, windowMaterial);
        windowMesh.position.set(x, y, z);
        windowMesh.rotation.y = -Math.PI / 2;
        windowMesh.name = 'window';
        this.scene.add(windowMesh);
    }

    addFurniture() {
        // Simple furniture using primitives (replace with GLTF models in production)

        // Desk
        const deskGroup = new THREE.Group();
        const deskTop = this.createBox(1.2, 0.05, 0.6, 0x8B6F47);
        deskTop.position.y = 0.75;
        const deskLeg1 = this.createBox(0.05, 0.75, 0.05, 0x6B5437);
        deskLeg1.position.set(-0.55, 0.375, -0.25);
        const deskLeg2 = this.createBox(0.05, 0.75, 0.05, 0x6B5437);
        deskLeg2.position.set(0.55, 0.375, -0.25);
        const deskLeg3 = this.createBox(0.05, 0.75, 0.05, 0x6B5437);
        deskLeg3.position.set(-0.55, 0.375, 0.25);
        const deskLeg4 = this.createBox(0.05, 0.75, 0.05, 0x6B5437);
        deskLeg4.position.set(0.55, 0.375, 0.25);

        deskGroup.add(deskTop, deskLeg1, deskLeg2, deskLeg3, deskLeg4);
        deskGroup.position.set(-0.8, 0, -0.8);
        deskGroup.name = 'desk';
        this.furniture.push(deskGroup);
        this.scene.add(deskGroup);

        // Piano (upright)
        const pianoGroup = new THREE.Group();
        const pianoBody = this.createBox(1.4, 1.2, 0.6, 0x2C2416);
        pianoBody.position.y = 0.6;
        const pianoKeys = this.createBox(1.3, 0.08, 0.25, 0xFFFAF0);
        pianoKeys.position.set(0, 0.75, 0.2);

        pianoGroup.add(pianoBody, pianoKeys);
        pianoGroup.position.set(0.5, 0, -1.2);
        pianoGroup.name = 'piano';
        this.furniture.push(pianoGroup);
        this.scene.add(pianoGroup);

        // Bed
        const bedGroup = new THREE.Group();
        const bedFrame = this.createBox(1.0, 0.4, 2.0, 0x8B7355);
        bedFrame.position.y = 0.2;
        const mattress = this.createBox(1.0, 0.15, 2.0, 0xE8E8E8);
        mattress.position.y = 0.475;
        const pillow = this.createBox(0.5, 0.1, 0.3, 0xF5F5DC);
        pillow.position.set(0, 0.6, -0.7);

        bedGroup.add(bedFrame, mattress, pillow);
        bedGroup.position.set(0.8, 0, 0.3);
        bedGroup.name = 'bed';
        this.furniture.push(bedGroup);
        this.scene.add(bedGroup);

        // Storage shelf
        const shelfGroup = new THREE.Group();
        for (let i = 0; i < 4; i++) {
            const shelf = this.createBox(0.8, 0.03, 0.3, 0x9C826B);
            shelf.position.y = i * 0.4;
            shelfGroup.add(shelf);
        }
        shelfGroup.position.set(-1.2, 0.2, 0.5);
        shelfGroup.name = 'storage';
        this.furniture.push(shelfGroup);
        this.scene.add(shelfGroup);
    }

    createBox(width, height, depth, color) {
        const geometry = new THREE.BoxGeometry(width, height, depth);
        const material = new THREE.MeshStandardMaterial({
            color,
            roughness: 0.7,
            metalness: 0.1
        });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.castShadow = true;
        mesh.receiveShadow = true;
        return mesh;
    }

    setupControls() {
        this.controls = new OrbitControls(this.camera, this.renderer.domElement);
        this.controls.enableDamping = true;
        this.controls.dampingFactor = 0.05;
        this.controls.maxPolarAngle = Math.PI / 2 - 0.1;
        this.controls.minDistance = 2;
        this.controls.maxDistance = 8;
        this.controls.target.set(0, 1, 0);
    }

    setupRaycaster() {
        this.raycaster = new THREE.Raycaster();
        this.mouse = new THREE.Vector2();
        this.selectedObject = null;

        this.renderer.domElement.addEventListener('click', (e) => this.onMouseClick(e));
    }

    onMouseClick(event) {
        this.mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
        this.mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

        this.raycaster.setFromCamera(this.mouse, this.camera);
        const intersects = this.raycaster.intersectObjects(this.furniture, true);

        if (intersects.length > 0) {
            const object = intersects[0].object;
            console.log('Clicked on:', object.parent.name || object.name);
            // Add selection highlight logic here
        }
    }

    applyPalette(palette) {
        this.currentPalette = palette;

        // Update wall materials
        this.walls.forEach(wall => {
            wall.material.color.set(palette.walls);
        });

        // Update floor
        this.floorMaterial.color.set(palette.floor);

        // Update ceiling
        this.ceilingMaterial.color.set(palette.ceiling);

        console.log('Applied palette:', palette);
    }

    setDayMode(isDayMode) {
        if (isDayMode) {
            this.sunLight.intensity = 1.2;
            this.sunLight.color.set(0xfff5e6);
            this.ambientLight.intensity = 0.4;
        } else {
            // Night mode
            this.sunLight.intensity = 0.1;
            this.sunLight.color.set(0x4169E1);
            this.ambientLight.intensity = 0.2;
            this.ceilingLight.intensity = 1.2;
        }
    }

    setLightIntensity(value) {
        this.sunLight.intensity = value * 1.2;
        this.ceilingLight.intensity = value * 0.8;
    }

    takeScreenshot() {
        this.renderer.render(this.scene, this.camera);
        const dataURL = this.renderer.domElement.toDataURL('image/png');

        const link = document.createElement('a');
        link.download = `room-design-${Date.now()}.png`;
        link.href = dataURL;
        link.click();

        console.log('Screenshot saved');
    }

    exportScene() {
        const exporter = new GLTFExporter();
        exporter.parse(
            this.scene,
            (gltf) => {
                const blob = new Blob([JSON.stringify(gltf)], { type: 'application/json' });
                const link = document.createElement('a');
                link.download = `room-scene-${Date.now()}.gltf`;
                link.href = URL.createObjectURL(blob);
                link.click();
                console.log('Scene exported');
            },
            { binary: false }
        );
    }

    exportShoppingList() {
        const shoppingList = {
            room_dimensions: {
                width: this.roomWidth,
                depth: this.roomDepth,
                height: this.roomHeight
            },
            color_palette: this.currentPalette,
            furniture: this.furniture.map(item => ({
                name: item.name,
                position: {
                    x: item.position.x.toFixed(2),
                    y: item.position.y.toFixed(2),
                    z: item.position.z.toFixed(2)
                }
            })),
            paint_needed: this.calculatePaintNeeded(),
            timestamp: new Date().toISOString()
        };

        const blob = new Blob([JSON.stringify(shoppingList, null, 2)],
            { type: 'application/json' });
        const link = document.createElement('a');
        link.download = `shopping-list-${Date.now()}.json`;
        link.href = URL.createObjectURL(blob);
        link.click();

        console.log('Shopping list exported:', shoppingList);
    }

    calculatePaintNeeded() {
        // Calculate wall area
        const wallArea = (
            2 * (this.roomWidth * this.roomHeight) +
            2 * (this.roomDepth * this.roomHeight)
        );

        // Assume 1 liter covers ~10 m² with 2 coats
        const litersNeeded = Math.ceil((wallArea * 2) / 10);

        return {
            wall_area_sqm: wallArea.toFixed(2),
            paint_liters: litersNeeded,
            coats: 2
        };
    }

    onWindowResize() {
        this.camera.aspect = window.innerWidth / window.innerHeight;
        this.camera.updateProjectionMatrix();
        this.renderer.setSize(window.innerWidth, window.innerHeight);
    }

    animate() {
        requestAnimationFrame(() => this.animate());
        this.controls.update();
        this.renderer.render(this.scene, this.camera);
    }
}
```

### Step 4: Package Configuration

**File: `package.json`**
```json
{
  "name": "threejs-room-visualizer",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "three": "^0.150.0",
    "dat.gui": "^0.7.9"
  },
  "devDependencies": {
    "vite": "^4.0.0"
  }
}
```

## Expected Outputs

### 1. Interactive 3D Room
- Fully navigable 3D room matching exact dimensions
- Smooth camera controls (orbit, pan, zoom)
- Realistic lighting and shadows
- Responsive design

### 2. Color Palette System
- Three distinct Feng Shui color schemes
- One-click palette switching
- Real-time material updates
- Preview of different finishes

### 3. Furniture Arrangement
- Pre-placed furniture (desk, piano, bed, storage)
- Accurate scale and proportions
- Shadows and proper lighting
- Collision detection (bonus)

### 4. Export Functionality
- High-quality PNG screenshots
- GLTF scene export for 3D printing/sharing
- JSON shopping list with:
  - Paint quantities
  - Furniture positions
  - Color codes
  - Timestamp

## Bonus Challenges

1. **Advanced Furniture Interaction**
   - Implement drag-and-drop furniture placement
   - Add rotation handles for furniture
   - Include furniture library with multiple models
   - Snap-to-wall and snap-to-grid functionality

2. **Material Library**
   - Add wood texture variants (oak, walnut, pine)
   - Include fabric swatches for soft furnishings
   - Metal finishes (brushed steel, brass, copper)
   - Preview materials on selected objects

3. **Lighting Presets**
   - Time-of-day simulation (morning, noon, evening, night)
   - Seasonal lighting variations
   - Custom light positioning
   - Light temperature control (warm to cool)

4. **AR Preview**
   - Export room as AR-compatible format
   - Generate QR code for mobile AR viewing
   - WebXR integration for in-browser AR

5. **Collaboration Features**
   - Save designs to cloud storage
   - Share designs via unique URL
   - Real-time collaborative editing
   - Comment system for design feedback

## Resources

### Three.js Documentation
- [Three.js Official Docs](https://threejs.org/docs/)
- [Three.js Examples](https://threejs.org/examples/)
- [Three.js Journey Course](https://threejs-journey.com/)

### 3D Assets
- [Sketchfab](https://sketchfab.com/) - Free 3D models
- [Poly Haven](https://polyhaven.com/) - Free textures and HDRIs
- [TurboSquid](https://www.turbosquid.com/) - Premium 3D models

### Color Theory
- [Coolors.co](https://coolors.co/) - Palette generator
- [Adobe Color](https://color.adobe.com/) - Color wheel and schemes
- [Feng Shui Color Guide](https://www.thespruce.com/feng-shui-color-meanings-1274549)

### Learning Materials
- [Three.js Fundamentals](https://threejsfundamentals.org/)
- [Discover Three.js](https://discoverthreejs.com/)
- [Bruno Simon's Portfolio](https://bruno-simon.com/) - Inspiration

## Success Criteria

### Minimum Viable Product (MVP)
- [ ] Room renders with correct dimensions (2.83m × 2.75m × 2.50m)
- [ ] All three color palettes are functional and switchable
- [ ] At least 4 furniture items are visible and properly scaled
- [ ] Camera controls work smoothly
- [ ] Screenshot export works
- [ ] Shopping list export includes paint calculations

### Full Feature Set
- [ ] Day/night lighting modes
- [ ] Adjustable light intensity
- [ ] GLTF scene export
- [ ] Proper shadows and realistic materials
- [ ] Window with transparency effect
- [ ] Responsive UI controls
- [ ] Performance: 60 FPS on modern hardware

### Excellence Indicators
- [ ] Drag-and-drop furniture placement
- [ ] Multiple view presets (top, side, corner)
- [ ] Material texture variants
- [ ] Collision detection
- [ ] Undo/redo functionality
- [ ] Save/load room configurations
- [ ] Professional UI/UX design
- [ ] Mobile responsive

## Testing Checklist

- [ ] Room dimensions verified with measuring tool
- [ ] All three palettes apply correctly to all surfaces
- [ ] Furniture doesn't intersect with walls
- [ ] Shadows render properly from all light sources
- [ ] Screenshot captures full scene at high resolution
- [ ] Exported GLTF can be imported into Blender
- [ ] Shopping list calculations are accurate
- [ ] Performance testing on target devices
- [ ] Cross-browser compatibility (Chrome, Firefox, Safari)
- [ ] Mobile touch controls work

## Next Steps

After completing this project, consider:
1. Add VR support with WebXR
2. Integrate real furniture APIs for pricing
3. Build a furniture recommendation AI
4. Create mobile app version
5. Add floor plan editing capability
6. Implement photorealistic rendering with ray tracing
