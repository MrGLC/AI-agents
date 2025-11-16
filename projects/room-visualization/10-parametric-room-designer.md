# Project 10: Real-Time Parametric Room Designer

## Overview
Build a real-time parametric room design system where every aspect of the room—dimensions, colors, materials, furniture placement, and lighting—is controlled by adjustable parameters. Changes update instantly in a 3D viewport, allowing rapid iteration and exploration of design possibilities. Perfect for professional interior designers and DIY enthusiasts.

## Learning Objectives
- Master parametric design principles
- Implement real-time 3D rendering pipelines
- Build reactive UI with instant feedback
- Create constraint-based layout systems
- Develop material and shader systems
- Export production-ready specifications

## Difficulty Level
**Advanced** - Requires strong programming skills, 3D graphics knowledge, and UI/UX design

## Technical Stack
- **Frontend**: React + TypeScript
- **3D Engine**: Three.js + React Three Fiber
- **UI Framework**: Leva (parameter controls), Zustand (state)
- **3D Helpers**:
  - drei (Three.js helpers)
  - postprocessing (effects)
- **Build**: Vite
- **Export**: jsPDF, GLTFExporter
- **Math**: gl-matrix, mathjs

## Room Specifications
- **Parametric Dimensions**: Adjustable 1-5m width/depth/height
- **Default**: 2.83m × 2.75m × 2.50m
- **Constraints**: Furniture auto-adjusts to room size
- **Real-Time**: 60 FPS minimum
- **Export**: Floor plans, 3D models, shopping lists

## Requirements

### 1. Parametric Controls
- [x] Room dimensions (width, depth, height)
- [x] Wall thickness
- [x] Window/door placement and size
- [x] Floor level and ceiling height
- [x] Grid system toggle

### 2. Material System
- [x] Wall materials (paint, wallpaper, brick, wood)
- [x] Floor materials (wood, tile, carpet)
- [x] Color parameters (hue, saturation, lightness)
- [x] Texture scaling and rotation
- [x] PBR material properties (roughness, metallic)

### 3. Furniture Parameters
- [x] Position (X, Y, Z)
- [x] Rotation (0-360°)
- [x] Scale (uniform or per-axis)
- [x] Auto-layout presets
- [x] Snap-to-grid and snap-to-wall

### 4. Lighting Controls
- [x] Ambient light intensity and color
- [x] Directional sun light (position, color, intensity)
- [x] Point lights (count, position, intensity)
- [x] Shadows (quality, softness)
- [x] Time-of-day presets

### 5. Real-Time Features
- [x] Instant parameter updates (<16ms)
- [x] Smooth camera controls
- [x] Performance optimization
- [x] Undo/redo stack
- [x] Preset save/load

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create React app with Vite
npm create vite@latest parametric-room-designer -- --template react-ts
cd parametric-room-designer

# Install dependencies
npm install three @react-three/fiber @react-three/drei
npm install @react-three/postprocessing
npm install leva zustand
npm install gl-matrix mathjs
npm install jspdf
npm install @types/three --save-dev
```

**File: `package.json`**
```json
{
  "name": "parametric-room-designer",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "three": "^0.158.0",
    "@react-three/fiber": "^8.15.0",
    "@react-three/drei": "^9.88.0",
    "@react-three/postprocessing": "^2.15.0",
    "leva": "^0.9.35",
    "zustand": "^4.4.6",
    "gl-matrix": "^3.4.3",
    "mathjs": "^12.0.0",
    "jspdf": "^2.5.1"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@types/three": "^0.158.0",
    "typescript": "^5.2.0",
    "vite": "^5.0.0"
  }
}
```

### Step 2: State Management Store

**File: `src/store/roomStore.ts`**
```typescript
import create from 'zustand';
import { Vector3 } from 'three';

interface RoomParameters {
  // Dimensions
  width: number;
  depth: number;
  height: number;
  wallThickness: number;

  // Colors
  wallColor: string;
  floorColor: string;
  ceilingColor: string;

  // Materials
  wallMaterial: 'paint' | 'wallpaper' | 'brick' | 'wood';
  floorMaterial: 'wood' | 'tile' | 'carpet';
  wallRoughness: number;
  floorRoughness: number;

