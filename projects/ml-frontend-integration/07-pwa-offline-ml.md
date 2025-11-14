# Project 07: Progressive Web App with Offline ML

## Overview
Build a Progressive Web App (PWA) with complete offline ML capabilities using Service Workers, IndexedDB, and TensorFlow.js. This project demonstrates creating an installable, offline-first ML application that works seamlessly without internet connectivity, caching models and predictions for a native app-like experience.

## Learning Objectives
- Master Service Worker API for offline functionality
- Implement IndexedDB for client-side data storage
- Create installable PWA with manifest configuration
- Handle offline ML model caching and versioning
- Build background sync for predictions
- Implement push notifications for ML updates
- Optimize caching strategies for ML applications
- Create responsive, app-like UI with native features

## Difficulty Level
**Intermediate to Advanced** - Requires understanding of Service Workers, PWA concepts, client-side storage, and offline-first architecture.

## Technical Stack
- **Framework**: Vanilla JavaScript or React/Vue (lightweight)
- **Service Worker**: Workbox
- **Storage**: IndexedDB (with Dexie.js wrapper)
- **ML**: TensorFlow.js (cached models)
- **Build Tool**: Vite or Webpack with PWA plugin
- **UI**: Tailwind CSS or custom CSS
- **Icons**: PWA Icon Generator
- **Testing**: Lighthouse, Workbox Testing

## Requirements

### PWA Requirements
1. Service Worker for offline functionality
2. Web App Manifest for installation
3. HTTPS deployment (required for PWA)
4. Responsive design (mobile-first)
5. App-like navigation and UI
6. Splash screens and app icons
7. Push notification support
8. Background sync capability

### Offline ML Requirements
1. Model caching in Cache API
2. Model versioning and updates
3. Offline prediction capability
4. IndexedDB for prediction history
5. Fallback mechanisms for failed loads
6. Progressive model loading
7. Bandwidth-aware loading

### Caching Strategies
1. Cache-first for ML models
2. Network-first for API calls
3. Stale-while-revalidate for app shell
4. Cache-only for offline mode
5. Background sync for failed requests

### Features
1. Offline image classification
2. Prediction queue with background sync
3. Model download management
4. Offline prediction history
5. Installation prompts
6. Update notifications
7. Offline indicators
8. Data export/import

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create Vite project with PWA plugin
npm create vite@latest ml-pwa-app -- --template vanilla-ts

cd ml-pwa-app

# Install dependencies
npm install workbox-window workbox-precaching workbox-routing workbox-strategies
npm install dexie # IndexedDB wrapper
npm install @tensorflow/tfjs
npm install idb-keyval # Simple IndexedDB operations
npm install vite-plugin-pwa -D
```

Update `vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.ico', 'robots.txt', 'apple-touch-icon.png'],
      manifest: {
        name: 'Offline ML Predictions',
        short_name: 'ML PWA',
        description: 'Progressive Web App with Offline Machine Learning',
        theme_color: '#4f46e5',
        background_color: '#ffffff',
        display: 'standalone',
        scope: '/',
        start_url: '/',
        orientation: 'portrait',
        icons: [
          {
            src: 'pwa-192x192.png',
            sizes: '192x192',
            type: 'image/png',
          },
          {
            src: 'pwa-512x512.png',
            sizes: '512x512',
            type: 'image/png',
          },
          {
            src: 'pwa-512x512.png',
            sizes: '512x512',
            type: 'image/png',
            purpose: 'any maskable',
          },
        ],
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/cdn\.jsdelivr\.net\/.*/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'tfjs-cache',
              expiration: {
                maxEntries: 10,
                maxAgeSeconds: 60 * 60 * 24 * 365, // 1 year
              },
              cacheableResponse: {
                statuses: [0, 200],
              },
            },
          },
        ],
      },
    }),
  ],
});
```

### Step 2: IndexedDB Setup with Dexie

Create `src/db/database.ts`:

```typescript
import Dexie, { Table } from 'dexie';

export interface ModelRecord {
  id: string;
  name: string;
  version: string;
  url: string;
  size: number;
  downloadedAt: number;
  lastUsed: number;
}

export interface PredictionRecord {
  id: string;
  modelId: string;
  input: any;
  output: any;
  confidence: number;
  timestamp: number;
  synced: boolean;
  imageData?: string;
}

export interface QueuedPrediction {
  id: string;
  modelId: string;
  input: any;
  timestamp: number;
  retries: number;
}

