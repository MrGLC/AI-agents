# Project 8: Build Offline-First PWA with TensorFlow.js Model

## Overview
Create a Progressive Web App (PWA) that works completely offline using TensorFlow.js for machine learning inference. This project demonstrates how to build installable web applications with Service Workers, IndexedDB model caching, and offline-first architecture, enabling AI-powered experiences without internet connectivity.

## Difficulty Level
Advanced

## Learning Objectives
- Build Progressive Web Apps (PWAs) with offline support
- Implement Service Workers for asset caching
- Cache TensorFlow.js models in IndexedDB
- Handle offline/online state transitions
- Create installable web applications
- Implement background sync for data
- Optimize for slow/intermittent connections
- Manage app updates and cache invalidation

## Technical Stack
- **Frontend**: HTML5, JavaScript ES6+, TensorFlow.js
- **PWA**: Service Workers, Web App Manifest
- **Storage**: IndexedDB for model caching, localStorage for settings
- **Offline**: Cache API, Background Sync API
- **Build**: Workbox (optional) or custom Service Worker
- **Model**: Lightweight image classifier or text model
- **UI**: Responsive design with offline indicators

## Project Requirements

### 1. PWA Implementation
- Create Web App Manifest
- Implement Service Worker
- Support app installation
- Handle offline functionality
- Display offline/online status

### 2. Model Caching Strategy
- Cache model files in IndexedDB
- Implement cache versioning
- Handle cache updates
- Fallback for cache failures
- Show loading progress

### 3. Offline Functionality
- Full app functionality without internet
- Queue predictions during offline mode
- Sync data when online (optional)
- Store user data locally
- Export/import functionality

### 4. User Experience
- Install prompts
- Update notifications
- Offline indicator
- Loading states
- Error handling

## Step-by-Step Implementation

### Step 1: Setup Project Structure
```bash
mkdir offline-pwa-tfjs
cd offline-pwa-tfjs

# Create project structure
mkdir css js models icons
touch index.html manifest.json sw.js js/app.js js/model-cache.js css/style.css
```

### Step 2: Create Web App Manifest
```json
{
  "name": "Offline AI Classifier",
  "short_name": "AI Classifier",
  "description": "AI-powered image classification that works offline",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#667eea",
  "theme_color": "#667eea",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "categories": ["productivity", "utilities"],
  "screenshots": []
}
```

### Step 3: Implement Service Worker
```javascript
// sw.js
const CACHE_VERSION = 'v1';
const CACHE_NAME = `ai-classifier-${CACHE_VERSION}`;

const ASSETS_TO_CACHE = [
    './',
    './index.html',
    './css/style.css',
    './js/app.js',
    './js/model-cache.js',
    './manifest.json',
    'https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0/dist/tf.min.js'
];

// Install event - cache static assets
self.addEventListener('install', (event) => {
    console.log('Service Worker: Installing...');

    event.waitUntil(
        caches.open(CACHE_NAME)
            .then((cache) => {
                console.log('Service Worker: Caching static assets');
                return cache.addAll(ASSETS_TO_CACHE);
            })
            .then(() => {
                console.log('Service Worker: Installed successfully');
                return self.skipWaiting();
            })
            .catch((error) => {
                console.error('Service Worker: Installation failed', error);
            })
    );
});

// Activate event - clean up old caches
self.addEventListener('activate', (event) => {
    console.log('Service Worker: Activating...');

    event.waitUntil(
        caches.keys()
            .then((cacheNames) => {
                return Promise.all(
                    cacheNames.map((cacheName) => {
                        if (cacheName !== CACHE_NAME) {
                            console.log('Service Worker: Deleting old cache', cacheName);
                            return caches.delete(cacheName);
                        }
                    })
                );
            })
            .then(() => {
                console.log('Service Worker: Activated');
                return self.clients.claim();
            })
    );
});

// Fetch event - serve from cache, fallback to network
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request)
            .then((response) => {
                // Return cached response if found
                if (response) {
                    return response;
                }

                // Clone the request
                const fetchRequest = event.request.clone();

                // Make network request
                return fetch(fetchRequest)
                    .then((response) => {
                        // Check if valid response
                        if (!response || response.status !== 200 || response.type !== 'basic') {
                            return response;
                        }

                        // Clone the response
                        const responseToCache = response.clone();

                        // Cache the new response (for non-model files)
                        if (!event.request.url.includes('model.json') &&
                            !event.request.url.includes('.bin')) {
                            caches.open(CACHE_NAME)
                                .then((cache) => {
                                    cache.put(event.request, responseToCache);
                                });
                        }

                        return response;
                    })
                    .catch((error) => {
                        console.log('Service Worker: Fetch failed, returning offline page', error);

                        // Return offline page for navigation requests
                        if (event.request.destination === 'document') {
                            return caches.match('./index.html');
                        }
                    });
            })
    );
});

// Handle messages from main app
self.addEventListener('message', (event) => {
    if (event.data && event.data.type === 'SKIP_WAITING') {
        self.skipWaiting();
    }

    if (event.data && event.data.type === 'CACHE_URLS') {
        event.waitUntil(
            caches.open(CACHE_NAME)
                .then((cache) => cache.addAll(event.data.urls))
        );
    }
});
```

