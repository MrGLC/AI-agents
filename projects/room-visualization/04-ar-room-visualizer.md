# Project 04: AR Room Visualizer - Mobile App

## Overview
Build a mobile augmented reality application that lets users visualize furniture and color schemes in their actual room using their phone camera. Users can place virtual furniture, test paint colors on real walls, and see design changes in real-time through AR.

## Learning Objectives
- Master AR development with ARKit (iOS) and ARCore (Android)
- Implement plane detection and surface tracking
- Build 3D object placement with physics
- Create real-time color overlay on detected surfaces
- Develop cross-platform mobile apps with React Native or Flutter
- Integrate ML for surface segmentation

## Difficulty Level
**Advanced** - Requires mobile development, AR frameworks, and 3D graphics

## Technical Stack
- **Mobile Framework**: React Native + ViroReact OR Flutter + ARCore/ARKit
- **AR SDKs**:
  - ARKit (iOS)
  - ARCore (Android)
  - ViroReact or AR Foundation
- **3D Models**: GLTF/GLB format
- **ML**: TensorFlow Lite for surface segmentation
- **Backend**: Firebase for asset storage
- **Additional**: Expo (React Native), Unity AR Foundation (alternative)

## Room Specifications
- **Target Room**: 2.83m × 2.75m
- **Furniture Scale**: 1:1 real-world scale
- **Color Palettes**: Real-time wall color overlay
- **Lighting**: Ambient occlusion and lighting estimation

## Requirements

### 1. AR Core Features
- [x] Plane detection (horizontal and vertical)
- [x] Surface tracking and anchoring
- [x] Environmental lighting estimation
- [x] Hit testing for object placement
- [x] Occlusion (object behind real furniture)

### 2. Furniture Placement
- [x] Drag-and-drop 3D furniture models
- [x] Rotation and scaling controls
- [x] Snap to detected surfaces
- [x] Collision detection
- [x] Shadow rendering

### 3. Color Visualization
- [x] Detect wall surfaces automatically
- [x] Apply color overlays to walls
- [x] Test multiple color palettes
- [x] Real-time preview with room lighting
- [x] Save before/after comparisons

### 4. Measurement Tools
- [x] AR measuring tape
- [x] Room dimension calculator
- [x] Furniture size indicators
- [x] Clearance checker

### 5. Export & Share
- [x] Capture AR screenshots
- [x] Record AR videos
- [x] Save room configurations
- [x] Share designs on social media
- [x] Generate shopping lists

## Step-by-Step Implementation

### Step 1: Project Setup (React Native + ViroReact)

```bash
# Install React Native CLI
npm install -g react-native-cli

# Create new project
npx react-native init ARRoomVisualizer
cd ARRoomVisualizer

# Install ViroReact
npm install react-viro --save

# Link dependencies (if needed)
npx react-native link react-viro

# Install additional dependencies
npm install @react-native-community/async-storage
npm install react-native-fs
npm install react-native-share
```

**File: `package.json`**
```json
{
  "name": "ARRoomVisualizer",
  "version": "1.0.0",
  "dependencies": {
    "react": "18.2.0",
    "react-native": "0.72.0",
    "react-viro": "^2.23.0",
    "@react-native-community/async-storage": "^1.12.1",
    "react-native-fs": "^2.20.0",
    "react-native-share": "^9.0.0",
    "react-native-permissions": "^3.8.0"
  }
}
```

### Step 2: Main AR Scene Component