export class MLDatabase extends Dexie {
  models!: Table<ModelRecord, string>;
  predictions!: Table<PredictionRecord, string>;
  queue!: Table<QueuedPrediction, string>;

  constructor() {
    super('MLDatabase');

    this.version(1).stores({
      models: 'id, name, version, downloadedAt, lastUsed',
      predictions: 'id, modelId, timestamp, synced',
      queue: 'id, modelId, timestamp, retries',
    });
  }

  async addModel(model: Omit<ModelRecord, 'downloadedAt' | 'lastUsed'>) {
    const record: ModelRecord = {
      ...model,
      downloadedAt: Date.now(),
      lastUsed: Date.now(),
    };

    await this.models.put(record);
    return record;
  }

  async updateModelLastUsed(modelId: string) {
    await this.models.update(modelId, { lastUsed: Date.now() });
  }

  async addPrediction(
    prediction: Omit<PredictionRecord, 'id' | 'timestamp' | 'synced'>
  ) {
    const record: PredictionRecord = {
      ...prediction,
      id: `pred_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      timestamp: Date.now(),
      synced: false,
    };

    await this.predictions.add(record);
    return record;
  }

  async getPredictionHistory(limit: number = 100) {
    return await this.predictions
      .orderBy('timestamp')
      .reverse()
      .limit(limit)
      .toArray();
  }

  async addToQueue(item: Omit<QueuedPrediction, 'id' | 'timestamp' | 'retries'>) {
    const record: QueuedPrediction = {
      ...item,
      id: `queue_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      timestamp: Date.now(),
      retries: 0,
    };

    await this.queue.add(record);
    return record;
  }

  async getQueuedItems() {
    return await this.queue.toArray();
  }

  async removeFromQueue(id: string) {
    await this.queue.delete(id);
  }

  async clearOldPredictions(daysToKeep: number = 30) {
    const cutoffTime = Date.now() - daysToKeep * 24 * 60 * 60 * 1000;
    await this.predictions.where('timestamp').below(cutoffTime).delete();
  }
}

export const db = new MLDatabase();
```

### Step 3: Service Worker with Model Caching

Create `src/sw/custom-sw.ts`:

```typescript
/// <reference lib="webworker" />

import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';

declare const self: ServiceWorkerGlobalScope;

// Precache app shell
precacheAndRoute(self.__WB_MANIFEST);

// Cache TensorFlow.js models
registerRoute(
  ({ url }) => url.pathname.endsWith('.json') || url.pathname.endsWith('.bin'),
  new CacheFirst({
    cacheName: 'ml-models',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200],
      }),
      new ExpirationPlugin({
        maxEntries: 20,
        maxAgeSeconds: 60 * 60 * 24 * 365, // 1 year
      }),
    ],
  })
);

// Cache images
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
      }),
    ],
  })
);

// Network first for API calls
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 60 * 60, // 1 hour
      }),
    ],
  })
);

// Background sync for failed predictions
self.addEventListener('sync', (event: any) => {
  if (event.tag === 'sync-predictions') {
    event.waitUntil(syncPredictions());
  }
});

async function syncPredictions() {
  console.log('Syncing predictions...');

  try {
    // Get queued predictions from IndexedDB
    const { db } = await import('../db/database');
    const queuedItems = await db.getQueuedItems();

    for (const item of queuedItems) {
      try {
        // Send to API
        const response = await fetch('/api/predictions', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(item),
        });

        if (response.ok) {
          await db.removeFromQueue(item.id);
          console.log(`Synced prediction: ${item.id}`);
        }
      } catch (error) {
        console.error(`Failed to sync prediction ${item.id}:`, error);
      }
    }
  } catch (error) {
    console.error('Background sync failed:', error);
  }
}

// Push notifications for model updates
self.addEventListener('push', (event: any) => {
  const data = event.data?.json() ?? {};

  const title = data.title || 'ML Model Update';
  const options = {
    body: data.body || 'A new model version is available',
    icon: '/pwa-192x192.png',
    badge: '/badge-72x72.png',
    tag: 'model-update',
    data: data,
  };

  event.waitUntil(self.registration.showNotification(title, options));
});

// Handle notification clicks
self.addEventListener('notificationclick', (event: any) => {
  event.notification.close();

  event.waitUntil(
    self.clients.openWindow(event.notification.data?.url || '/')
  );
});
```

### Step 4: ML Manager with Offline Support

Create `src/ml/OfflineMLManager.ts`:

```typescript
import * as tf from '@tensorflow/tfjs';
import { db, ModelRecord } from '../db/database';