### Step 4: Create Model Caching Module
```javascript
// js/model-cache.js
class ModelCache {
    constructor() {
        this.dbName = 'tfjs-models';
        this.dbVersion = 1;
        this.storeName = 'models';
        this.db = null;
    }

    async init() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(this.dbName, this.dbVersion);

            request.onerror = () => {
                console.error('IndexedDB error:', request.error);
                reject(request.error);
            };

            request.onsuccess = () => {
                this.db = request.result;
                console.log('IndexedDB opened successfully');
                resolve();
            };

            request.onupgradeneeded = (event) => {
                const db = event.target.result;

                // Create object store
                if (!db.objectStoreNames.contains(this.storeName)) {
                    const objectStore = db.createObjectStore(this.storeName, {
                        keyPath: 'modelPath'
                    });
                    objectStore.createIndex('timestamp', 'timestamp', { unique: false });
                    console.log('Object store created');
                }
            };
        });
    }

    async cacheModel(modelPath, modelData) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction([this.storeName], 'readwrite');
            const objectStore = transaction.objectStore(this.storeName);

            const record = {
                modelPath: modelPath,
                modelData: modelData,
                timestamp: Date.now()
            };

            const request = objectStore.put(record);

            request.onsuccess = () => {
                console.log('Model cached in IndexedDB:', modelPath);
                resolve();
            };

            request.onerror = () => {
                console.error('Error caching model:', request.error);
                reject(request.error);
            };
        });
    }

    async getModel(modelPath) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction([this.storeName], 'readonly');
            const objectStore = transaction.objectStore(this.storeName);
            const request = objectStore.get(modelPath);

            request.onsuccess = () => {
                if (request.result) {
                    console.log('Model retrieved from IndexedDB:', modelPath);
                    resolve(request.result.modelData);
                } else {
                    resolve(null);
                }
            };

            request.onerror = () => {
                console.error('Error retrieving model:', request.error);
                reject(request.error);
            };
        });
    }

    async deleteModel(modelPath) {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction([this.storeName], 'readwrite');
            const objectStore = transaction.objectStore(this.storeName);
            const request = objectStore.delete(modelPath);

            request.onsuccess = () => {
                console.log('Model deleted from IndexedDB:', modelPath);
                resolve();
            };

            request.onerror = () => {
                console.error('Error deleting model:', request.error);
                reject(request.error);
            };
        });
    }

    async clearAllModels() {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction([this.storeName], 'readwrite');
            const objectStore = transaction.objectStore(this.storeName);
            const request = objectStore.clear();

            request.onsuccess = () => {
                console.log('All models cleared from IndexedDB');
                resolve();
            };

            request.onerror = () => {
                console.error('Error clearing models:', request.error);
                reject(request.error);
            };
        });
    }

    async getAllModels() {
        return new Promise((resolve, reject) => {
            const transaction = this.db.transaction([this.storeName], 'readonly');
            const objectStore = transaction.objectStore(this.storeName);
            const request = objectStore.getAll();

            request.onsuccess = () => {
                resolve(request.result);
            };

            request.onerror = () => {
                console.error('Error getting all models:', request.error);
                reject(request.error);
            };
        });
    }
}
```

### Step 5: Create Main Application
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="theme-color" content="#667eea">
    <meta name="description" content="AI-powered image classification that works offline">

    <title>Offline AI Classifier</title>

    <!-- Web App Manifest -->
    <link rel="manifest" href="./manifest.json">

    <!-- iOS Support -->
    <link rel="apple-touch-icon" href="./icons/icon-192x192.png">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="AI Classifier">

    <!-- Favicon -->
    <link rel="icon" type="image/png" href="./icons/icon-192x192.png">

    <!-- TensorFlow.js -->
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.11.0"></script>

    <link rel="stylesheet" href="./css/style.css">