**File: `src/ARScene.js`**
```javascript
import React, { Component } from 'react';
import {
  ViroARScene,
  ViroARSceneNavigator,
  ViroAmbientLight,
  ViroSpotLight,
  Viro3DObject,
  ViroNode,
  ViroARPlane,
  ViroARPlaneSelector,
  ViroText,
  ViroMaterials,
  ViroQuad,
  ViroARTrackingTargets
} from 'react-viro';
import { StyleSheet } from 'react-native';

class ARRoomScene extends Component {
  constructor(props) {
    super(props);

    this.state = {
      planeDetected: false,
      furnitureItems: [],
      selectedFurniture: null,
      currentPalette: 'warm',
      wallOverlays: [],
      measurements: null
    };

    // Color palettes
    this.palettes = {
      warm: { wall: '#D4735E', floor: '#E8D5C4' },
      soft: { wall: '#B8C5D6', floor: '#F5F3EF' },
      calm: { wall: '#A8B5A0', floor: '#F2EBD9' }
    };

    // Furniture catalog
    this.furnitureModels = {
      desk: {
        source: require('./assets/models/desk.glb'),
        scale: [1, 1, 1],
        position: [0, 0, -2]
      },
      piano: {
        source: require('./assets/models/piano.glb'),
        scale: [1, 1, 1],
        position: [0, 0, -2.5]
      },
      bed: {
        source: require('./assets/models/bed.glb'),
        scale: [1, 1, 1],
        position: [0, 0, -3]
      }
    };

    this._onARPlaneDetected = this._onARPlaneDetected.bind(this);
    this._onPlaneSelected = this._onPlaneSelected.bind(this);
    this._addFurniture = this._addFurniture.bind(this);
  }

  componentDidMount() {
    // Define materials for wall overlays
    ViroMaterials.createMaterials({
      wallMaterial: {
        diffuseColor: this.palettes[this.state.currentPalette].wall,
        lightingModel: 'Blinn',
        opacity: 0.8
      },
      floorMaterial: {
        diffuseColor: this.palettes[this.state.currentPalette].floor,
        lightingModel: 'Blinn',
        opacity: 0.6
      }
    });
  }

  _onARPlaneDetected(plane) {
    console.log('Plane detected:', plane);
    this.setState({ planeDetected: true });

    // Store plane information for measurements
    if (plane.alignment === 'Vertical') {
      // Wall detected
      this._addWallOverlay(plane);
    }
  }

  _onPlaneSelected(plane) {
    console.log('Plane selected:', plane);
    // User tapped on a detected plane
    // Can be used for furniture placement
  }

  _addWallOverlay(plane) {
    // Create semi-transparent color overlay on detected wall
    const wallOverlay = {
      id: `wall_${Date.now()}`,
      position: plane.center,
      width: plane.width,
      height: plane.height,
      rotation: plane.rotation,
      color: this.palettes[this.state.currentPalette].wall
    };

    this.setState(prevState => ({
      wallOverlays: [...prevState.wallOverlays, wallOverlay]
    }));
  }

  _addFurniture(type, position) {
    const model = this.furnitureModels[type];
    if (!model) return;

    const furniture = {
      id: `${type}_${Date.now()}`,
      type: type,
      position: position || model.position,
      scale: model.scale,
      rotation: [0, 0, 0],
      source: model.source
    };

    this.setState(prevState => ({
      furnitureItems: [...prevState.furnitureItems, furniture]
    }));
  }

  _rotateFurniture(id, angle) {
    this.setState(prevState => ({
      furnitureItems: prevState.furnitureItems.map(item =>
        item.id === id
          ? { ...item, rotation: [0, item.rotation[1] + angle, 0] }
          : item
      )
    }));
  }

  _removeFurniture(id) {
    this.setState(prevState => ({
      furnitureItems: prevState.furnitureItems.filter(item => item.id !== id)
    }));
  }

  render() {
    const { furnitureItems, wallOverlays, planeDetected } = this.state;

    return (
      <ViroARScene
        onTrackingUpdated={this._onTrackingUpdated}
        onAnchorFound={this._onAnchorFound}
      >
        {/* Ambient lighting for realistic appearance */}
        <ViroAmbientLight color="#ffffff" intensity={300} />

        {/* Directional light for shadows */}
        <ViroSpotLight
          innerAngle={5}
          outerAngle={20}
          direction={[0, -1, 0]}
          position={[0, 5, 0]}
          color="#ffffff"
          castsShadow={true}
          shadowMapSize={2048}
          shadowNearZ={2}
          shadowFarZ={7}
          shadowOpacity={0.7}
        />

        {/* AR Plane detection */}
        <ViroARPlaneSelector
          onPlaneSelected={this._onPlaneSelected}
          minHeight={0.5}
          minWidth={0.5}
        >
          {/* Grid helper on detected planes */}
          <ViroQuad
            rotation={[-90, 0, 0]}
            width={2.83}
            height={2.75}
            materials={["grid"]}
            opacity={0.3}
          />
        </ViroARPlaneSelector>

        {/* Instructions when no plane detected */}
        {!planeDetected && (
          <ViroText
            text="Move your device to detect surfaces"
            scale={[0.5, 0.5, 0.5]}
            position={[0, 0, -2]}
            style={styles.instructionText}
          />
        )}

        {/* Wall color overlays */}
        {wallOverlays.map(wall => (
          <ViroQuad
            key={wall.id}
            position={wall.position}
            width={wall.width}
            height={wall.height}
            rotation={wall.rotation}
            materials={["wallMaterial"]}
          />
        ))}

        {/* Furniture items */}
        {furnitureItems.map(item => (
          <ViroNode
            key={item.id}
            position={item.position}
            rotation={item.rotation}
            scale={item.scale}
            dragType="FixedToWorld"
            onDrag={() => {}}
            onClick={() => this.setState({ selectedFurniture: item.id })}
          >
            <Viro3DObject
              source={item.source}
              type="GLB"
              scale={item.scale}
              onLoadStart={() => console.log(`Loading ${item.type}`)}
              onLoadEnd={() => console.log(`Loaded ${item.type}`)}
              onError={(error) => console.log('Error loading model:', error)}
              transformBehaviors={["billboard"]}
              shadowCastingBitMask={1}
            />

            {/* Selection indicator */}
            {this.state.selectedFurniture === item.id && (
              <ViroQuad
                rotation={[-90, 0, 0]}
                width={item.scale[0] * 1.2}
                height={item.scale[2] * 1.2}
                position={[0, -0.01, 0]}
                materials={["selectionRing"]}
              />
            )}

            {/* Dimension label */}
            <ViroText
              text={item.type.toUpperCase()}
              scale={[0.2, 0.2, 0.2]}
              position={[0, 0.5, 0]}
              style={styles.furnitureLabel}
              transformBehaviors={["billboardY"]}
            />
          </ViroNode>
        ))}

        {/* Room measurements overlay */}
        {this.state.measurements && (
          <ViroText
            text={`Room: ${this.state.measurements.width}m × ${this.state.measurements.depth}m`}
            position={[0, 2, -2]}
            scale={[0.3, 0.3, 0.3]}
            style={styles.measurementText}
          />
        )}
      </ViroARScene>
    );
  }
}

// Define materials
ViroMaterials.createMaterials({
  grid: {
    diffuseTexture: require('./assets/grid.png'),
    lightingModel: 'Constant'
  },
  selectionRing: {
    diffuseColor: '#2196F3',
    lightingModel: 'Constant',
    opacity: 0.5
  }
});

const styles = StyleSheet.create({
  instructionText: {
    fontFamily: 'Arial',
    fontSize: 20,
    color: '#ffffff',
    textAlignVertical: 'center',
    textAlign: 'center',
  },
  furnitureLabel: {
    fontFamily: 'Arial',
    fontSize: 12,
    color: '#ffffff',
    textAlignVertical: 'center',
    textAlign: 'center',
  },
  measurementText: {
    fontFamily: 'Arial',
    fontSize: 16,
    color: '#00ff00',
    textAlignVertical: 'center',
    textAlign: 'center',
  }
});

export default ARRoomScene;
```

