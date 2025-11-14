# Project 10: Electron Desktop App with Local Model Inference

## Overview
Build a cross-platform desktop application using Electron with local ML inference capabilities. This project demonstrates running ML models entirely on the user's machine with native desktop features, file system access, system tray integration, and offline-first ML without requiring internet connectivity or external APIs.

## Learning Objectives
- Master Electron for cross-platform desktop development
- Implement IPC (Inter-Process Communication) between main and renderer
- Run ML inference in the main process for better performance
- Utilize native Node.js modules for ML (TensorFlow Node.js)
- Implement file system operations for model management
- Create native desktop UI with system integration
- Handle auto-updates and application packaging
- Optimize Electron app performance and bundle size

## Difficulty Level
**Advanced** - Requires understanding of Electron architecture, Node.js, native modules, and desktop application development.

## Technical Stack
- **Framework**: Electron 28+
- **Renderer**: React with TypeScript
- **ML Backend**: TensorFlow Node.js (in main process)
- **IPC**: Electron IPC (contextBridge)
- **State Management**: Zustand
- **Build Tool**: Electron Forge or Electron Builder
- **UI**: Electron-React-Boilerplate or custom setup
- **Native Modules**: @tensorflow/tfjs-node
- **File System**: Node.js fs/promises
- **Auto-Update**: Electron-updater
- **Testing**: Spectron (deprecated, use Playwright)

## Requirements

### Desktop Features
1. Native menu bar and system tray
2. File drag-and-drop support
3. Native notifications
4. Keyboard shortcuts
5. Multiple window management
6. Context menus
7. File system dialogs
8. Operating system integration

### ML Features
1. Local model storage and loading
2. CPU/GPU inference support
3. Batch file processing
4. Model versioning and updates
5. Export predictions to CSV/JSON
6. Model performance benchmarking
7. Background processing
8. Progress tracking for long operations

### Application Features
1. Settings panel with preferences
2. Model management interface
3. Prediction history database (SQLite)
4. Dark/light theme support
5. Multi-language support (i18n)
6. Auto-update mechanism
7. Crash reporting
8. Analytics (privacy-focused)

### Performance Requirements
1. App startup time < 3 seconds
2. Model loading < 5 seconds
3. Inference latency < 100ms
4. App bundle size < 200MB
5. Memory usage < 500MB
6. Support for models up to 500MB

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create Electron Forge app
npx create-electron-app ml-desktop-app --template=webpack-typescript

cd ml-desktop-app

# Install dependencies
npm install @tensorflow/tfjs-node
npm install electron-store # For settings persistence
npm install sqlite3 better-sqlite3 # For local database
npm install electron-updater # For auto-updates
npm install react react-dom
npm install @types/react @types/react-dom
npm install zustand
npm install recharts # For visualizations
npm install electron-log # For logging
```

### Step 2: Main Process Setup

Create `src/main/main.ts`:

```typescript
import { app, BrowserWindow, ipcMain, Menu, Tray, dialog, shell } from 'electron';
import path from 'path';
import { autoUpdater } from 'electron-updater';
import log from 'electron-log';
import Store from 'electron-store';
import { MLService } from './services/MLService';
import { DatabaseService } from './services/DatabaseService';

// Configure logging
log.transports.file.level = 'info';
autoUpdater.logger = log;

// Persistent storage
const store = new Store();

// Services
const mlService = new MLService();
const dbService = new DatabaseService();

let mainWindow: BrowserWindow | null = null;
let tray: Tray | null = null;

async function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: false,
    },
    title: 'ML Desktop App',
    icon: path.join(__dirname, 'assets/icon.png'),
  });

  // Load app
  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:3000');
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(path.join(__dirname, '../renderer/index.html'));
  }

  // Window events
  mainWindow.on('closed', () => {
    mainWindow = null;
  });

  mainWindow.on('minimize', (event: Event) => {
    if (store.get('minimizeToTray', true)) {
      event.preventDefault();
      mainWindow?.hide();
    }
  });

  // Create application menu
  createMenu();

  // Create system tray
  createTray();
}