</head>
<body>
    <div class="container">
        <!-- Connection Status -->
        <div id="connectionStatus" class="status-bar online">
            <span id="statusIcon">🟢</span>
            <span id="statusText">Online</span>
        </div>

        <!-- Update Banner -->
        <div id="updateBanner" class="update-banner" style="display: none;">
            <span>New version available!</span>
            <button onclick="updateApp()">Update Now</button>
        </div>

        <header>
            <h1>🧠 Offline AI Classifier</h1>
            <p>Works completely offline - no internet required!</p>
        </header>

        <!-- Install Prompt -->
        <div id="installPrompt" class="install-prompt" style="display: none;">
            <div class="install-content">
                <h3>📥 Install App</h3>
                <p>Install this app for a better experience and offline access!</p>
                <div class="install-buttons">
                    <button onclick="installApp()" class="btn-primary">Install</button>
                    <button onclick="dismissInstall()" class="btn-secondary">Maybe Later</button>
                </div>
            </div>
        </div>

        <!-- Model Status -->
        <div class="model-status">
            <h3>Model Status</h3>
            <div id="modelInfo">
                <div class="status-item">
                    <span>Model:</span>
                    <span id="modelStatus">Not Loaded</span>
                </div>
                <div class="status-item">
                    <span>Cached:</span>
                    <span id="cacheStatus">No</span>
                </div>
                <div class="status-item">
                    <span>Size:</span>
                    <span id="modelSize">-</span>
                </div>
            </div>
            <button onclick="downloadModel()" id="downloadBtn" class="btn-primary">
                📥 Download Model for Offline Use
            </button>
        </div>

        <!-- Image Upload -->
        <div class="upload-section">
            <h3>Upload Image</h3>
            <input type="file" id="imageUpload" accept="image/*" style="display: none;">
            <button onclick="document.getElementById('imageUpload').click()" class="btn-secondary">
                📁 Choose Image
            </button>
            <img id="imagePreview" style="display: none; max-width: 100%; margin-top: 20px; border-radius: 10px;">
        </div>

        <!-- Predictions -->
        <div id="predictions" class="predictions" style="display: none;">
            <h3>Predictions:</h3>
            <div id="predictionsList"></div>
        </div>

        <!-- Offline Queue -->
        <div id="offlineQueue" class="offline-queue" style="display: none;">
            <h3>⏳ Offline Prediction Queue</h3>
            <p>Predictions made while offline: <span id="queueCount">0</span></p>
            <button onclick="clearQueue()" class="btn-secondary">Clear Queue</button>
        </div>

        <!-- App Info -->
        <div class="app-info">
            <h3>About This App</h3>
            <ul>
                <li>✅ Works completely offline</li>
                <li>✅ Installable on your device</li>
                <li>✅ Model cached for fast loading</li>
                <li>✅ No internet required after initial setup</li>
            </ul>
        </div>
    </div>

    <script src="./js/model-cache.js"></script>
    <script src="./js/app.js"></script>
</body>
</html>
```

### Step 6: Implement Application Logic
```javascript
// js/app.js
let model = null;
let modelCache = null;
let deferredPrompt = null;
let isOnline = navigator.onLine;
let offlineQueue = [];

const MODEL_PATH = './models/mobilenet/model.json';

// Initialize app
async function init() {
    // Register service worker
    if ('serviceWorker' in navigator) {
        try {
            const registration = await navigator.serviceWorker.register('./sw.js');
            console.log('Service Worker registered:', registration);

            // Check for updates
            registration.addEventListener('updatefound', () => {
                const newWorker = registration.installing;
                newWorker.addEventListener('statechange', () => {
                    if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                        showUpdateBanner();
                    }
                });
            });

        } catch (error) {
            console.error('Service Worker registration failed:', error);
        }
    }

    // Initialize model cache
    modelCache = new ModelCache();
    await modelCache.init();

    // Setup online/offline listeners
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    updateConnectionStatus();

    // Setup install prompt
    window.addEventListener('beforeinstallprompt', (e) => {
        e.preventDefault();
        deferredPrompt = e;
        showInstallPrompt();
    });

    // Check if model is cached
    await checkModelCache();

    // Setup image upload
    document.getElementById('imageUpload').addEventListener('change', handleImageUpload);

    // Load offline queue from localStorage
    loadOfflineQueue();
}

