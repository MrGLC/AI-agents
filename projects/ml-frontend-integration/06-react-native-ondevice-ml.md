# Project 06: React Native Mobile App with On-Device ML

## Overview
Build a cross-platform mobile application using React Native with on-device machine learning inference. This project demonstrates running ML models directly on iOS and Android devices using TensorFlow Lite, Core ML, and ML Kit, enabling offline-first ML experiences with privacy-preserving local inference.

## Learning Objectives
- Master React Native for cross-platform mobile development
- Implement on-device ML with TensorFlow Lite and Core ML
- Handle camera and image processing in React Native
- Optimize ML models for mobile deployment
- Manage device resources and battery life
- Build offline-first ML applications
- Implement platform-specific ML optimizations
- Handle permissions and native modules

## Difficulty Level
**Advanced** - Requires understanding of React Native, mobile development, model optimization, and native module integration.

## Technical Stack
- **Framework**: React Native (Expo or bare React Native)
- **ML Libraries**:
  - TensorFlow Lite (cross-platform)
  - Core ML (iOS)
  - ML Kit (Firebase)
  - React Native TensorFlow Lite
- **State Management**: Zustand or Redux Toolkit
- **Navigation**: React Navigation
- **Camera**: React Native Vision Camera
- **Image Processing**: React Native Image Resizer
- **Storage**: AsyncStorage, MMKV
- **Native Modules**: Custom native bridges (if needed)
- **Testing**: Jest, Detox

## Requirements

### Mobile-Specific Requirements
1. Camera integration for real-time inference
2. Gallery image selection and processing
3. Offline model storage and loading
4. Battery-efficient inference
5. Platform-specific optimizations (iOS/Android)
6. Background processing support
7. Model update mechanism
8. Progressive model loading

### ML Model Requirements
1. TensorFlow Lite model support (.tflite)
2. Core ML model support (.mlmodel) - iOS only
3. Model quantization (INT8, FP16)
4. Dynamic model loading
5. Model version management
6. Fallback mechanisms for unsupported devices

### Performance Requirements
1. Inference latency < 100ms on modern devices
2. App bundle size < 50MB
3. Model size < 10MB (compressed)
4. Memory usage < 100MB during inference
5. Battery-efficient processing
6. 60 FPS camera preview

### Features
1. Real-time object detection via camera
2. Image classification from gallery
3. Pose estimation
4. Text recognition (OCR)
5. Offline prediction history
6. Model management (download, update, delete)
7. Performance metrics dashboard
8. Settings for quality/speed tradeoff

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create Expo project (managed workflow)
npx create-expo-app ml-mobile-app --template blank-typescript

cd ml-mobile-app

# Install dependencies
npx expo install expo-camera expo-image-picker expo-file-system
npx expo install expo-asset expo-constants expo-device
npx expo install @react-navigation/native @react-navigation/stack
npx expo install react-native-screens react-native-safe-area-context
npm install zustand
npm install @tensorflow/tfjs @tensorflow/tfjs-react-native
npm install @react-native-async-storage/async-storage
npm install react-native-image-resizer

# For bare React Native (alternative)
# npx react-native init MLMobileApp --template react-native-template-typescript
```

### Step 2: TensorFlow Lite Setup

Create `src/ml/TFLiteManager.ts`:

```typescript
import * as tf from '@tensorflow/tfjs';
import '@tensorflow/tfjs-react-native';
import * as FileSystem from 'expo-file-system';
import { bundleResourceIO, decodeJpeg } from '@tensorflow/tfjs-react-native';
import { Asset } from 'expo-asset';

export interface ModelConfig {
  name: string;
  modelPath: string;
  labelsPath?: string;
  inputShape: number[];
  outputShape: number[];
  quantized: boolean;
}

export interface InferenceResult {
  predictions: Array<{
    label: string;
    confidence: number;
  }>;
  inferenceTime: number;
  modelName: string;
}

class TFLiteManager {
  private models: Map<string, tf.GraphModel> = new Map();
  private labels: Map<string, string[]> = new Map();
  private isInitialized = false;