export interface OfflineModelConfig {
  id: string;
  name: string;
  url: string;
  version: string;
}

export class OfflineMLManager {
  private models: Map<string, tf.GraphModel | tf.LayersModel> = new Map();
  private isOnline: boolean = navigator.onLine;

  constructor() {
    this.setupOnlineListener();
  }

  private setupOnlineListener() {
    window.addEventListener('online', () => {
      this.isOnline = true;
      console.log('Online - can download models');
    });

    window.addEventListener('offline', () => {
      this.isOnline = false;
      console.log('Offline - using cached models');
    });
  }

  async initialize() {
    await tf.ready();
    console.log('TensorFlow.js ready');
  }

  async downloadModel(config: OfflineModelConfig): Promise<void> {
    if (!this.isOnline) {
      throw new Error('Cannot download model while offline');
    }

    try {
      console.log(`Downloading model: ${config.name}`);

      // Load and cache model
      const model = await tf.loadGraphModel(config.url, {
        onProgress: (fraction) => {
          this.notifyDownloadProgress(config.id, fraction);
        },
      });

      this.models.set(config.id, model);

      // Save to IndexedDB
      await db.addModel({
        id: config.id,
        name: config.name,
        version: config.version,
        url: config.url,
        size: 0, // Could calculate from model weights
      });

      console.log(`Model ${config.name} downloaded and cached`);
    } catch (error) {
      console.error(`Failed to download model ${config.name}:`, error);
      throw error;
    }
  }

  async loadCachedModel(modelId: string): Promise<void> {
    try {
      const modelRecord = await db.models.get(modelId);

      if (!modelRecord) {
        throw new Error(`Model ${modelId} not found in cache`);
      }

      console.log(`Loading cached model: ${modelRecord.name}`);

      // TensorFlow.js will use cached version from Cache API
      const model = await tf.loadGraphModel(modelRecord.url);

      this.models.set(modelId, model);

      await db.updateModelLastUsed(modelId);

      console.log(`Cached model ${modelRecord.name} loaded`);
    } catch (error) {
      console.error(`Failed to load cached model ${modelId}:`, error);
      throw error;
    }
  }

  async predict(modelId: string, input: tf.Tensor): Promise<any> {
    const model = this.models.get(modelId);

    if (!model) {
      // Try to load from cache
      await this.loadCachedModel(modelId);
      return this.predict(modelId, input);
    }

    const startTime = performance.now();

    try {
      const output = model.predict(input) as tf.Tensor;
      const data = await output.data();
      const inferenceTime = performance.now() - startTime;

      // Save to IndexedDB
      await db.addPrediction({
        modelId,
        input: input.shape,
        output: Array.from(data),
        confidence: Math.max(...Array.from(data)),
      });

      output.dispose();

      return {
        output: Array.from(data),
        inferenceTime,
      };
    } catch (error) {
      console.error('Prediction error:', error);

      // Queue for sync if offline
      if (!this.isOnline) {
        await db.addToQueue({
          modelId,
          input: { shape: input.shape },
        });
      }

      throw error;
    }
  }

  async getAvailableModels(): Promise<ModelRecord[]> {
    return await db.models.toArray();
  }

  async deleteModel(modelId: string): Promise<void> {
    const model = this.models.get(modelId);
    if (model) {
      model.dispose();
      this.models.delete(modelId);
    }

    await db.models.delete(modelId);

    // Clear from cache
    const cache = await caches.open('ml-models');
    const keys = await cache.keys();

    for (const request of keys) {
      if (request.url.includes(modelId)) {
        await cache.delete(request);
      }
    }
  }

  private notifyDownloadProgress(modelId: string, progress: number) {
    window.dispatchEvent(
      new CustomEvent('model-download-progress', {
        detail: { modelId, progress },
      })
    );
  }

  isModelLoaded(modelId: string): boolean {
    return this.models.has(modelId);
  }

  getOnlineStatus(): boolean {
    return this.isOnline;
  }

  dispose() {
    this.models.forEach((model) => model.dispose());
    this.models.clear();
  }
}

export const mlManager = new OfflineMLManager();
```

### Step 5: Install Prompt Component

Create `src/components/InstallPrompt.ts`:

```typescript
export class InstallPrompt {
  private deferredPrompt: any = null;
  private installed = false;