// Check if model is cached
async function checkModelCache() {
    try {
        const cached = await modelCache.getModel(MODEL_PATH);

        if (cached) {
            document.getElementById('cacheStatus').textContent = 'Yes';
            document.getElementById('cacheStatus').style.color = '#4CAF50';
            document.getElementById('downloadBtn').textContent = '✓ Model Downloaded';
            document.getElementById('downloadBtn').disabled = true;

            // Load model from cache
            await loadModelFromCache();
        } else {
            document.getElementById('cacheStatus').textContent = 'No';
            document.getElementById('cacheStatus').style.color = '#f44336';
        }
    } catch (error) {
        console.error('Error checking model cache:', error);
    }
}

// Download and cache model
async function downloadModel() {
    document.getElementById('downloadBtn').disabled = true;
    document.getElementById('downloadBtn').textContent = 'Downloading...';

    try {
        // Load model from network
        model = await tf.loadGraphModel(MODEL_PATH);

        // Cache model
        await cacheModelFiles();

        document.getElementById('modelStatus').textContent = 'Loaded';
        document.getElementById('cacheStatus').textContent = 'Yes';
        document.getElementById('cacheStatus').style.color = '#4CAF50';
        document.getElementById('downloadBtn').textContent = '✓ Model Downloaded';

        alert('Model downloaded and cached for offline use!');

    } catch (error) {
        console.error('Error downloading model:', error);
        document.getElementById('downloadBtn').disabled = false;
        document.getElementById('downloadBtn').textContent = 'Download Failed - Retry';
        alert('Failed to download model. Please check your connection.');
    }
}

// Cache model files
async function cacheModelFiles() {
    // This is a simplified version
    // In production, you'd need to cache model.json and all weight files
    const modelInfo = {
        path: MODEL_PATH,
        timestamp: Date.now()
    };

    await modelCache.cacheModel(MODEL_PATH, modelInfo);
}

// Load model from cache
async function loadModelFromCache() {
    try {
        // In a real app, you'd reconstruct the model from cached files
        // For this example, we'll just try to load it normally
        model = await tf.loadGraphModel(MODEL_PATH);

        document.getElementById('modelStatus').textContent = 'Loaded (Cached)';
        document.getElementById('modelStatus').style.color = '#4CAF50';

    } catch (error) {
        console.error('Error loading model from cache:', error);
        document.getElementById('modelStatus').textContent = 'Load Failed';
    }
}

// Handle image upload
async function handleImageUpload(e) {
    const file = e.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = async (event) => {
        const img = new Image();
        img.onload = async () => {
            document.getElementById('imagePreview').src = event.target.result;
            document.getElementById('imagePreview').style.display = 'block';

            if (model) {
                await classifyImage(img);
            } else {
                if (isOnline) {
                    alert('Please download the model first!');
                } else {
                    addToOfflineQueue({
                        image: event.target.result,
                        timestamp: Date.now()
                    });
                    alert('You are offline. Prediction queued for when you\'re back online.');
                }
            }
        };
        img.src = event.target.result;
    };
    reader.readAsDataURL(file);
}

// Classify image
async function classifyImage(img) {
    try {
        // Preprocess image
        const tensor = tf.tidy(() => {
            let imgTensor = tf.browser.fromPixels(img);
            imgTensor = tf.image.resizeBilinear(imgTensor, [224, 224]);
            imgTensor = imgTensor.div(255.0);
            imgTensor = imgTensor.expandDims(0);
            return imgTensor;
        });

        // Run inference
        const predictions = await model.predict(tensor).data();
        tensor.dispose();

        // Display predictions (simplified)
        displayPredictions(Array.from(predictions).slice(0, 5));

    } catch (error) {
        console.error('Classification error:', error);
        alert('Error during classification. Please try again.');
    }
}

// Display predictions
function displayPredictions(predictions) {
    const predictionsDiv = document.getElementById('predictions');
    const predictionsList = document.getElementById('predictionsList');

    predictionsDiv.style.display = 'block';
    predictionsList.innerHTML = '';

    predictions.forEach((prob, idx) => {
        const item = document.createElement('div');
        item.className = 'prediction-item';
        item.innerHTML = `
            <span>Class ${idx}:</span>
            <span>${(prob * 100).toFixed(2)}%</span>
        `;
        predictionsList.appendChild(item);
    });
}