  async initialize() {
    if (this.isInitialized) return;

    try {
      // Initialize TensorFlow.js for React Native
      await tf.ready();

      // Use WebGL backend if available, otherwise CPU
      const backend = tf.getBackend();
      console.log('TensorFlow.js backend:', backend);

      this.isInitialized = true;
    } catch (error) {
      console.error('Failed to initialize TensorFlow:', error);
      throw error;
    }
  }

  async loadModel(config: ModelConfig): Promise<void> {
    try {
      if (this.models.has(config.name)) {
        console.log(`Model ${config.name} already loaded`);
        return;
      }

      console.log(`Loading model: ${config.name}`);

      // Load model from bundle
      const modelAsset = Asset.fromModule(require(config.modelPath));
      await modelAsset.downloadAsync();

      const model = await tf.loadGraphModel(
        bundleResourceIO(modelAsset.localUri!)
      );

      this.models.set(config.name, model);

      // Load labels if provided
      if (config.labelsPath) {
        const labelsAsset = Asset.fromModule(require(config.labelsPath));
        await labelsAsset.downloadAsync();

        const labelsContent = await FileSystem.readAsStringAsync(
          labelsAsset.localUri!
        );
        const labels = labelsContent.split('\n').filter((l) => l.trim());
        this.labels.set(config.name, labels);
      }

      console.log(`Model ${config.name} loaded successfully`);

      // Warmup
      await this.warmup(config.name, config.inputShape);
    } catch (error) {
      console.error(`Failed to load model ${config.name}:`, error);
      throw error;
    }
  }

  async warmup(modelName: string, inputShape: number[]): Promise<void> {
    const model = this.models.get(modelName);
    if (!model) return;

    console.log(`Warming up model: ${modelName}`);

    const warmupTensor = tf.zeros(inputShape);
    const output = model.predict(warmupTensor) as tf.Tensor;

    // Wait for completion
    await output.data();

    // Cleanup
    warmupTensor.dispose();
    output.dispose();

    console.log(`Model ${modelName} warmed up`);
  }

  async predict(
    modelName: string,
    imageUri: string,
    topK: number = 5
  ): Promise<InferenceResult> {
    const model = this.models.get(modelName);
    if (!model) {
      throw new Error(`Model ${modelName} not loaded`);
    }

    const startTime = Date.now();

    try {
      // Read and decode image
      const imageAssetPath = await FileSystem.readAsStringAsync(imageUri, {
        encoding: FileSystem.EncodingType.Base64,
      });

      const imageData = Uint8Array.from(atob(imageAssetPath), (c) =>
        c.charCodeAt(0)
      );

      const imageTensor = decodeJpeg(imageData);

      // Preprocess
      const preprocessed = this.preprocessImage(imageTensor);

      // Predict
      const output = model.predict(preprocessed) as tf.Tensor;
      const outputData = await output.data();

      // Postprocess
      const predictions = this.postprocess(
        modelName,
        Array.from(outputData),
        topK
      );

      const inferenceTime = Date.now() - startTime;

      // Cleanup tensors
      imageTensor.dispose();
      preprocessed.dispose();
      output.dispose();

      return {
        predictions,
        inferenceTime,
        modelName,
      };
    } catch (error) {
      console.error('Prediction error:', error);
      throw error;
    }
  }

  async predictFromCamera(
    modelName: string,
    imageData: ImageData,
    topK: number = 5
  ): Promise<InferenceResult> {
    const model = this.models.get(modelName);
    if (!model) {
      throw new Error(`Model ${modelName} not loaded`);
    }

    const startTime = Date.now();

    try {
      // Convert ImageData to tensor
      const imageTensor = tf.browser.fromPixels(imageData);

      // Preprocess
      const preprocessed = this.preprocessImage(imageTensor);

      // Predict
      const output = model.predict(preprocessed) as tf.Tensor;
      const outputData = await output.data();

      // Postprocess
      const predictions = this.postprocess(
        modelName,
        Array.from(outputData),
        topK
      );

      const inferenceTime = Date.now() - startTime;

      // Cleanup
      imageTensor.dispose();
      preprocessed.dispose();
      output.dispose();

      return {
        predictions,
        inferenceTime,
        modelName,
      };
    } catch (error) {
      console.error('Camera prediction error:', error);
      throw error;
    }
  }