function createMenu() {
  const template: Electron.MenuItemConstructorOptions[] = [
    {
      label: 'File',
      submenu: [
        {
          label: 'Load Model',
          accelerator: 'CmdOrCtrl+O',
          click: async () => {
            const result = await dialog.showOpenDialog({
              properties: ['openFile'],
              filters: [
                { name: 'TensorFlow Models', extensions: ['json', 'tflite'] },
                { name: 'All Files', extensions: ['*'] },
              ],
            });

            if (!result.canceled && result.filePaths.length > 0) {
              mainWindow?.webContents.send('model-file-selected', result.filePaths[0]);
            }
          },
        },
        {
          label: 'Import Images',
          accelerator: 'CmdOrCtrl+I',
          click: async () => {
            const result = await dialog.showOpenDialog({
              properties: ['openFile', 'multiSelections'],
              filters: [
                {
                  name: 'Images',
                  extensions: ['jpg', 'jpeg', 'png', 'bmp', 'gif'],
                },
              ],
            });

            if (!result.canceled && result.filePaths.length > 0) {
              mainWindow?.webContents.send('images-selected', result.filePaths);
            }
          },
        },
        { type: 'separator' },
        {
          label: 'Export Results',
          accelerator: 'CmdOrCtrl+E',
          click: () => {
            mainWindow?.webContents.send('export-results');
          },
        },
        { type: 'separator' },
        { role: 'quit' },
      ],
    },
    {
      label: 'Edit',
      submenu: [
        { role: 'undo' },
        { role: 'redo' },
        { type: 'separator' },
        { role: 'cut' },
        { role: 'copy' },
        { role: 'paste' },
        { role: 'selectAll' },
      ],
    },
    {
      label: 'View',
      submenu: [
        { role: 'reload' },
        { role: 'forceReload' },
        { role: 'toggleDevTools' },
        { type: 'separator' },
        { role: 'resetZoom' },
        { role: 'zoomIn' },
        { role: 'zoomOut' },
        { type: 'separator' },
        { role: 'togglefullscreen' },
      ],
    },
    {
      label: 'Models',
      submenu: [
        {
          label: 'Manage Models',
          click: () => {
            mainWindow?.webContents.send('open-models-manager');
          },
        },
        {
          label: 'Check for Updates',
          click: () => {
            autoUpdater.checkForUpdatesAndNotify();
          },
        },
      ],
    },
    {
      label: 'Help',
      submenu: [
        {
          label: 'Documentation',
          click: async () => {
            await shell.openExternal('https://docs.example.com');
          },
        },
        {
          label: 'Report Issue',
          click: async () => {
            await shell.openExternal('https://github.com/example/issues');
          },
        },
        { type: 'separator' },
        {
          label: 'About',
          click: () => {
            dialog.showMessageBox({
              title: 'About ML Desktop App',
              message: 'ML Desktop App v1.0.0',
              detail: 'A desktop application for local ML inference',
            });
          },
        },
      ],
    },
  ];

  const menu = Menu.buildFromTemplate(template);
  Menu.setApplicationMenu(menu);
}

function createTray() {
  tray = new Tray(path.join(__dirname, 'assets/tray-icon.png'));

  const contextMenu = Menu.buildFromTemplate([
    {
      label: 'Show App',
      click: () => {
        mainWindow?.show();
      },
    },
    {
      label: 'Quick Prediction',
      click: () => {
        mainWindow?.show();
        mainWindow?.webContents.send('open-quick-prediction');
      },
    },
    { type: 'separator' },
    {
      label: 'Quit',
      click: () => {
        app.quit();
      },
    },
  ]);

  tray.setToolTip('ML Desktop App');
  tray.setContextMenu(contextMenu);

  tray.on('click', () => {
    mainWindow?.show();
  });
}

// IPC Handlers
ipcMain.handle('load-model', async (event, modelPath: string) => {
  try {
    log.info('Loading model:', modelPath);
    const result = await mlService.loadModel(modelPath);
    return { success: true, modelInfo: result };
  } catch (error) {
    log.error('Failed to load model:', error);
    return { success: false, error: (error as Error).message };
  }
});

ipcMain.handle('predict', async (event, modelId: string, inputData: any) => {
  try {
    const result = await mlService.predict(modelId, inputData);

    // Save to database
    await dbService.savePrediction({
      modelId,
      input: inputData,
      output: result.output,
      confidence: result.confidence,
      timestamp: Date.now(),
    });

    return { success: true, ...result };
  } catch (error) {
    log.error('Prediction failed:', error);
    return { success: false, error: (error as Error).message };
  }
});

