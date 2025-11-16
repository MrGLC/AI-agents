# Project 08: VR-Ready Room Experience with WebXR

## Overview
Build an immersive virtual reality room experience using WebXR and A-Frame that allows users to explore and design their 2.83m × 2.75m room in VR. Users can walk through the space, interact with furniture, change colors in real-time, and experience the room design as if they were physically present.

## Learning Objectives
- Master WebXR API for VR development
- Build immersive 3D environments with A-Frame
- Implement VR controllers and hand tracking
- Create interactive VR UI systems
- Develop teleportation and locomotion mechanics
- Optimize performance for VR (90 FPS target)

## Difficulty Level
**Advanced** - Requires 3D graphics knowledge, VR development experience, and WebXR understanding

## Technical Stack
- **VR Framework**: A-Frame 1.4+, Three.js
- **WebXR**: WebXR Device API
- **VR Devices**:
  - Meta Quest 2/3
  - HTC Vive
  - Valve Index
  - Browser-based VR (desktop fallback)
- **Additional**:
  - A-Frame Extras (locomotion)
  - A-Frame Environment
  - Super Hands (interactions)
- **Build**: Webpack or Vite

## Room Specifications
- **Dimensions**: 2.83m × 2.75m × 2.50m (1:1 scale)
- **VR Scale**: Real-world measurements
- **Interaction**: Grab, move, and rotate furniture
- **Color Palettes**: Real-time switching in VR
- **Performance**: 90 FPS minimum for comfort

## Requirements

### 1. VR Scene Setup
- [x] Room with accurate dimensions
- [x] Realistic lighting and shadows
- [x] Environment mapping
- [x] Floor grid for reference
- [x] Optimized for VR performance

### 2. VR Controls
- [x] Controller support (both hands)
- [x] Hand tracking (Quest 2/3)
- [x] Teleportation locomotion
- [x] Smooth turning
- [x] Grab and throw physics

### 3. Interactive Furniture
- [x] Grabbable furniture pieces
- [x] Snap-to-grid placement
- [x] Rotation controls
- [x] Collision detection
- [x] Physics simulation

### 4. VR UI System
- [x] Floating menu panels
- [x] Color palette selector
- [x] Furniture spawner
- [x] Settings panel
- [x] Tooltips and labels

### 5. Immersive Features
- [x] Spatial audio
- [x] Haptic feedback
- [x] Eye-level view adjustment
- [x] Comfort settings (vignette, snap turn)
- [x] Performance monitoring

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create project directory
mkdir vr-room-experience
cd vr-room-experience

# Initialize npm
npm init -y

# Install A-Frame and dependencies
npm install aframe
npm install aframe-extras
npm install aframe-environment-component
npm install super-hands
npm install aframe-physics-system