  private preprocessImage(imageTensor: tf.Tensor3D): tf.Tensor4D {
    // Resize to model input size (224x224 for MobileNet)
    const resized = tf.image.resizeBilinear(imageTensor, [224, 224]);

    // Normalize to [-1, 1] or [0, 1] depending on model
    const normalized = resized.div(127.5).sub(1);

    // Add batch dimension
    const batched = normalized.expandDims(0);

    // Cleanup intermediate tensors
    resized.dispose();
    normalized.dispose();

    return batched;
  }

  private postprocess(
    modelName: string,
    output: number[],
    topK: number
  ): Array<{ label: string; confidence: number }> {
    const labels = this.labels.get(modelName);

    // Get top K predictions
    const indexed = output.map((score, index) => ({ score, index }));
    const sorted = indexed.sort((a, b) => b.score - a.score);
    const topPredictions = sorted.slice(0, topK);

    return topPredictions.map((pred) => ({
      label: labels ? labels[pred.index] : `Class ${pred.index}`,
      confidence: pred.score,
    }));
  }

  getModelInfo(modelName: string) {
    const model = this.models.get(modelName);
    if (!model) return null;

    return {
      name: modelName,
      loaded: true,
      inputShape: model.inputs[0].shape,
      outputShape: model.outputs[0].shape,
    };
  }

  async unloadModel(modelName: string): Promise<void> {
    const model = this.models.get(modelName);
    if (model) {
      model.dispose();
      this.models.delete(modelName);
      this.labels.delete(modelName);
      console.log(`Model ${modelName} unloaded`);
    }
  }

  getMemoryInfo() {
    return tf.memory();
  }

  dispose() {
    this.models.forEach((model) => model.dispose());
    this.models.clear();
    this.labels.clear();
  }
}

export const tfLiteManager = new TFLiteManager();
```

### Step 3: State Management with Zustand

Create `src/store/mlStore.ts`:

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

export interface Prediction {
  id: string;
  result: {
    label: string;
    confidence: number;
  }[];
  inferenceTime: number;
  timestamp: number;
  imageUri?: string;
}

interface MLState {
  // Models
  modelsLoaded: boolean;
  currentModel: string;
  availableModels: string[];

  // Predictions
  predictions: Prediction[];
  currentPrediction: Prediction | null;

  // UI State
  isProcessing: boolean;
  error: string | null;

  // Settings
  settings: {
    autoPredict: boolean;
    saveHistory: boolean;
    modelQuality: 'high' | 'medium' | 'low';
  };

  // Actions
  setModelsLoaded: (loaded: boolean) => void;
  setCurrentModel: (model: string) => void;
  addPrediction: (prediction: Prediction) => void;
  setCurrentPrediction: (prediction: Prediction | null) => void;
  setProcessing: (processing: boolean) => void;
  setError: (error: string | null) => void;
  clearHistory: () => void;
  updateSettings: (settings: Partial<MLState['settings']>) => void;
}

export const useMLStore = create<MLState>()(
  persist(
    (set) => ({
      // Initial state
      modelsLoaded: false,
      currentModel: 'mobilenet',
      availableModels: ['mobilenet', 'efficientnet', 'custom'],
      predictions: [],
      currentPrediction: null,
      isProcessing: false,
      error: null,
      settings: {
        autoPredict: false,
        saveHistory: true,
        modelQuality: 'medium',
      },

      // Actions
      setModelsLoaded: (loaded) => set({ modelsLoaded: loaded }),

      setCurrentModel: (model) => set({ currentModel: model }),

      addPrediction: (prediction) =>
        set((state) => ({
          predictions: [prediction, ...state.predictions].slice(0, 100),
          currentPrediction: prediction,
        })),

      setCurrentPrediction: (prediction) =>
        set({ currentPrediction: prediction }),

      setProcessing: (processing) => set({ isProcessing: processing }),

      setError: (error) => set({ error }),

      clearHistory: () => set({ predictions: [], currentPrediction: null }),

      updateSettings: (newSettings) =>
        set((state) => ({
          settings: { ...state.settings, ...newSettings },
        })),
    }),
    {
      name: 'ml-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({
        predictions: state.predictions,
        settings: state.settings,
      }),
    }
  )
);
```