### Step 3: Main App with UI Controls

**File: `App.js`**
```javascript
import React, { Component } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  SafeAreaView,
  ScrollView,
  Modal
} from 'react-native';
import { ViroARSceneNavigator } from 'react-viro';
import ARRoomScene from './src/ARScene';

class App extends Component {
  constructor(props) {
    super(props);

    this.state = {
      showControls: true,
      currentPalette: 'warm',
      selectedFurniture: null,
      showPaletteMenu: false,
      showFurnitureMenu: false
    };

    this._onExitViro = this._onExitViro.bind(this);
  }

  _onExitViro() {
    // Handle exit from AR view
  }

  _addFurniture = (type) => {
    // This would communicate with ARScene to add furniture
    console.log('Adding furniture:', type);
    this.setState({ showFurnitureMenu: false });
  };

  _changePalette = (palette) => {
    this.setState({
      currentPalette: palette,
      showPaletteMenu: false
    });
    // Update AR scene with new palette
  };

  _takeScreenshot = () => {
    // Capture AR screenshot
    console.log('Taking screenshot');
  };

  _exportDesign = () => {
    // Export room configuration
    console.log('Exporting design');
  };

  render() {
    const { showControls, showPaletteMenu, showFurnitureMenu } = this.state;

    return (
      <View style={styles.container}>
        {/* AR Scene */}
        <ViroARSceneNavigator
          autofocus={true}
          initialScene={{
            scene: ARRoomScene,
          }}
          style={styles.arView}
        />

        {/* Top Control Bar */}
        {showControls && (
          <SafeAreaView style={styles.topBar}>
            <TouchableOpacity
              style={styles.controlButton}
              onPress={() => this.setState({ showPaletteMenu: true })}
            >
              <Text style={styles.buttonText}>🎨 Colors</Text>
            </TouchableOpacity>

            <TouchableOpacity
              style={styles.controlButton}
              onPress={this._takeScreenshot}
            >
              <Text style={styles.buttonText}>📸</Text>
            </TouchableOpacity>

            <TouchableOpacity
              style={styles.controlButton}
              onPress={this._exportDesign}
            >
              <Text style={styles.buttonText}>💾</Text>
            </TouchableOpacity>
          </SafeAreaView>
        )}

        {/* Bottom Furniture Menu */}
        {showControls && (
          <View style={styles.bottomBar}>
            <ScrollView horizontal showsHorizontalScrollIndicator={false}>
              <TouchableOpacity
                style={styles.furnitureButton}
                onPress={() => this._addFurniture('desk')}
              >
                <Text style={styles.furnitureText}>📚 Desk</Text>
              </TouchableOpacity>

              <TouchableOpacity
                style={styles.furnitureButton}
                onPress={() => this._addFurniture('piano')}
              >
                <Text style={styles.furnitureText}>🎹 Piano</Text>
              </TouchableOpacity>

              <TouchableOpacity
                style={styles.furnitureButton}
                onPress={() => this._addFurniture('bed')}
              >
                <Text style={styles.furnitureText}>🛏️ Bed</Text>
              </TouchableOpacity>

              <TouchableOpacity
                style={styles.furnitureButton}
                onPress={() => this._addFurniture('storage')}
              >
                <Text style={styles.furnitureText}>📦 Storage</Text>
              </TouchableOpacity>
            </ScrollView>
          </View>
        )}

        {/* Palette Selection Modal */}
        <Modal
          visible={showPaletteMenu}
          transparent={true}
          animationType="slide"
        >
          <View style={styles.modalContainer}>
            <View style={styles.modalContent}>
              <Text style={styles.modalTitle}>Choose Color Palette</Text>

              <TouchableOpacity
                style={styles.paletteOption}
                onPress={() => this._changePalette('warm')}
              >
                <View style={[styles.colorSwatch, { backgroundColor: '#D4735E' }]} />
                <Text>Warm Focus</Text>
              </TouchableOpacity>

              <TouchableOpacity
                style={styles.paletteOption}
                onPress={() => this._changePalette('soft')}
              >
                <View style={[styles.colorSwatch, { backgroundColor: '#B8C5D6' }]} />
                <Text>Soft Tech</Text>
              </TouchableOpacity>

              <TouchableOpacity
                style={styles.paletteOption}
                onPress={() => this._changePalette('calm')}
              >
                <View style={[styles.colorSwatch, { backgroundColor: '#A8B5A0' }]} />
                <Text>Calm Studio</Text>
              </TouchableOpacity>

              <TouchableOpacity
                style={styles.closeButton}
                onPress={() => this.setState({ showPaletteMenu: false })}
              >
                <Text style={styles.closeButtonText}>Close</Text>
              </TouchableOpacity>
            </View>
          </View>
        </Modal>

        {/* Instructions Overlay */}
        <View style={styles.instructions}>
          <Text style={styles.instructionText}>
            Point camera at floor and walls to detect surfaces
          </Text>
          <Text style={styles.instructionText}>
            Tap furniture below to place in room
          </Text>
        </View>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#000000'
  },
  arView: {
    flex: 1
  },
  topBar: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    flexDirection: 'row',
    justifyContent: 'space-around',
    padding: 16,
    backgroundColor: 'rgba(0,0,0,0.5)'
  },
  controlButton: {
    backgroundColor: 'rgba(255,255,255,0.9)',
    padding: 12,
    borderRadius: 8,
    minWidth: 80,
    alignItems: 'center'
  },
  buttonText: {
    fontSize: 16,
    fontWeight: '600'
  },
  bottomBar: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    backgroundColor: 'rgba(0,0,0,0.7)',
    padding: 16
  },
  furnitureButton: {
    backgroundColor: 'rgba(255,255,255,0.9)',
    padding: 16,
    borderRadius: 12,
    marginHorizontal: 8,
    minWidth: 100,
    alignItems: 'center'
  },
  furnitureText: {
    fontSize: 14,
    fontWeight: '600'
  },
  instructions: {
    position: 'absolute',
    top: 100,
    left: 20,
    right: 20,
    backgroundColor: 'rgba(0,0,0,0.6)',
    padding: 16,
    borderRadius: 8
  },
  instructionText: {
    color: '#ffffff',
    fontSize: 14,
    marginBottom: 4
  },
  modalContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'rgba(0,0,0,0.7)'
  },
  modalContent: {
    backgroundColor: '#ffffff',
    borderRadius: 16,
    padding: 24,
    width: '80%',
    maxWidth: 400
  },
  modalTitle: {
    fontSize: 20,
    fontWeight: 'bold',
    marginBottom: 20,
    textAlign: 'center'
  },
  paletteOption: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0'
  },
  colorSwatch: {
    width: 40,
    height: 40,
    borderRadius: 20,
    marginRight: 16,
    borderWidth: 2,
    borderColor: '#ddd'
  },
  closeButton: {
    backgroundColor: '#2196F3',
    padding: 16,
    borderRadius: 8,
    marginTop: 20,
    alignItems: 'center'
  },
  closeButtonText: {
    color: '#ffffff',
    fontSize: 16,
    fontWeight: '600'
  }
});

export default App;
```