ipcMain.handle('batch-predict', async (event, modelId: string, imagePaths: string[]) => {
  try {
    const results = [];
    const total = imagePaths.length;

    for (let i = 0; i < imagePaths.length; i++) {
      const imagePath = imagePaths[i];

      // Send progress update
      mainWindow?.webContents.send('batch-progress', {
        current: i + 1,
        total,
        percentage: ((i + 1) / total) * 100,
      });

      const result = await mlService.predictFromImage(modelId, imagePath);
      results.push({
        imagePath,
        ...result,
      });

      // Save to database
      await dbService.savePrediction({
        modelId,
        input: { imagePath },
        output: result.output,
        confidence: result.confidence,
        timestamp: Date.now(),
      });
    }

    return { success: true, results };
  } catch (error) {
    log.error('Batch prediction failed:', error);
    return { success: false, error: (error as Error).message };
  }
});

ipcMain.handle('get-history', async (event, limit: number = 100) => {
  try {
    const history = await dbService.getHistory(limit);
    return { success: true, history };
  } catch (error) {
    return { success: false, error: (error as Error).message };
  }
});

ipcMain.handle('export-results', async (event, format: 'csv' | 'json') => {
  try {
    const { filePath } = await dialog.showSaveDialog({
      defaultPath: `predictions_${Date.now()}.${format}`,
      filters: [
        { name: format.toUpperCase(), extensions: [format] },
        { name: 'All Files', extensions: ['*'] },
      ],
    });

    if (!filePath) {
      return { success: false, error: 'Export cancelled' };
    }

    const history = await dbService.getHistory(1000);
    await dbService.exportResults(history, filePath, format);

    return { success: true, filePath };
  } catch (error) {
    return { success: false, error: (error as Error).message };
  }
});

ipcMain.handle('get-models', async () => {
  try {
    const models = await mlService.getLoadedModels();
    return { success: true, models };
  } catch (error) {
    return { success: false, error: (error as Error).message };
  }
});

ipcMain.handle('get-settings', async () => {
  return store.store;
});

ipcMain.handle('update-settings', async (event, settings: any) => {
  store.set(settings);
  return { success: true };
});