### Step 4: Camera Component

Create `src/components/CameraView.tsx`:

```typescript
import React, { useState, useRef, useEffect } from 'react';
import { View, Text, TouchableOpacity, StyleSheet, Platform } from 'react-native';
import { Camera, CameraType } from 'expo-camera';
import { tfLiteManager } from '../ml/TFLiteManager';
import { useMLStore } from '../store/mlStore';

export default function CameraView() {
  const [hasPermission, setHasPermission] = useState<boolean | null>(null);
  const [type, setType] = useState(CameraType.back);
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const cameraRef = useRef<Camera>(null);

  const { addPrediction, setProcessing, setError, settings } = useMLStore();

  useEffect(() => {
    (async () => {
      const { status } = await Camera.requestCameraPermissionsAsync();
      setHasPermission(status === 'granted');
    })();
  }, []);

  const captureAndAnalyze = async () => {
    if (!cameraRef.current || isAnalyzing) return;

    setIsAnalyzing(true);
    setProcessing(true);
    setError(null);

    try {
      const photo = await cameraRef.current.takePictureAsync({
        quality: 0.8,
        base64: false,
        skipProcessing: true,
      });

      // Run inference
      const result = await tfLiteManager.predict('mobilenet', photo.uri, 5);

      // Save prediction
      const prediction = {
        id: Date.now().toString(),
        result: result.predictions,
        inferenceTime: result.inferenceTime,
        timestamp: Date.now(),
        imageUri: photo.uri,
      };

      addPrediction(prediction);
    } catch (error) {
      console.error('Analysis error:', error);
      setError(error instanceof Error ? error.message : 'Analysis failed');
    } finally {
      setIsAnalyzing(false);
      setProcessing(false);
    }
  };

  if (hasPermission === null) {
    return <View />;
  }

  if (hasPermission === false) {
    return (
      <View style={styles.container}>
        <Text>No access to camera</Text>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <Camera style={styles.camera} type={type} ref={cameraRef}>
        <View style={styles.overlay}>
          <View style={styles.topBar}>
            <TouchableOpacity
              style={styles.flipButton}
              onPress={() => {
                setType(
                  type === CameraType.back ? CameraType.front : CameraType.back
                );
              }}
            >
              <Text style={styles.buttonText}>Flip</Text>
            </TouchableOpacity>
          </View>

          <View style={styles.bottomBar}>
            <TouchableOpacity
              style={[styles.captureButton, isAnalyzing && styles.captureButtonDisabled]}
              onPress={captureAndAnalyze}
              disabled={isAnalyzing}
            >
              <View style={styles.captureButtonInner} />
            </TouchableOpacity>
          </View>

          {isAnalyzing && (
            <View style={styles.analyzingOverlay}>
              <Text style={styles.analyzingText}>Analyzing...</Text>
            </View>
          )}
        </View>
      </Camera>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  camera: {
    flex: 1,
  },
  overlay: {
    flex: 1,
    backgroundColor: 'transparent',
  },
  topBar: {
    flexDirection: 'row',
    justifyContent: 'flex-end',
    padding: 20,
    paddingTop: Platform.OS === 'ios' ? 60 : 20,
  },
  bottomBar: {
    position: 'absolute',
    bottom: 40,
    left: 0,
    right: 0,
    alignItems: 'center',
  },
  flipButton: {
    backgroundColor: 'rgba(0, 0, 0, 0.5)',
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 20,
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  captureButton: {
    width: 70,
    height: 70,
    borderRadius: 35,
    backgroundColor: 'white',
    justifyContent: 'center',
    alignItems: 'center',
    borderWidth: 4,
    borderColor: '#fff',
  },
  captureButtonDisabled: {
    opacity: 0.5,
  },
  captureButtonInner: {
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#007AFF',
  },
  analyzingOverlay: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    bottom: 0,
    backgroundColor: 'rgba(0, 0, 0, 0.7)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  analyzingText: {
    color: 'white',
    fontSize: 20,
    fontWeight: '600',
  },
});
```