  // Lighting
  ambientIntensity: number;
  sunIntensity: number;
  sunPosition: [number, number, number];
  enableShadows: boolean;

  // Furniture
  furniture: FurnitureItem[];
}

interface FurnitureItem {
  id: string;
  type: 'desk' | 'piano' | 'bed' | 'storage' | 'chair';
  position: [number, number, number];
  rotation: [number, number, number];
  scale: [number, number, number];
}

interface RoomStore extends RoomParameters {
  // Actions
  updateDimension: (key: keyof Pick<RoomParameters, 'width' | 'depth' | 'height'>, value: number) => void;
  updateColor: (key: keyof Pick<RoomParameters, 'wallColor' | 'floorColor' | 'ceilingColor'>, value: string) => void;
  updateMaterial: (surface: 'wall' | 'floor', material: string) => void;
  addFurniture: (item: FurnitureItem) => void;
  updateFurniture: (id: string, updates: Partial<FurnitureItem>) => void;
  removeFurniture: (id: string) => void;
  resetRoom: () => void;
  loadPreset: (preset: string) => void;
}

const defaultState: RoomParameters = {
  width: 2.83,
  depth: 2.75,
  height: 2.50,
  wallThickness: 0.15,
  wallColor: '#D4735E',
  floorColor: '#E8D5C4',
  ceilingColor: '#E8D5C4',
  wallMaterial: 'paint',
  floorMaterial: 'wood',
  wallRoughness: 0.85,
  floorRoughness: 0.7,
  ambientIntensity: 0.4,
  sunIntensity: 1.2,
  sunPosition: [2, 4, 1],
  enableShadows: true,
  furniture: [
    {
      id: 'desk-1',
      type: 'desk',
      position: [-0.8, 0.375, -0.8],
      rotation: [0, 0, 0],
      scale: [1.2, 0.75, 0.6]
    },
    {
      id: 'piano-1',
      type: 'piano',
      position: [0.5, 0.6, -1.1],
      rotation: [0, 0, 0],
      scale: [1.4, 1.2, 0.6]
    },
    {
      id: 'bed-1',
      type: 'bed',
      position: [0.8, 0.2, 0.3],
      rotation: [0, Math.PI / 2, 0],
      scale: [1.0, 0.4, 2.0]
    }
  ]
};

export const useRoomStore = create<RoomStore>((set) => ({
  ...defaultState,

  updateDimension: (key, value) => set((state) => ({ [key]: value })),

  updateColor: (key, value) => set((state) => ({ [key]: value })),

  updateMaterial: (surface, material) =>
    set((state) => ({
      [`${surface}Material`]: material
    })),

  addFurniture: (item) =>
    set((state) => ({
      furniture: [...state.furniture, item]
    })),

  updateFurniture: (id, updates) =>
    set((state) => ({
      furniture: state.furniture.map((item) =>
        item.id === id ? { ...item, ...updates } : item
      )
    })),

  removeFurniture: (id) =>
    set((state) => ({
      furniture: state.furniture.filter((item) => item.id !== id)
    })),

  resetRoom: () => set(defaultState),

  loadPreset: (preset) => {
    const presets: Record<string, Partial<RoomParameters>> = {
      warm: {
        wallColor: '#D4735E',
        floorColor: '#E8D5C4',
        ceilingColor: '#E8D5C4'
      },
      soft: {
        wallColor: '#B8C5D6',
        floorColor: '#F5F3EF',
        ceilingColor: '#F5F3EF'
      },
      calm: {
        wallColor: '#A8B5A0',
        floorColor: '#F2EBD9',
        ceilingColor: '#F2EBD9'
      }
    };

    if (presets[preset]) {
      set((state) => ({ ...state, ...presets[preset] }));
    }
  }
}));
```

### Step 3: Parametric Room Component

**File: `src/components/ParametricRoom.tsx`**
```typescript
import React, { useMemo } from 'react';
import { useRoomStore } from '../store/roomStore';
import * as THREE from 'three';