  constructor() {
    this.setupInstallPrompt();
  }

  private setupInstallPrompt() {
    window.addEventListener('beforeinstallprompt', (e) => {
      e.preventDefault();
      this.deferredPrompt = e;
      this.showInstallButton();
    });

    window.addEventListener('appinstalled', () => {
      this.installed = true;
      this.hideInstallButton();
      console.log('PWA installed successfully');
    });
  }

  async promptInstall(): Promise<boolean> {
    if (!this.deferredPrompt) {
      console.log('Install prompt not available');
      return false;
    }

    this.deferredPrompt.prompt();

    const { outcome } = await this.deferredPrompt.userChoice;

    this.deferredPrompt = null;

    return outcome === 'accepted';
  }

  private showInstallButton() {
    const installButton = document.getElementById('install-button');
    if (installButton) {
      installButton.style.display = 'block';
    }
  }

  private hideInstallButton() {
    const installButton = document.getElementById('install-button');
    if (installButton) {
      installButton.style.display = 'none';
    }
  }

  isInstalled(): boolean {
    return this.installed || window.matchMedia('(display-mode: standalone)').matches;
  }
}
```

### Step 6: Main Application

Create `src/main.ts`:

```typescript
import './style.css';
import { mlManager } from './ml/OfflineMLManager';
import { db } from './db/database';
import { InstallPrompt } from './components/InstallPrompt';
import { registerSW } from 'virtual:pwa-register';

const app = document.querySelector<HTMLDivElement>('#app')!;

app.innerHTML = `
  <div class="container">
    <header>
      <h1>Offline ML PWA</h1>
      <div class="status">
        <span id="online-status" class="online">Online</span>
      </div>
    </header>

    <div id="install-banner" style="display: none;">
      <p>Install this app for offline access</p>
      <button id="install-button">Install</button>
    </div>

    <main>
      <section class="models">
        <h2>Available Models</h2>
        <div id="models-list"></div>
        <button id="download-model">Download Model</button>
      </section>

      <section class="prediction">
        <h2>Make Prediction</h2>
        <input type="file" id="image-input" accept="image/*" />
        <canvas id="preview-canvas"></canvas>
        <button id="predict-button" disabled>Predict</button>
        <div id="result"></div>
      </section>

      <section class="history">
        <h2>Prediction History</h2>
        <div id="history-list"></div>
      </section>
    </main>
  </div>
`;

class App {
  private installPrompt: InstallPrompt;

  constructor() {
    this.installPrompt = new InstallPrompt();
    this.init();
  }

  private async init() {
    // Register service worker
    registerSW({
      onNeedRefresh() {
        if (confirm('New content available. Reload?')) {
          window.location.reload();
        }
      },
      onOfflineReady() {
        console.log('App ready to work offline');
      },
    });

    // Initialize ML
    await mlManager.initialize();

    // Setup event listeners
    this.setupEventListeners();
    this.updateOnlineStatus();
    this.loadModels();
    this.loadHistory();

    // Show install button if not installed
    if (!this.installPrompt.isInstalled()) {
      document.getElementById('install-banner')!.style.display = 'block';
    }
  }

  private setupEventListeners() {
    // Install button
    document.getElementById('install-button')?.addEventListener('click', async () => {
      const installed = await this.installPrompt.promptInstall();
      if (installed) {
        document.getElementById('install-banner')!.style.display = 'none';
      }
    });

    // Online/offline status
    window.addEventListener('online', () => this.updateOnlineStatus());
    window.addEventListener('offline', () => this.updateOnlineStatus());

    // Download model
    document.getElementById('download-model')?.addEventListener('click', () => {
      this.downloadModel();
    });

    // Image input
    document.getElementById('image-input')?.addEventListener('change', (e) => {
      this.handleImageSelect(e as Event);
    });

    // Predict button
    document.getElementById('predict-button')?.addEventListener('click', () => {
      this.runPrediction();
    });
  }

  private updateOnlineStatus() {
    const statusEl = document.getElementById('online-status');
    if (navigator.onLine) {
      statusEl!.textContent = 'Online';
      statusEl!.className = 'online';
    } else {
      statusEl!.textContent = 'Offline';
      statusEl!.className = 'offline';
    }
  }