### Step 5: Results Display Component

Create `src/components/PredictionResults.tsx`:

```typescript
import React from 'react';
import { View, Text, StyleSheet, ScrollView, Image } from 'react-native';
import { useMLStore } from '../store/mlStore';

export default function PredictionResults() {
  const { currentPrediction } = useMLStore();

  if (!currentPrediction) {
    return (
      <View style={styles.emptyContainer}>
        <Text style={styles.emptyText}>No prediction yet</Text>
        <Text style={styles.emptySubtext}>
          Capture an image to see results
        </Text>
      </View>
    );
  }

  const topPrediction = currentPrediction.result[0];

  return (
    <ScrollView style={styles.container}>
      {currentPrediction.imageUri && (
        <Image
          source={{ uri: currentPrediction.imageUri }}
          style={styles.image}
          resizeMode="cover"
        />
      )}

      <View style={styles.mainResult}>
        <Text style={styles.mainLabel}>{topPrediction.label}</Text>
        <Text style={styles.mainConfidence}>
          {(topPrediction.confidence * 100).toFixed(1)}% confident
        </Text>
        <Text style={styles.inferenceTime}>
          Inference: {currentPrediction.inferenceTime}ms
        </Text>
      </View>

      <View style={styles.allResults}>
        <Text style={styles.sectionTitle}>Top Predictions</Text>
        {currentPrediction.result.map((pred, index) => (
          <View key={index} style={styles.predictionRow}>
            <Text style={styles.rank}>{index + 1}</Text>
            <Text style={styles.label}>{pred.label}</Text>
            <View style={styles.confidenceBar}>
              <View
                style={[
                  styles.confidenceFill,
                  { width: `${pred.confidence * 100}%` },
                ]}
              />
            </View>
            <Text style={styles.confidence}>
              {(pred.confidence * 100).toFixed(1)}%
            </Text>
          </View>
        ))}
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  emptyContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  emptyText: {
    fontSize: 20,
    fontWeight: '600',
    color: '#666',
    marginBottom: 8,
  },
  emptySubtext: {
    fontSize: 14,
    color: '#999',
  },
  image: {
    width: '100%',
    height: 300,
    backgroundColor: '#eee',
  },
  mainResult: {
    backgroundColor: 'white',
    padding: 20,
    marginBottom: 16,
  },
  mainLabel: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#333',
    marginBottom: 8,
  },
  mainConfidence: {
    fontSize: 18,
    color: '#007AFF',
    marginBottom: 4,
  },
  inferenceTime: {
    fontSize: 14,
    color: '#999',
  },
  allResults: {
    backgroundColor: 'white',
    padding: 20,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: '600',
    marginBottom: 16,
    color: '#333',
  },
  predictionRow: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 12,
  },
  rank: {
    width: 24,
    fontSize: 14,
    fontWeight: '600',
    color: '#666',
  },
  label: {
    flex: 1,
    fontSize: 14,
    color: '#333',
  },
  confidenceBar: {
    width: 60,
    height: 6,
    backgroundColor: '#eee',
    borderRadius: 3,
    marginHorizontal: 12,
    overflow: 'hidden',
  },
  confidenceFill: {
    height: '100%',
    backgroundColor: '#007AFF',
  },
  confidence: {
    width: 50,
    fontSize: 12,
    color: '#666',
    textAlign: 'right',
  },
});
```