export function ParametricRoom() {
  const {
    width,
    depth,
    height,
    wallThickness,
    wallColor,
    floorColor,
    ceilingColor,
    wallRoughness,
    floorRoughness
  } = useRoomStore();

  // Memoize geometries and materials for performance
  const floorGeometry = useMemo(
    () => new THREE.PlaneGeometry(width, depth),
    [width, depth]
  );

  const ceilingGeometry = useMemo(
    () => new THREE.PlaneGeometry(width, depth),
    [width, depth]
  );

  const wallGeometries = useMemo(() => {
    return {
      back: new THREE.PlaneGeometry(width, height),
      front: new THREE.PlaneGeometry(width, height),
      left: new THREE.PlaneGeometry(depth, height),
      right: new THREE.PlaneGeometry(depth, height)
    };
  }, [width, depth, height]);

  return (
    <group>
      {/* Floor */}
      <mesh
        geometry={floorGeometry}
        rotation={[-Math.PI / 2, 0, 0]}
        position={[0, 0, 0]}
        receiveShadow
      >
        <meshStandardMaterial
          color={floorColor}
          roughness={floorRoughness}
          metalness={0.1}
        />
      </mesh>

      {/* Ceiling */}
      <mesh
        geometry={ceilingGeometry}
        rotation={[Math.PI / 2, 0, 0]}
        position={[0, height, 0]}
      >
        <meshStandardMaterial
          color={ceilingColor}
          roughness={0.9}
          metalness={0}
        />
      </mesh>

      {/* Back Wall */}
      <mesh
        geometry={wallGeometries.back}
        position={[0, height / 2, -depth / 2]}
        receiveShadow
      >
        <meshStandardMaterial
          color={wallColor}
          roughness={wallRoughness}
          metalness={0}
        />
      </mesh>

      {/* Front Wall */}
      <mesh
        geometry={wallGeometries.front}
        position={[0, height / 2, depth / 2]}
        rotation={[0, Math.PI, 0]}
        receiveShadow
      >
        <meshStandardMaterial
          color={wallColor}
          roughness={wallRoughness}
          metalness={0}
        />
      </mesh>

      {/* Left Wall */}
      <mesh
        geometry={wallGeometries.left}
        position={[-width / 2, height / 2, 0]}
        rotation={[0, Math.PI / 2, 0]}
        receiveShadow
      >
        <meshStandardMaterial
          color={wallColor}
          roughness={wallRoughness}
          metalness={0}
        />
      </mesh>

      {/* Right Wall (with window) */}
      <mesh
        geometry={wallGeometries.right}
        position={[width / 2, height / 2, 0]}
        rotation={[0, -Math.PI / 2, 0]}
        receiveShadow
      >
        <meshStandardMaterial
          color={wallColor}
          roughness={wallRoughness}
          metalness={0}
        />
      </mesh>

      {/* Window */}
      <mesh
        position={[width / 2 - 0.01, height / 2, 0]}
        rotation={[0, -Math.PI / 2, 0]}
      >
        <planeGeometry args={[0.8, 1.2]} />
        <meshPhysicalMaterial
          color="#87CEEB"
          transparent
          opacity={0.3}
          roughness={0.1}
          metalness={0.1}
          transmission={0.9}
        />
      </mesh>

      {/* Grid Helper (optional) */}
      <gridHelper args={[Math.max(width, depth), 20]} position={[0, 0.01, 0]} />
    </group>
  );
}
```

### Step 4: Parametric Furniture Component

**File: `src/components/ParametricFurniture.tsx`**
```typescript
import React, { useRef, useState } from 'react';
import { useRoomStore } from '../store/roomStore';
import { ThreeEvent } from '@react-three/fiber';
import * as THREE from 'three';

interface FurnitureProps {
  id: string;
  type: string;
  position: [number, number, number];
  rotation: [number, number, number];
  scale: [number, number, number];
}