// App lifecycle
app.whenReady().then(async () => {
  await createWindow();

  // Initialize services
  await dbService.initialize();

  // Check for updates
  if (process.env.NODE_ENV === 'production') {
    autoUpdater.checkForUpdatesAndNotify();
  }

  app.on('activate', () => {
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

// Auto-updater events
autoUpdater.on('update-available', () => {
  log.info('Update available');
  mainWindow?.webContents.send('update-available');
});

autoUpdater.on('update-downloaded', () => {
  log.info('Update downloaded');
  dialog
    .showMessageBox({
      type: 'info',
      title: 'Update Ready',
      message: 'A new version has been downloaded. Restart to apply?',
      buttons: ['Restart', 'Later'],
    })
    .then((result) => {
      if (result.response === 0) {
        autoUpdater.quitAndInstall();
      }
    });
});
```

### Step 3: ML Service

Create `src/main/services/MLService.ts`:

```typescript
import * as tf from '@tensorflow/tfjs-node';
import * as fs from 'fs/promises';
import * as path from 'path';
import * as jpeg from 'jpeg-js';
import * as png from 'pngjs';
import log from 'electron-log';

export interface ModelInfo {
  id: string;
  name: string;
  path: string;
  inputShape: number[];
  outputShape: number[];
  loadedAt: number;
}

export class MLService {
  private models: Map<string, tf.GraphModel | tf.LayersModel> = new Map();
  private modelInfo: Map<string, ModelInfo> = new Map();

  async loadModel(modelPath: string): Promise<ModelInfo> {
    try {
      log.info(`Loading model from: ${modelPath}`);

      // Load TensorFlow model
      const model = await tf.loadGraphModel(`file://${modelPath}`);

      const modelId = path.basename(modelPath, path.extname(modelPath));

      const info: ModelInfo = {
        id: modelId,
        name: modelId,
        path: modelPath,
        inputShape: model.inputs[0].shape || [],
        outputShape: model.outputs[0].shape || [],
        loadedAt: Date.now(),
      };

      this.models.set(modelId, model);
      this.modelInfo.set(modelId, info);

      // Warmup
      const warmupInput = tf.zeros(info.inputShape);
      const warmupOutput = model.predict(warmupInput) as tf.Tensor;
      warmupInput.dispose();
      warmupOutput.dispose();

      log.info(`Model ${modelId} loaded successfully`);

      return info;
    } catch (error) {
      log.error('Failed to load model:', error);
      throw error;
    }
  }

  async predict(modelId: string, inputData: number[]): Promise<any> {
    const model = this.models.get(modelId);
    if (!model) {
      throw new Error(`Model ${modelId} not loaded`);
    }

    const info = this.modelInfo.get(modelId)!;

    const startTime = Date.now();

    try {
      // Create input tensor
      const inputTensor = tf.tensor(inputData, info.inputShape);

      // Run prediction
      const outputTensor = model.predict(inputTensor) as tf.Tensor;
      const outputData = await outputTensor.data();

      const inferenceTime = Date.now() - startTime;

      // Cleanup
      inputTensor.dispose();
      outputTensor.dispose();

      return {
        output: Array.from(outputData),
        inferenceTime,
        modelId,
        confidence: Math.max(...Array.from(outputData)),
      };
    } catch (error) {
      log.error('Prediction failed:', error);
      throw error;
    }
  }

  async predictFromImage(modelId: string, imagePath: string): Promise<any> {
    const model = this.models.get(modelId);
    if (!model) {
      throw new Error(`Model ${modelId} not loaded`);
    }

    try {
      // Read image file
      const imageBuffer = await fs.readFile(imagePath);

      // Decode image based on extension
      let imageData: { data: Uint8Array; width: number; height: number };

      if (imagePath.toLowerCase().endsWith('.png')) {
        const pngData = png.PNG.sync.read(imageBuffer);
        imageData = {
          data: pngData.data,
          width: pngData.width,
          height: pngData.height,
        };
      } else {
        const jpegData = jpeg.decode(imageBuffer);
        imageData = jpegData;
      }

      // Convert to tensor
      const imageTensor = tf.browser.fromPixels({
        data: imageData.data,
        width: imageData.width,
        height: imageData.height,
      } as any);

      // Preprocess
      const resized = tf.image.resizeBilinear(imageTensor, [224, 224]);
      const normalized = resized.div(255.0);
      const batched = normalized.expandDims(0);

      // Predict
      const startTime = Date.now();
      const outputTensor = model.predict(batched) as tf.Tensor;
      const outputData = await outputTensor.data();
      const inferenceTime = Date.now() - startTime;

      // Cleanup
      imageTensor.dispose();
      resized.dispose();
      normalized.dispose();
      batched.dispose();
      outputTensor.dispose();

      return {
        output: Array.from(outputData),
        inferenceTime,
        modelId,
        confidence: Math.max(...Array.from(outputData)),
      };
    } catch (error) {
      log.error('Image prediction failed:', error);
      throw error;
    }
  }

  getLoadedModels(): ModelInfo[] {
    return Array.from(this.modelInfo.values());
  }

  unloadModel(modelId: string): void {
    const model = this.models.get(modelId);
    if (model) {
      model.dispose();
      this.models.delete(modelId);
      this.modelInfo.delete(modelId);
      log.info(`Model ${modelId} unloaded`);
    }
  }

  getMemoryUsage() {
    return tf.memory();
  }

  dispose() {
    this.models.forEach((model) => model.dispose());
    this.models.clear();
    this.modelInfo.clear();
  }
}
```

### Step 4: Preload Script

Create `src/main/preload.ts`:

```typescript
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // Model operations
  loadModel: (modelPath: string) => ipcRenderer.invoke('load-model', modelPath),
  getModels: () => ipcRenderer.invoke('get-models'),

  // Predictions
  predict: (modelId: string, inputData: any) =>
    ipcRenderer.invoke('predict', modelId, inputData),
  batchPredict: (modelId: string, imagePaths: string[]) =>
    ipcRenderer.invoke('batch-predict', modelId, imagePaths),

  // History
  getHistory: (limit?: number) => ipcRenderer.invoke('get-history', limit),
  exportResults: (format: 'csv' | 'json') =>
    ipcRenderer.invoke('export-results', format),

  // Settings
  getSettings: () => ipcRenderer.invoke('get-settings'),
  updateSettings: (settings: any) => ipcRenderer.invoke('update-settings', settings),

  // Event listeners
  onModelFileSelected: (callback: (path: string) => void) => {
    ipcRenderer.on('model-file-selected', (event, path) => callback(path));
  },
  onImagesSelected: (callback: (paths: string[]) => void) => {
    ipcRenderer.on('images-selected', (event, paths) => callback(paths));
  },
  onBatchProgress: (callback: (progress: any) => void) => {
    ipcRenderer.on('batch-progress', (event, progress) => callback(progress));
  },
  onUpdateAvailable: (callback: () => void) => {
    ipcRenderer.on('update-available', callback);
  },
});
```

### Step 5: Renderer (React App)

Create `src/renderer/App.tsx`:

```typescript
import React, { useState, useEffect } from 'react';
import { ModelManager } from './components/ModelManager';
import { PredictionPanel } from './components/PredictionPanel';
import { HistoryPanel } from './components/HistoryPanel';

declare global {
  interface Window {
    electronAPI: any;
  }
}

function App() {
  const [activeTab, setActiveTab] = useState<'predict' | 'history' | 'models'>('predict');
  const [models, setModels] = useState([]);
  const [selectedModel, setSelectedModel] = useState<string | null>(null);

  useEffect(() => {
    loadModels();

    // Setup event listeners
    window.electronAPI.onUpdateAvailable(() => {
      alert('A new update is available!');
    });
  }, []);

  const loadModels = async () => {
    const result = await window.electronAPI.getModels();
    if (result.success) {
      setModels(result.models);
      if (result.models.length > 0 && !selectedModel) {
        setSelectedModel(result.models[0].id);
      }
    }
  };

  return (
    <div className="h-screen flex flex-col bg-gray-100">
      {/* Header */}
      <header className="bg-white border-b px-6 py-4">
        <h1 className="text-2xl font-bold">ML Desktop App</h1>
      </header>

      {/* Navigation */}
      <nav className="bg-white border-b">
        <div className="flex space-x-1 px-6">
          <button
            onClick={() => setActiveTab('predict')}
            className={`px-4 py-2 ${
              activeTab === 'predict'
                ? 'border-b-2 border-blue-500 text-blue-600'
                : 'text-gray-600'
            }`}
          >
            Predictions
          </button>
          <button
            onClick={() => setActiveTab('history')}
            className={`px-4 py-2 ${
              activeTab === 'history'
                ? 'border-b-2 border-blue-500 text-blue-600'
                : 'text-gray-600'
            }`}
          >
            History
          </button>
          <button
            onClick={() => setActiveTab('models')}
            className={`px-4 py-2 ${
              activeTab === 'models'
                ? 'border-b-2 border-blue-500 text-blue-600'
                : 'text-gray-600'
            }`}
          >
            Models
          </button>
        </div>
      </nav>

      {/* Main Content */}
      <main className="flex-1 overflow-y-auto p-6">
        {activeTab === 'predict' && (
          <PredictionPanel selectedModel={selectedModel} />
        )}
        {activeTab === 'history' && <HistoryPanel />}
        {activeTab === 'models' && <ModelManager onModelsUpdated={loadModels} />}
      </main>
    </div>
  );
}

export default App;
```

## Expected Outputs

1. **Cross-Platform Desktop App**:
   - Windows, macOS, Linux support
   - Native menus and system tray
   - File system integration
   - Auto-updates

2. **Local ML Inference**:
   - Offline operation
   - Fast CPU/GPU inference
   - Batch processing
   - Model management

3. **Desktop Features**:
   - Drag-and-drop files
   - Native notifications
   - Keyboard shortcuts
   - Multi-window support

4. **Data Management**:
   - SQLite database
   - Export functionality
   - Prediction history
   - Settings persistence

## Bonus Challenges

1. **GPU Acceleration**: CUDA support for NVIDIA GPUs
2. **Model Training**: Add transfer learning capabilities
3. **Plugin System**: Extensible architecture
4. **Cloud Sync**: Optional cloud backup
5. **Monitoring**: System resource monitoring
6. **CLI Mode**: Command-line interface
7. **Script Automation**: Batch processing scripts
8. **Custom Models**: Model converter tool

## Resources

- [Electron Documentation](https://www.electronjs.org/docs)
- [TensorFlow Node.js](https://www.tensorflow.org/js/guide/nodejs)
- [Electron Forge](https://www.electronforge.io/)
- [Electron Builder](https://www.electron.build/)
- [Spectron (deprecated)](https://www.electronjs.org/spectron)

## Success Criteria

### Functionality (40%)
- [ ] App runs on all platforms
- [ ] ML inference working
- [ ] File operations functional
- [ ] Model management
- [ ] History tracking

### Performance (30%)
- [ ] Fast startup (<3s)
- [ ] Efficient inference
- [ ] Small bundle size
- [ ] Low memory usage
- [ ] Smooth UI

### Code Quality (20%)
- [ ] Clean architecture
- [ ] IPC best practices
- [ ] Error handling
- [ ] TypeScript typing
- [ ] Security practices

### Desktop Integration (10%)
- [ ] Native menus
- [ ] System tray
- [ ] Notifications
- [ ] Auto-updates
- [ ] File associations