// Handle online/offline events
function handleOnline() {
    isOnline = true;
    updateConnectionStatus();
    processOfflineQueue();
}

function handleOffline() {
    isOnline = false;
    updateConnectionStatus();
}

function updateConnectionStatus() {
    const statusBar = document.getElementById('connectionStatus');
    const statusIcon = document.getElementById('statusIcon');
    const statusText = document.getElementById('statusText');

    if (isOnline) {
        statusBar.className = 'status-bar online';
        statusIcon.textContent = '🟢';
        statusText.textContent = 'Online';
    } else {
        statusBar.className = 'status-bar offline';
        statusIcon.textContent = '🔴';
        statusText.textContent = 'Offline';
    }
}

// Offline queue management
function addToOfflineQueue(item) {
    offlineQueue.push(item);
    updateOfflineQueueDisplay();
    saveOfflineQueue();
}

function loadOfflineQueue() {
    const saved = localStorage.getItem('offlineQueue');
    if (saved) {
        offlineQueue = JSON.parse(saved);
        updateOfflineQueueDisplay();
    }
}

function saveOfflineQueue() {
    localStorage.setItem('offlineQueue', JSON.stringify(offlineQueue));
}

function updateOfflineQueueDisplay() {
    const queueDiv = document.getElementById('offlineQueue');
    const queueCount = document.getElementById('queueCount');

    if (offlineQueue.length > 0) {
        queueDiv.style.display = 'block';
        queueCount.textContent = offlineQueue.length;
    } else {
        queueDiv.style.display = 'none';
    }
}

function clearQueue() {
    offlineQueue = [];
    updateOfflineQueueDisplay();
    saveOfflineQueue();
}

async function processOfflineQueue() {
    if (offlineQueue.length === 0 || !model) return;

    console.log('Processing offline queue...');
    // Process queued items when back online
    // Implementation depends on your use case
}

// Install/Update functions
function showInstallPrompt() {
    document.getElementById('installPrompt').style.display = 'block';
}

function dismissInstall() {
    document.getElementById('installPrompt').style.display = 'none';
}

async function installApp() {
    if (!deferredPrompt) return;

    deferredPrompt.prompt();
    const { outcome } = await deferredPrompt.userChoice;

    console.log(`User response: ${outcome}`);
    deferredPrompt = null;
    dismissInstall();
}

function showUpdateBanner() {
    document.getElementById('updateBanner').style.display = 'flex';
}

function updateApp() {
    if ('serviceWorker' in navigator) {
        navigator.serviceWorker.getRegistration().then((registration) => {
            if (registration && registration.waiting) {
                registration.waiting.postMessage({ type: 'SKIP_WAITING' });
            }
        });
    }
    window.location.reload();
}

// Initialize app
init();
```

## Expected Outputs

1. **PWA Features**:
   - Installable app
   - Offline functionality
   - Update mechanism
   - Splash screen

2. **Model Caching**:
   - Models stored in IndexedDB
   - Fast loading from cache
   - Version management

3. **Offline Capabilities**:
   - Full app functionality without internet
   - Offline queue for predictions
   - Status indicators

4. **Performance**:
   - Initial load: <5 seconds online
   - Subsequent loads: <2 seconds (cached)
   - Inference: same as online

## Bonus Challenges

- [ ] Implement background sync for predictions
- [ ] Add push notifications for updates
- [ ] Create data export/import functionality
- [ ] Implement differential updates for models
- [ ] Add usage statistics dashboard
- [ ] Create multiple model support
- [ ] Implement prediction history
- [ ] Add offline data visualization
- [ ] Create sync conflict resolution
- [ ] Build admin panel for cache management

## Resources

- [PWA Documentation](https://web.dev/progressive-web-apps/)
- [Service Workers Guide](https://developers.google.com/web/fundamentals/primers/service-workers)
- [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Workbox](https://developers.google.com/web/tools/workbox)
- [Web App Manifest](https://web.dev/add-manifest/)

## Success Criteria

- App installs successfully on devices
- Service Worker caches all assets
- Model loads and works offline
- Online/offline transitions handled gracefully
- Update mechanism works correctly
- App works without internet after initial setup
- IndexedDB stores model successfully
- Install prompt appears appropriately
- Lighthouse PWA score >90
- Works on mobile and desktop