### Step 6: Main App Component

Create `App.tsx`:

```typescript
import React, { useEffect, useState } from 'react';
import { View, Text, StyleSheet, ActivityIndicator } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import { tfLiteManager } from './src/ml/TFLiteManager';
import { useMLStore } from './src/store/mlStore';
import CameraView from './src/components/CameraView';
import PredictionResults from './src/components/PredictionResults';

const Stack = createStackNavigator();

function LoadingScreen() {
  return (
    <View style={styles.loadingContainer}>
      <ActivityIndicator size="large" color="#007AFF" />
      <Text style={styles.loadingText}>Loading ML Models...</Text>
    </View>
  );
}

export default function App() {
  const [isReady, setIsReady] = useState(false);
  const { setModelsLoaded, setError } = useMLStore();

  useEffect(() => {
    async function initializeML() {
      try {
        await tfLiteManager.initialize();

        // Load models
        await tfLiteManager.loadModel({
          name: 'mobilenet',
          modelPath: './assets/models/mobilenet_v2.tflite',
          labelsPath: './assets/models/labels.txt',
          inputShape: [1, 224, 224, 3],
          outputShape: [1, 1000],
          quantized: true,
        });

        setModelsLoaded(true);
        setIsReady(true);
      } catch (error) {
        console.error('Failed to initialize ML:', error);
        setError(error instanceof Error ? error.message : 'Initialization failed');
        setIsReady(true); // Still show app
      }
    }

    initializeML();

    return () => {
      tfLiteManager.dispose();
    };
  }, []);

  if (!isReady) {
    return <LoadingScreen />;
  }

  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen
          name="Camera"
          component={CameraView}
          options={{ headerShown: false }}
        />
        <Stack.Screen
          name="Results"
          component={PredictionResults}
          options={{ title: 'Prediction Results' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

const styles = StyleSheet.create({
  loadingContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#fff',
  },
  loadingText: {
    marginTop: 16,
    fontSize: 16,
    color: '#666',
  },
});
```

## Expected Outputs

1. **Cross-Platform Mobile App**:
   - iOS and Android support
   - Native performance
   - Offline-first architecture

2. **On-Device ML**:
   - Real-time camera inference
   - Gallery image classification
   - < 100ms latency

3. **User Experience**:
   - Smooth 60 FPS camera
   - Instant results
   - Offline functionality
   - Battery efficient

4. **Production Features**:
   - Model management
   - Prediction history
   - Performance metrics
   - Error handling

## Bonus Challenges

1. **Custom Model**: Train and deploy custom TFLite model
2. **Object Detection**: Add real-time object detection
3. **Pose Estimation**: Implement pose tracking
4. **OCR**: Add text recognition
5. **Model Updates**: Over-the-air model updates
6. **Cloud Sync**: Sync predictions to backend
7. **Augmented Reality**: AR overlays for predictions
8. **Background Processing**: Background ML inference

## Resources

- [React Native Documentation](https://reactnative.dev/)
- [Expo Documentation](https://docs.expo.dev/)
- [TensorFlow Lite](https://www.tensorflow.org/lite)
- [React Native Vision Camera](https://react-native-vision-camera.com/)
- [Zustand](https://github.com/pmndrs/zustand)

## Success Criteria

### Functionality (40%)
- [ ] Camera integration working
- [ ] ML inference functional
- [ ] Gallery image processing
- [ ] Offline support
- [ ] Prediction history

### Performance (30%)
- [ ] Inference < 100ms
- [ ] 60 FPS camera
- [ ] App size < 50MB
- [ ] Memory efficient
- [ ] Battery friendly

### Code Quality (20%)
- [ ] TypeScript typing
- [ ] Clean architecture
- [ ] Error handling
- [ ] State management
- [ ] Native optimization

### User Experience (10%)
- [ ] Intuitive UI
- [ ] Smooth animations
- [ ] Clear feedback
- [ ] Responsive design
- [ ] Accessibility