# Install build tools
npm install --save-dev vite
```

**File: `package.json`**
```json
{
  "name": "vr-room-experience",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "aframe": "^1.4.2",
    "aframe-extras": "^7.1.0",
    "aframe-environment-component": "^1.3.3",
    "super-hands": "^3.0.3",
    "aframe-physics-system": "^4.0.1"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

### Step 2: Main HTML Structure

**File: `index.html`**
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>VR Room Experience</title>
    <meta name="description" content="Immersive VR room design and visualization">
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://cdn.jsdelivr.net/gh/c-frame/aframe-extras@7.1.0/dist/aframe-extras.min.js"></script>
    <script src="https://unpkg.com/super-hands@3.0.3/dist/super-hands.min.js"></script>
    <script src="https://unpkg.com/aframe-physics-system@4.0.1/dist/aframe-physics-system.min.js"></script>
    <script src="./components/room-generator.js"></script>
    <script src="./components/color-changer.js"></script>
    <script src="./components/furniture-spawner.js"></script>
    <script src="./components/ui-panel.js"></script>
</head>
<body>
    <a-scene
        physics="gravity: -9.8; debug: false"
        vr-mode-ui="enabled: true"
        renderer="antialias: true; colorManagement: true; physicallyCorrectLights: true"
    >
        <!-- Assets -->
        <a-assets>
            <!-- Textures -->
            <img id="floor-texture" src="assets/textures/wood-floor.jpg">
            <img id="wall-texture" src="assets/textures/wall-paint.jpg">

            <!-- 3D Models -->
            <a-asset-item id="desk-model" src="assets/models/desk.glb"></a-asset-item>
            <a-asset-item id="piano-model" src="assets/models/piano.glb"></a-asset-item>
            <a-asset-item id="bed-model" src="assets/models/bed.glb"></a-asset-item>
            <a-asset-item id="storage-model" src="assets/models/storage.glb"></a-asset-item>

            <!-- UI Assets -->
            <img id="menu-bg" src="assets/ui/menu-background.png">
        </a-assets>

        <!-- Lighting -->
        <a-entity light="type: ambient; intensity: 0.4; color: #FFF"></a-entity>

        <a-entity light="type: directional; intensity: 0.8; color: #FFF5E6"
                  position="2 4 1"
                  shadow="cast: true"></a-entity>

        <a-entity light="type: point; intensity: 0.6; color: #FFF8DC"
                  position="0 2.4 0"></a-entity>

        <!-- Room Structure -->
        <a-entity id="room" room-generator="width: 2.83; depth: 2.75; height: 2.50">
            <!-- Floor -->
            <a-plane
                id="floor"
                static-body
                width="2.83"
                height="2.75"
                rotation="-90 0 0"
                position="0 0 0"
                material="src: #floor-texture; repeat: 3 3; roughness: 0.8"
                shadow="receive: true">
            </a-plane>

            <!-- Ceiling -->
            <a-plane
                id="ceiling"
                width="2.83"
                height="2.75"
                rotation="90 0 0"
                position="0 2.5 0"
                material="color: #E8D5C4; roughness: 0.9">
            </a-plane>

            <!-- Walls -->
            <a-plane
                id="wall-back"
                class="wall"
                width="2.83"
                height="2.5"
                position="0 1.25 -1.375"
                material="color: #D4735E; roughness: 0.85"
                shadow="receive: true">
            </a-plane>

            <a-plane
                id="wall-front"
                class="wall"
                width="2.83"
                height="2.5"
                position="0 1.25 1.375"
                rotation="0 180 0"
                material="color: #D4735E; roughness: 0.85"
                shadow="receive: true">
            </a-plane>

            <a-plane
                id="wall-left"
                class="wall"
                width="2.75"
                height="2.5"
                position="-1.415 1.25 0"
                rotation="0 90 0"
                material="color: #D4735E; roughness: 0.85"
                shadow="receive: true">
            </a-plane>

            <a-plane
                id="wall-right"
                class="wall"
                width="2.75"
                height="2.5"
                position="1.415 1.25 0"
                rotation="0 -90 0"
                material="color: #D4735E; roughness: 0.85"
                shadow="receive: true">
            </a-plane>

            <!-- Window -->
            <a-plane
                id="window"
                width="0.8"
                height="1.2"
                position="1.41 1.3 0"
                rotation="0 -90 0"
                material="color: #87CEEB; opacity: 0.3; transparent: true">
            </a-plane>
        </a-entity>

        <!-- Furniture (Interactive) -->
        <a-entity id="furniture-container">
            <!-- Desk -->
            <a-box
                id="desk"
                class="furniture grabbable"
                width="1.2" height="0.75" depth="0.6"
                position="-0.8 0.375 -0.8"
                material="color: #8B6F47"
                dynamic-body="shape: box; mass: 5"
                grabbable
                stretchable
                draggable
                shadow="cast: true; receive: true">
                <a-text value="Desk" align="center" position="0 0.4 0" scale="0.5 0.5 0.5"></a-text>
            </a-box>

            <!-- Piano -->
            <a-box
                id="piano"
                class="furniture grabbable"
                width="1.4" height="1.2" depth="0.6"
                position="0.5 0.6 -1.1"
                material="color: #2C2416"
                dynamic-body="shape: box; mass: 10"
                grabbable
                stretchable
                draggable
                shadow="cast: true; receive: true">
                <a-text value="Piano" align="center" position="0 0.7 0" scale="0.5 0.5 0.5"></a-text>
            </a-box>

            <!-- Bed -->
            <a-box
                id="bed"
                class="furniture grabbable"
                width="1.0" height="0.4" depth="2.0"
                position="0.8 0.2 0.3"
                material="color: #A0826D"
                dynamic-body="shape: box; mass: 8"
                grabbable
                stretchable
                draggable
                shadow="cast: true; receive: true">
                <a-text value="Bed" align="center" position="0 0.3 0" scale="0.5 0.5 0.5"></a-text>
            </a-box>
        </a-entity>

        <!-- VR UI Panel -->
        <a-entity id="ui-panel" position="1.5 1.5 0" rotation="0 -45 0">
            <a-plane
                width="0.8"
                height="1.0"
                material="color: #1a1a1a; opacity: 0.9; transparent: true"
                shadow>
            </a-plane>

            <!-- Title -->
            <a-text
                value="Room Designer"
                align="center"
                position="0 0.45 0.01"
                width="0.7"
                color="#fff">
            </a-text>

            <!-- Color Palette Buttons -->
            <a-text
                value="Color Palettes"
                align="center"
                position="0 0.3 0.01"
                width="0.6"
                color="#aaa">
            </a-text>

            <a-box
                class="ui-button palette-btn"
                data-palette="warm"
                width="0.2" height="0.08" depth="0.02"
                position="-0.25 0.15 0.01"
                material="color: #D4735E"
                clickable>
                <a-text value="Warm" align="center" position="0 0 0.02" width="0.4" color="#fff"></a-text>
            </a-box>

            <a-box
                class="ui-button palette-btn"
                data-palette="soft"
                width="0.2" height="0.08" depth="0.02"
                position="0 0.15 0.01"
                material="color: #B8C5D6"
                clickable>
                <a-text value="Soft" align="center" position="0 0 0.02" width="0.4" color="#333"></a-text>
            </a-box>

            <a-box
                class="ui-button palette-btn"
                data-palette="calm"
                width="0.2" height="0.08" depth="0.02"
                position="0.25 0.15 0.01"
                material="color: #A8B5A0"
                clickable>
                <a-text value="Calm" align="center" position="0 0 0.02" width="0.4" color="#fff"></a-text>
            </a-box>

            <!-- Furniture Spawner -->
            <a-text
                value="Add Furniture"
                align="center"
                position="0 -0.05 0.01"
                width="0.6"
                color="#aaa">
            </a-text>

            <a-box
                class="ui-button spawn-btn"
                data-furniture="chair"
                width="0.15" height="0.08" depth="0.02"
                position="-0.2 -0.2 0.01"
                material="color: #4CAF50"
                clickable>
                <a-text value="Chair" align="center" position="0 0 0.02" width="0.35" color="#fff"></a-text>
            </a-box>

            <a-box
                class="ui-button spawn-btn"
                data-furniture="shelf"
                width="0.15" height="0.08" depth="0.02"
                position="0.2 -0.2 0.01"
                material="color: #2196F3"
                clickable>
                <a-text value="Shelf" align="center" position="0 0 0.02" width="0.35" color="#fff"></a-text>
            </a-box>

            <!-- Reset Button -->
            <a-box
                id="reset-btn"
                class="ui-button"
                width="0.3" height="0.08" depth="0.02"
                position="0 -0.4 0.01"
                material="color: #f44336"
                clickable>
                <a-text value="Reset Room" align="center" position="0 0 0.02" width="0.5" color="#fff"></a-text>
            </a-box>
        </a-entity>

        <!-- Camera Rig (VR Player) -->
        <a-entity id="rig"
                  movement-controls="speed: 0.15; fly: false"
                  position="0 0 1.5">

            <!-- Camera -->
            <a-entity id="camera"
                      camera
                      position="0 1.6 0"
                      look-controls="pointerLockEnabled: false">
            </a-entity>

            <!-- Left Hand Controller -->
            <a-entity id="left-hand"
                      hand-controls="hand: left; handModelStyle: highPoly; color: #15ACCF"
                      laser-controls
                      super-hands="colliderEvent: raycaster-intersection;
                                   colliderEventProperty: els;
                                   colliderEndEvent: raycaster-intersection-cleared;
                                   colliderEndEventProperty: clearedEls">
            </a-entity>

            <!-- Right Hand Controller -->
            <a-entity id="right-hand"
                      hand-controls="hand: right; handModelStyle: highPoly; color: #15ACCF"
                      laser-controls
                      super-hands="colliderEvent: raycaster-intersection;
                                   colliderEventProperty: els;
                                   colliderEndEvent: raycaster-intersection-cleared;
                                   colliderEndEventProperty: clearedEls">
            </a-entity>
        </a-entity>

        <!-- Teleportation System -->
        <a-entity id="teleport-target"
                  visible="false"
                  geometry="primitive: cylinder; radius: 0.3; height: 0.01"
                  material="color: #00ff00; opacity: 0.5; transparent: true">
        </a-entity>

        <!-- Performance Monitor -->
        <a-entity
            id="stats"
            position="-2 2 -2"
            text="value: FPS: 90; color: #0f0; align: left; width: 3">
        </a-entity>
    </a-scene>

    <script type="module" src="./main.js"></script>
</body>
</html>
```

### Step 3: Color Changer Component

**File: `components/color-changer.js`**
```javascript
AFRAME.registerComponent('color-changer', {
    schema: {
        palette: { type: 'string', default: 'warm' }
    },

    init: function() {
        this.palettes = {
            warm: {
                wall: '#D4735E',
                floor: '#E8D5C4',
                ceiling: '#E8D5C4'
            },
            soft: {
                wall: '#B8C5D6',
                floor: '#F5F3EF',
                ceiling: '#F5F3EF'
            },
            calm: {
                wall: '#A8B5A0',
                floor: '#F2EBD9',
                ceiling: '#F2EBD9'
            }
        };

        // Listen for palette change events
        this.el.sceneEl.addEventListener('changePalette', (e) => {
            this.applyPalette(e.detail.palette);
        });

        // Listen for button clicks
        const buttons = document.querySelectorAll('.palette-btn');
        buttons.forEach(btn => {
            btn.addEventListener('click', (e) => {
                const palette = e.target.getAttribute('data-palette');
                this.applyPalette(palette);
            });
        });
    },

    applyPalette: function(paletteName) {
        const palette = this.palettes[paletteName];
        if (!palette) return;

        console.log('Applying palette:', paletteName);

        // Update walls
        const walls = document.querySelectorAll('.wall');
        walls.forEach(wall => {
            wall.setAttribute('material', 'color', palette.wall);
        });

        // Update floor
        const floor = document.querySelector('#floor');
        if (floor) {
            floor.setAttribute('material', 'color', palette.floor);
        }

        // Update ceiling
        const ceiling = document.querySelector('#ceiling');
        if (ceiling) {
            ceiling.setAttribute('material', 'color', palette.ceiling);
        }

        // Trigger haptic feedback
        this.triggerHaptic();
    },

    triggerHaptic: function() {
        const hands = document.querySelectorAll('[hand-controls]');
        hands.forEach(hand => {
            const gamepad = hand.components['hand-controls']?.gamepad;
            if (gamepad && gamepad.hapticActuators) {
                gamepad.hapticActuators[0].pulse(0.5, 100);
            }
        });
    }
});
```

### Step 4: Furniture Spawner Component

**File: `components/furniture-spawner.js`**
```javascript
AFRAME.registerComponent('furniture-spawner', {
    init: function() {
        this.furnitureTemplates = {
            chair: {
                primitive: 'box',
                width: 0.5,
                height: 0.5,
                depth: 0.5,
                color: '#654321'
            },
            shelf: {
                primitive: 'box',
                width: 0.8,
                height: 1.2,
                depth: 0.3,
                color: '#9C826B'
            },
            table: {
                primitive: 'box',
                width: 1.0,
                height: 0.75,
                depth: 0.8,
                color: '#8B6F47'
            }
        };

        // Listen for spawn button clicks
        const spawnButtons = document.querySelectorAll('.spawn-btn');
        spawnButtons.forEach(btn => {
            btn.addEventListener('click', (e) => {
                const furnitureType = e.target.getAttribute('data-furniture');
                this.spawnFurniture(furnitureType);
            });
        });

        // Listen for reset button
        const resetBtn = document.querySelector('#reset-btn');
        if (resetBtn) {
            resetBtn.addEventListener('click', () => {
                this.resetRoom();
            });
        }
    },

    spawnFurniture: function(type) {
        const template = this.furnitureTemplates[type];
        if (!template) return;

        const container = document.querySelector('#furniture-container');
        const camera = document.querySelector('#camera');

        // Spawn in front of camera
        const cameraPos = camera.getAttribute('position');
        const cameraRot = camera.getAttribute('rotation');

        // Calculate spawn position (1 meter in front of camera)
        const spawnDistance = 1.5;
        const radians = (cameraRot.y * Math.PI) / 180;
        const spawnX = cameraPos.x - Math.sin(radians) * spawnDistance;
        const spawnZ = cameraPos.z - Math.cos(radians) * spawnDistance;

        // Create furniture entity
        const furniture = document.createElement('a-box');
        furniture.setAttribute('class', 'furniture grabbable');
        furniture.setAttribute('width', template.width);
        furniture.setAttribute('height', template.height);
        furniture.setAttribute('depth', template.depth);
        furniture.setAttribute('position', `${spawnX} ${template.height / 2} ${spawnZ}`);
        furniture.setAttribute('material', `color: ${template.color}`);
        furniture.setAttribute('dynamic-body', 'shape: box; mass: 5');
        furniture.setAttribute('grabbable', '');
        furniture.setAttribute('stretchable', '');
        furniture.setAttribute('draggable', '');
        furniture.setAttribute('shadow', 'cast: true; receive: true');

        // Add label
        const label = document.createElement('a-text');
        label.setAttribute('value', type.charAt(0).toUpperCase() + type.slice(1));
        label.setAttribute('align', 'center');
        label.setAttribute('position', `0 ${template.height / 2 + 0.1} 0`);
        label.setAttribute('scale', '0.5 0.5 0.5');
        furniture.appendChild(label);

        container.appendChild(furniture);

        console.log(`Spawned ${type} at position:`, spawnX, spawnZ);

        // Haptic feedback
        this.triggerHaptic();
    },

    resetRoom: function() {
        const furniture = document.querySelectorAll('#furniture-container .furniture');

        // Remove all spawned furniture (keep originals)
        const originalIds = ['desk', 'piano', 'bed'];
        furniture.forEach(item => {
            if (!originalIds.includes(item.id)) {
                item.parentNode.removeChild(item);
            }
        });

        console.log('Room reset');
        this.triggerHaptic();
    },

    triggerHaptic: function() {
        const hands = document.querySelectorAll('[hand-controls]');
        hands.forEach(hand => {
            const gamepad = hand.components['hand-controls']?.gamepad;
            if (gamepad && gamepad.hapticActuators) {
                gamepad.hapticActuators[0].pulse(0.7, 150);
            }
        });
    }
});
```

### Step 5: Main JavaScript

**File: `main.js`**
```javascript
// Initialize color changer on scene
document.addEventListener('DOMContentLoaded', () => {
    const scene = document.querySelector('a-scene');

    scene.addEventListener('loaded', () => {
        console.log('VR Scene loaded');

        // Initialize color changer
        scene.setAttribute('color-changer', '');

        // Initialize furniture spawner
        scene.setAttribute('furniture-spawner', '');

        // Performance monitoring
        monitorPerformance();

        // Setup clickable UI elements
        setupUIInteractions();
    });
});

function setupUIInteractions() {
    // Make UI buttons interactive
    const buttons = document.querySelectorAll('.ui-button');

    buttons.forEach(btn => {
        btn.setAttribute('class', btn.getAttribute('class') + ' clickable');

        // Visual feedback on hover
        btn.addEventListener('mouseenter', () => {
            btn.setAttribute('scale', '1.1 1.1 1.1');
        });

        btn.addEventListener('mouseleave', () => {
            btn.setAttribute('scale', '1 1 1');
        });
    });
}

function monitorPerformance() {
    const stats = document.querySelector('#stats');
    let lastTime = performance.now();
    let frames = 0;
    let fps = 0;

    function updateStats() {
        frames++;
        const currentTime = performance.now();

        if (currentTime >= lastTime + 1000) {
            fps = Math.round((frames * 1000) / (currentTime - lastTime));
            frames = 0;
            lastTime = currentTime;

            if (stats) {
                const color = fps >= 80 ? '#0f0' : fps >= 60 ? '#ff0' : '#f00';
                stats.setAttribute('text', `value: FPS: ${fps}; color: ${color}`);
            }
        }

        requestAnimationFrame(updateStats);
    }

    updateStats();
}

// VR session management
if (navigator.xr) {
    navigator.xr.isSessionSupported('immersive-vr').then((supported) => {
        if (supported) {
            console.log('VR is supported!');
        } else {
            console.log('VR not supported, fallback to desktop mode');
        }
    });
}
```

## Expected Outputs

1. **Immersive VR Room**: 1:1 scale room environment
2. **VR Controllers**: Full hand tracking and controller support
3. **Interactive Furniture**: Grab, move, and place furniture
4. **Color Switching**: Real-time palette changes in VR
5. **VR UI**: Floating menu panels with buttons
6. **Smooth Performance**: 90 FPS on VR hardware
7. **Haptic Feedback**: Controller vibrations for interactions

## Bonus Challenges

1. **Multiplayer VR**: Share room design sessions
2. **Voice Commands**: "Change color to warm"
3. **Gesture Recognition**: Hand gestures for controls
4. **Room Scanner**: Import real room dimensions
5. **VR Measuring Tape**: Measure distances in VR
6. **Screenshot/Video**: Capture VR views
7. **AR Passthrough**: Mixed reality mode (Quest 3)

## Resources

- [A-Frame Documentation](https://aframe.io/docs/)
- [WebXR Device API](https://developer.mozilla.org/en-US/docs/Web/API/WebXR_Device_API)
- [A-Frame School](https://aframe.io/aframe-school/)
- [Super Hands](https://github.com/wmurphyrd/aframe-super-hands-component)
- [VR Best Practices](https://developer.oculus.com/resources/bp-intro/)

## Success Criteria

- [ ] Room loads with correct dimensions
- [ ] VR mode activates on headset
- [ ] Controllers work in both hands
- [ ] Furniture can be grabbed and moved
- [ ] Color palettes switch in real-time
- [ ] UI buttons are clickable
- [ ] Performance stays above 80 FPS
- [ ] No motion sickness triggers
- [ ] Works on Quest 2, Quest 3, and desktop VR