### Step 4: iOS Configuration (Info.plist)

**File: `ios/ARRoomVisualizer/Info.plist`** (Add to existing file)
```xml
<key>NSCameraUsageDescription</key>
<string>This app needs camera access for AR room visualization</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>This app needs photo library access to save AR screenshots</string>

<key>UIRequiredDeviceCapabilities</key>
<array>
    <string>arkit</string>
</array>
```

### Step 5: Android Configuration

**File: `android/app/src/main/AndroidManifest.xml`**
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

<uses-feature android:name="android.hardware.camera.ar" android:required="true"/>

<application>
    <!-- Add ARCore requirement -->
    <meta-data
        android:name="com.google.ar.core"
        android:value="required" />
</application>
```

## Expected Outputs

1. **Real-Time AR Furniture Placement**: Place and move furniture in live camera view
2. **Wall Color Preview**: See paint colors on actual walls
3. **Accurate Scaling**: 1:1 scale furniture models
4. **Surface Detection**: Automatic floor/wall detection
5. **Screenshots/Videos**: Capture AR designs
6. **Shopping List**: Export furniture and paint requirements

## Bonus Challenges

1. **Room Measurement**: Auto-measure room dimensions using AR
2. **Occlusion**: Hide virtual furniture behind real objects
3. **Lighting Estimation**: Match virtual furniture lighting to real room
4. **Multi-User AR**: Share AR sessions between devices
5. **Object Persistence**: Save AR placements between sessions
6. **Custom Furniture**: Users upload their own 3D models
7. **AI Recommendations**: Suggest furniture based on room scan

## Resources

- [ARKit Documentation](https://developer.apple.com/augmented-reality/)
- [ARCore Guides](https://developers.google.com/ar)
- [ViroReact Documentation](https://docs.viromedia.com/)
- [React Native AR Tutorial](https://www.raywenderlich.com/)
- [3D Models (Free)](https://sketchfab.com/features/gltf)

## Success Criteria

- [ ] App detects floors and walls accurately
- [ ] Furniture places at correct scale
- [ ] Color overlays match palette selections
- [ ] Smooth 60 FPS AR performance
- [ ] Screenshots capture AR scene correctly
- [ ] Works on ARKit (iOS 12+) and ARCore (Android 7+)
- [ ] No crashes during extended AR sessions
- [ ] Intuitive UI for non-technical users