export function ParametricFurniture({ id, type, position, rotation, scale }: FurnitureProps) {
  const meshRef = useRef<THREE.Mesh>(null);
  const [hovered, setHovered] = useState(false);
  const [dragging, setDragging] = useState(false);

  const updateFurniture = useRoomStore((state) => state.updateFurniture);

  const furnitureColors: Record<string, string> = {
    desk: '#8B6F47',
    piano: '#2C2416',
    bed: '#A0826D',
    storage: '#9C826B',
    chair: '#654321'
  };

  const handlePointerDown = (e: ThreeEvent<PointerEvent>) => {
    e.stopPropagation();
    setDragging(true);
  };

  const handlePointerUp = (e: ThreeEvent<PointerEvent>) => {
    setDragging(false);
  };

  const handlePointerMove = (e: ThreeEvent<PointerEvent>) => {
    if (dragging) {
      // Update position based on pointer movement
      const newPosition: [number, number, number] = [
        e.point.x,
        position[1],
        e.point.z
      ];
      updateFurniture(id, { position: newPosition });
    }
  };

  return (
    <mesh
      ref={meshRef}
      position={position}
      rotation={rotation}
      scale={scale}
      castShadow
      receiveShadow
      onPointerOver={() => setHovered(true)}
      onPointerOut={() => setHovered(false)}
      onPointerDown={handlePointerDown}
      onPointerUp={handlePointerUp}
      onPointerMove={handlePointerMove}
    >
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial
        color={furnitureColors[type] || '#888888'}
        roughness={0.7}
        metalness={0.1}
        emissive={hovered ? '#444444' : '#000000'}
        emissiveIntensity={hovered ? 0.2 : 0}
      />

      {/* Label */}
      {hovered && (
        <group position={[0, 0.6, 0]}>
          <mesh>
            <planeGeometry args={[0.8, 0.2]} />
            <meshBasicMaterial color="#000000" opacity={0.7} transparent />
          </mesh>
          {/* Text would be rendered here in a real implementation */}
        </group>
      )}

      {/* Selection outline */}
      {dragging && (
        <lineSegments>
          <edgesGeometry args={[new THREE.BoxGeometry(1.1, 1.1, 1.1)]} />
          <lineBasicMaterial color="#00ff00" linewidth={2} />
        </lineSegments>
      )}
    </mesh>
  );
}
```

### Step 5: Main App Component

**File: `src/App.tsx`**
```typescript
import React, { Suspense } from 'react';
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Sky, ContactShadows } from '@react-three/drei';
import { EffectComposer, SSAO } from '@react-three/postprocessing';
import { useControls, button, folder } from 'leva';
import { useRoomStore } from './store/roomStore';
import { ParametricRoom } from './components/ParametricRoom';
import { ParametricFurniture } from './components/ParametricFurniture';
import './App.css';

function Scene() {
  const {
    ambientIntensity,
    sunIntensity,
    sunPosition,
    enableShadows,
    furniture
  } = useRoomStore();

  return (
    <>
      {/* Lighting */}
      <ambientLight intensity={ambientIntensity} />

      <directionalLight
        position={sunPosition}
        intensity={sunIntensity}
        castShadow={enableShadows}
        shadow-mapSize={[2048, 2048]}
        shadow-camera-far={15}
        shadow-camera-left={-5}
        shadow-camera-right={5}
        shadow-camera-top={5}
        shadow-camera-bottom={-5}
      />

      <pointLight position={[0, 2.4, 0]} intensity={0.6} castShadow />

      {/* Room */}
      <ParametricRoom />

      {/* Furniture */}
      {furniture.map((item) => (
        <ParametricFurniture key={item.id} {...item} />
      ))}

      {/* Shadows */}
      <ContactShadows
        position={[0, 0.01, 0]}
        opacity={0.5}
        scale={10}
        blur={1.5}
        far={2}
      />

      {/* Sky */}
      <Sky sunPosition={sunPosition} />

      {/* Camera Controls */}
      <OrbitControls
        makeDefault
        minDistance={2}
        maxDistance={10}
        maxPolarAngle={Math.PI / 2 - 0.1}
      />
    </>
  );
}