  private async loadModels() {
    const models = await db.models.toArray();
    const listEl = document.getElementById('models-list');

    if (models.length === 0) {
      listEl!.innerHTML = '<p>No models downloaded</p>';
      return;
    }

    listEl!.innerHTML = models
      .map(
        (model) => `
        <div class="model-item">
          <h3>${model.name}</h3>
          <p>Version: ${model.version}</p>
          <p>Downloaded: ${new Date(model.downloadedAt).toLocaleDateString()}</p>
        </div>
      `
      )
      .join('');
  }

  private async downloadModel() {
    try {
      await mlManager.downloadModel({
        id: 'mobilenet',
        name: 'MobileNet',
        url: 'https://cdn.jsdelivr.net/npm/@tensorflow-models/mobilenet@2.1.0/dist/model.json',
        version: '2.1.0',
      });

      alert('Model downloaded successfully');
      this.loadModels();
    } catch (error) {
      alert('Failed to download model: ' + (error as Error).message);
    }
  }

  private handleImageSelect(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.getElementById('preview-canvas') as HTMLCanvasElement;
        const ctx = canvas.getContext('2d')!;

        canvas.width = 224;
        canvas.height = 224;
        ctx.drawImage(img, 0, 0, 224, 224);

        (document.getElementById('predict-button') as HTMLButtonElement).disabled = false;
      };
      img.src = e.target!.result as string;
    };
    reader.readAsDataURL(file);
  }

  private async runPrediction() {
    const canvas = document.getElementById('preview-canvas') as HTMLCanvasElement;
    const tensor = tf.browser.fromPixels(canvas).expandDims(0).div(255);

    try {
      const result = await mlManager.predict('mobilenet', tensor);

      document.getElementById('result')!.innerHTML = `
        <h3>Result</h3>
        <p>Inference time: ${result.inferenceTime.toFixed(2)}ms</p>
      `;

      this.loadHistory();
    } catch (error) {
      alert('Prediction failed: ' + (error as Error).message);
    } finally {
      tensor.dispose();
    }
  }

  private async loadHistory() {
    const history = await db.getPredictionHistory(10);
    const listEl = document.getElementById('history-list');

    if (history.length === 0) {
      listEl!.innerHTML = '<p>No predictions yet</p>';
      return;
    }

    listEl!.innerHTML = history
      .map(
        (pred) => `
        <div class="history-item">
          <p>Model: ${pred.modelId}</p>
          <p>Confidence: ${(pred.confidence * 100).toFixed(2)}%</p>
          <p>Time: ${new Date(pred.timestamp).toLocaleString()}</p>
          <p>Synced: ${pred.synced ? '✓' : '✗'}</p>
        </div>
      `
      )
      .join('');
  }
}

// Start app
new App();
```

## Expected Outputs

1. **Installable PWA**:
   - Add to home screen
   - Standalone app mode
   - Splash screen
   - App icons

2. **Offline Functionality**:
   - Works without internet
   - Cached models
   - Local predictions
   - Background sync

3. **Performance**:
   - Fast load times
   - Instant subsequent loads
   - Efficient caching
   - Low data usage

4. **Features**:
   - Model management
   - Offline predictions
   - History tracking
   - Update notifications

## Bonus Challenges

1. **Advanced Caching**: Implement smart cache management
2. **Push Notifications**: Model update notifications
3. **Share Target**: Share images to PWA
4. **File System Access**: Save models locally
5. **Periodic Sync**: Auto-update models
6. **Badge API**: Unread predictions count
7. **Payment Handler**: Premium models
8. **Web Share**: Share predictions

## Resources

- [PWA Documentation](https://web.dev/progressive-web-apps/)
- [Workbox](https://developers.google.com/web/tools/workbox)
- [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Service Workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [Dexie.js](https://dexie.org/)

## Success Criteria

### Functionality (40%)
- [ ] PWA installable
- [ ] Offline ML working
- [ ] Service worker caching
- [ ] IndexedDB storage
- [ ] Background sync

### Performance (30%)
- [ ] Lighthouse PWA score >90
- [ ] Fast load times
- [ ] Efficient caching
- [ ] Low data usage
- [ ] Smooth offline transition

### Code Quality (20%)
- [ ] Clean architecture
- [ ] Error handling
- [ ] TypeScript typing
- [ ] Cache strategies
- [ ] Testing coverage

### User Experience (10%)
- [ ] Install prompts
- [ ] Offline indicators
- [ ] Update notifications
- [ ] Responsive design
- [ ] App-like feel