function App() {
  const {
    width,
    depth,
    height,
    wallColor,
    floorColor,
    ceilingColor,
    wallRoughness,
    floorRoughness,
    ambientIntensity,
    sunIntensity,
    updateDimension,
    updateColor,
    loadPreset,
    resetRoom
  } = useRoomStore();

  // Leva controls
  useControls({
    'Room Dimensions': folder({
      width: {
        value: width,
        min: 1,
        max: 5,
        step: 0.01,
        onChange: (v) => updateDimension('width', v)
      },
      depth: {
        value: depth,
        min: 1,
        max: 5,
        step: 0.01,
        onChange: (v) => updateDimension('depth', v)
      },
      height: {
        value: height,
        min: 2,
        max: 4,
        step: 0.01,
        onChange: (v) => updateDimension('height', v)
      }
    }),
    'Colors': folder({
      wallColor: {
        value: wallColor,
        onChange: (v) => updateColor('wallColor', v)
      },
      floorColor: {
        value: floorColor,
        onChange: (v) => updateColor('floorColor', v)
      },
      ceilingColor: {
        value: ceilingColor,
        onChange: (v) => updateColor('ceilingColor', v)
      }
    }),
    'Materials': folder({
      wallRoughness: {
        value: wallRoughness,
        min: 0,
        max: 1,
        step: 0.01
      },
      floorRoughness: {
        value: floorRoughness,
        min: 0,
        max: 1,
        step: 0.01
      }
    }),
    'Lighting': folder({
      ambientIntensity: {
        value: ambientIntensity,
        min: 0,
        max: 2,
        step: 0.1
      },
      sunIntensity: {
        value: sunIntensity,
        min: 0,
        max: 3,
        step: 0.1
      }
    }),
    'Presets': folder({
      'Warm Focus': button(() => loadPreset('warm')),
      'Soft Tech': button(() => loadPreset('soft')),
      'Calm Studio': button(() => loadPreset('calm')),
      'Reset': button(() => resetRoom())
    })
  });

  return (
    <div className="app">
      <Canvas
        shadows
        camera={{ position: [3, 2, 4], fov: 60 }}
        gl={{ antialias: true }}
      >
        <Suspense fallback={null}>
          <Scene />

          {/* Post-processing */}
          <EffectComposer>
            <SSAO radius={0.5} intensity={50} />
          </EffectComposer>
        </Suspense>
      </Canvas>

      {/* Info Panel */}
      <div className="info-panel">
        <h2>Parametric Room Designer</h2>
        <p>Room: {width.toFixed(2)}m × {depth.toFixed(2)}m × {height.toFixed(2)}m</p>
        <p>Area: {(width * depth).toFixed(2)}m²</p>
      </div>
    </div>
  );
}

export default App;
```

### Step 6: Styling

**File: `src/App.css`**
```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  overflow: hidden;
}

.app {
  width: 100vw;
  height: 100vh;
  position: relative;
}

.info-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(255, 255, 255, 0.95);
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  pointer-events: none;
}

.info-panel h2 {
  font-size: 18px;
  margin-bottom: 10px;
  color: #333;
}

.info-panel p {
  font-size: 14px;
  color: #666;
  margin-bottom: 4px;
}
```

## Expected Outputs

1. **Real-Time 3D View**: Smooth 60 FPS viewport
2. **Parametric Controls**: All parameters adjustable via UI
3. **Instant Updates**: Changes reflect immediately
4. **Interactive Furniture**: Drag-and-drop placement
5. **Multiple Presets**: One-click palette switching
6. **Export Capabilities**: Floor plans, 3D models, specifications

## Bonus Challenges

1. **Constraint System**: Auto-resize furniture when room shrinks
2. **Auto-Layout**: AI-powered furniture arrangement
3. **Material Library**: 100+ realistic materials
4. **Animation Presets**: Camera path recordings
5. **Collaboration**: Real-time multi-user editing
6. **VR Mode**: WebXR support for VR editing
7. **Cost Estimation**: Real-time budget calculator

## Resources

- [React Three Fiber](https://docs.pmnd.rs/react-three-fiber/)
- [Drei Helpers](https://github.com/pmndrs/drei)
- [Leva Controls](https://github.com/pmndrs/leva)
- [Zustand State](https://github.com/pmndrs/zustand)
- [Three.js Manual](https://threejs.org/manual/)

## Success Criteria

- [ ] Room dimensions update in real-time (<16ms)
- [ ] All 3 color palettes work instantly
- [ ] Furniture can be dragged and placed
- [ ] 60 FPS maintained during interaction
- [ ] Undo/redo functionality works
- [ ] Presets load correctly
- [ ] Export generates valid files
- [ ] Responsive on different screen sizes
- [ ] No visual glitches during parameter changes
- [ ] Memory usage stays under 500MB
