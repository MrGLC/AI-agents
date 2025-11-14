# Project 02: Vue.js Dashboard for Model Monitoring

## Overview
Build a comprehensive Vue.js dashboard for monitoring machine learning model performance in production. This project demonstrates real-time monitoring, metrics visualization, model drift detection, and alert management using Vue 3 Composition API with modern state management patterns.

## Learning Objectives
- Master Vue 3 Composition API for complex applications
- Implement real-time data fetching and updates
- Create interactive dashboards with multiple visualization types
- Handle time-series data and metric calculations
- Implement Pinia for centralized state management
- Build reusable composables for ML monitoring logic
- Integrate WebSocket connections for live updates
- Apply responsive design patterns for dashboards

## Difficulty Level
**Advanced** - Requires strong understanding of Vue 3, state management, real-time data, and data visualization.

## Technical Stack
- **Frontend**: Vue 3 (Composition API)
- **State Management**: Pinia
- **Routing**: Vue Router
- **API Client**: Axios
- **Real-time**: Socket.IO Client
- **Visualization**: Chart.js, Vue-ChartJS, Apache ECharts
- **UI Framework**: Vuetify or Element Plus
- **Date Handling**: Day.js
- **Testing**: Vitest, Vue Test Utils

## Requirements

### UI/UX Requirements
1. Multi-panel dashboard with metrics overview
2. Time-range selector for historical data
3. Real-time metric updates with animations
4. Alert notifications panel
5. Model comparison views
6. Drill-down capability for detailed metrics
7. Dark/light theme toggle
8. Responsive layout for different screen sizes

### State Management Requirements
1. Centralized model metrics store
2. Real-time data synchronization
3. Historical data caching
4. Alert state management
5. User preferences persistence
6. WebSocket connection state

### Monitoring Features
1. **Performance Metrics**:
   - Accuracy, Precision, Recall, F1-Score
   - Latency (p50, p95, p99)
   - Throughput (requests/second)
   - Error rate

2. **Data Quality Metrics**:
   - Input distribution monitoring
   - Feature drift detection
   - Missing value tracking
   - Outlier detection

3. **Model Health**:
   - Prediction confidence trends
   - Model version tracking
   - Resource utilization
   - Uptime monitoring

### Visualization Requirements
1. Line charts for time-series metrics
2. Gauge charts for current status
3. Heatmaps for confusion matrices
4. Distribution plots for feature drift
5. Bar charts for comparison metrics
6. Sparklines for quick metric overview

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create Vue 3 project
npm create vue@latest ml-monitoring-dashboard

# Select the following:
# ✔ TypeScript? Yes
# ✔ JSX Support? No
# ✔ Vue Router? Yes
# ✔ Pinia? Yes
# ✔ Vitest? Yes
# ✔ ESLint? Yes

cd ml-monitoring-dashboard

# Install additional dependencies
npm install axios socket.io-client chart.js vue-chartjs
npm install dayjs element-plus @element-plus/icons-vue
npm install lodash-es
```

### Step 2: API Client and WebSocket Setup

Create `src/services/api.ts`:

```typescript
import axios, { AxiosInstance } from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000';

class APIClient {
  private client: AxiosInstance;

  constructor() {
    this.client = axios.create({
      baseURL: API_BASE_URL,
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    this.setupInterceptors();
  }

  private setupInterceptors() {
    this.client.interceptors.request.use(
      (config) => {
        const token = localStorage.getItem('authToken');
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );

    this.client.interceptors.response.use(
      (response) => response,
      (error) => {
        console.error('API Error:', error.response?.data || error.message);
        return Promise.reject(error);
      }
    );
  }

  // Metrics endpoints
  async getMetrics(modelId: string, timeRange: string) {
    const response = await this.client.get(`/metrics/${modelId}`, {
      params: { range: timeRange },
    });
    return response.data;
  }

  async getMetricHistory(modelId: string, metricName: string, timeRange: string) {
    const response = await this.client.get(
      `/metrics/${modelId}/${metricName}/history`,
      { params: { range: timeRange } }
    );
    return response.data;
  }

  // Model endpoints
  async getModels() {
    const response = await this.client.get('/models');
    return response.data;
  }

  async getModelInfo(modelId: string) {
    const response = await this.client.get(`/models/${modelId}`);
    return response.data;
  }

  // Alerts endpoints
  async getAlerts(modelId?: string) {
    const response = await this.client.get('/alerts', {
      params: { model_id: modelId },
    });
    return response.data;
  }

  async acknowledgeAlert(alertId: string) {
    const response = await this.client.post(`/alerts/${alertId}/acknowledge`);
    return response.data;
  }

  // Drift detection
  async getDriftReport(modelId: string) {
    const response = await this.client.get(`/drift/${modelId}`);
    return response.data;
  }
}

export const apiClient = new APIClient();
```

Create `src/services/websocket.ts`:

```typescript
import { io, Socket } from 'socket.io-client';

const WS_URL = import.meta.env.VITE_WS_URL || 'http://localhost:8000';

class WebSocketService {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect() {
    this.socket = io(WS_URL, {
      transports: ['websocket'],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
    });

    this.socket.on('connect', () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
    });

    this.socket.on('disconnect', () => {
      console.log('WebSocket disconnected');
    });

    this.socket.on('error', (error) => {
      console.error('WebSocket error:', error);
    });

    return this.socket;
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }

  subscribe(event: string, callback: (data: any) => void) {
    if (!this.socket) {
      throw new Error('Socket not connected');
    }
    this.socket.on(event, callback);
  }

  unsubscribe(event: string) {
    if (this.socket) {
      this.socket.off(event);
    }
  }

  emit(event: string, data: any) {
    if (!this.socket) {
      throw new Error('Socket not connected');
    }
    this.socket.emit(event, data);
  }

  getSocket() {
    return this.socket;
  }
}

export const wsService = new WebSocketService();
```

### Step 3: Pinia Store for Monitoring

Create `src/stores/monitoring.ts`:

```typescript
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import { apiClient } from '@/services/api';
import { wsService } from '@/services/websocket';
import dayjs from 'dayjs';

export interface Metric {
  name: string;
  value: number;
  timestamp: string;
  threshold?: number;
}

export interface Alert {
  id: string;
  modelId: string;
  severity: 'info' | 'warning' | 'critical';
  message: string;
  timestamp: string;
  acknowledged: boolean;
}

export interface ModelMetrics {
  modelId: string;
  modelName: string;
  accuracy: number;
  latency: {
    p50: number;
    p95: number;
    p99: number;
  };
  throughput: number;
  errorRate: number;
  timestamp: string;
}

export const useMonitoringStore = defineStore('monitoring', () => {
  // State
  const models = ref<any[]>([]);
  const selectedModel = ref<string>('');
  const currentMetrics = ref<ModelMetrics | null>(null);
  const metricsHistory = ref<Map<string, Metric[]>>(new Map());
  const alerts = ref<Alert[]>([]);
  const timeRange = ref<string>('1h');
  const isLoading = ref(false);
  const isConnected = ref(false);

  // Computed
  const unacknowledgedAlerts = computed(() =>
    alerts.value.filter(alert => !alert.acknowledged)
  );

  const criticalAlerts = computed(() =>
    alerts.value.filter(alert => alert.severity === 'critical' && !alert.acknowledged)
  );

  const modelHealth = computed(() => {
    if (!currentMetrics.value) return 'unknown';

    const { accuracy, errorRate, latency } = currentMetrics.value;

    if (accuracy > 0.9 && errorRate < 0.01 && latency.p99 < 100) {
      return 'healthy';
    } else if (accuracy > 0.8 && errorRate < 0.05 && latency.p99 < 200) {
      return 'degraded';
    } else {
      return 'unhealthy';
    }
  });

  // Actions
  async function fetchModels() {
    try {
      isLoading.value = true;
      const data = await apiClient.getModels();
      models.value = data;

      if (data.length > 0 && !selectedModel.value) {
        selectedModel.value = data[0].id;
      }
    } catch (error) {
      console.error('Failed to fetch models:', error);
    } finally {
      isLoading.value = false;
    }
  }

  async function fetchCurrentMetrics(modelId: string) {
    try {
      const data = await apiClient.getMetrics(modelId, timeRange.value);
      currentMetrics.value = data;
    } catch (error) {
      console.error('Failed to fetch metrics:', error);
    }
  }

  async function fetchMetricHistory(modelId: string, metricName: string) {
    try {
      const data = await apiClient.getMetricHistory(
        modelId,
        metricName,
        timeRange.value
      );
      metricsHistory.value.set(metricName, data);
    } catch (error) {
      console.error('Failed to fetch metric history:', error);
    }
  }

  async function fetchAlerts(modelId?: string) {
    try {
      const data = await apiClient.getAlerts(modelId);
      alerts.value = data;
    } catch (error) {
      console.error('Failed to fetch alerts:', error);
    }
  }

  async function acknowledgeAlert(alertId: string) {
    try {
      await apiClient.acknowledgeAlert(alertId);
      const alert = alerts.value.find(a => a.id === alertId);
      if (alert) {
        alert.acknowledged = true;
      }
    } catch (error) {
      console.error('Failed to acknowledge alert:', error);
    }
  }

  function connectWebSocket() {
    const socket = wsService.connect();

    socket.on('connect', () => {
      isConnected.value = true;
    });

    socket.on('disconnect', () => {
      isConnected.value = false;
    });

    // Subscribe to metric updates
    wsService.subscribe('metric_update', (data: ModelMetrics) => {
      if (data.modelId === selectedModel.value) {
        currentMetrics.value = data;
      }
    });

    // Subscribe to new alerts
    wsService.subscribe('new_alert', (alert: Alert) => {
      alerts.value.unshift(alert);
    });

    // Join model room for updates
    if (selectedModel.value) {
      wsService.emit('join_model', { modelId: selectedModel.value });
    }
  }

  function disconnectWebSocket() {
    wsService.disconnect();
    isConnected.value = false;
  }

  function updateMetricHistory(metricName: string, newData: Metric) {
    const history = metricsHistory.value.get(metricName) || [];
    history.push(newData);

    // Keep last 100 data points
    if (history.length > 100) {
      history.shift();
    }

    metricsHistory.value.set(metricName, history);
  }

  function setTimeRange(range: string) {
    timeRange.value = range;
    if (selectedModel.value) {
      fetchCurrentMetrics(selectedModel.value);
    }
  }

  function setSelectedModel(modelId: string) {
    selectedModel.value = modelId;

    // Update WebSocket subscription
    if (isConnected.value) {
      wsService.emit('leave_model', { modelId: selectedModel.value });
      wsService.emit('join_model', { modelId });
    }

    // Fetch metrics for new model
    fetchCurrentMetrics(modelId);
    fetchAlerts(modelId);
  }

  return {
    // State
    models,
    selectedModel,
    currentMetrics,
    metricsHistory,
    alerts,
    timeRange,
    isLoading,
    isConnected,

    // Computed
    unacknowledgedAlerts,
    criticalAlerts,
    modelHealth,

    // Actions
    fetchModels,
    fetchCurrentMetrics,
    fetchMetricHistory,
    fetchAlerts,
    acknowledgeAlert,
    connectWebSocket,
    disconnectWebSocket,
    updateMetricHistory,
    setTimeRange,
    setSelectedModel,
  };
});
```

### Step 4: Composables for Reusable Logic

Create `src/composables/useMetrics.ts`:

```typescript
import { ref, watch, onUnmounted } from 'vue';
import { useMonitoringStore } from '@/stores/monitoring';
import { storeToRefs } from 'pinia';

export function useMetrics() {
  const store = useMonitoringStore();
  const { selectedModel, timeRange } = storeToRefs(store);
  const refreshInterval = ref<number | null>(null);

  const startAutoRefresh = (intervalMs: number = 30000) => {
    if (refreshInterval.value) {
      clearInterval(refreshInterval.value);
    }

    refreshInterval.value = window.setInterval(() => {
      if (selectedModel.value) {
        store.fetchCurrentMetrics(selectedModel.value);
      }
    }, intervalMs);
  };

  const stopAutoRefresh = () => {
    if (refreshInterval.value) {
      clearInterval(refreshInterval.value);
      refreshInterval.value = null;
    }
  };

  // Watch for model or time range changes
  watch([selectedModel, timeRange], ([newModel, newRange]) => {
    if (newModel) {
      store.fetchCurrentMetrics(newModel);
      store.fetchAlerts(newModel);
    }
  });

  onUnmounted(() => {
    stopAutoRefresh();
  });

  return {
    startAutoRefresh,
    stopAutoRefresh,
  };
}
```

Create `src/composables/useChartData.ts`:

```typescript
import { computed } from 'vue';
import type { Metric } from '@/stores/monitoring';
import dayjs from 'dayjs';

export function useChartData(metrics: Metric[]) {
  const chartData = computed(() => ({
    labels: metrics.map(m => dayjs(m.timestamp).format('HH:mm:ss')),
    datasets: [
      {
        label: metrics[0]?.name || 'Metric',
        data: metrics.map(m => m.value),
        borderColor: 'rgb(59, 130, 246)',
        backgroundColor: 'rgba(59, 130, 246, 0.1)',
        tension: 0.4,
      },
    ],
  }));

  const chartOptions = computed(() => ({
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: {
        display: true,
        position: 'top' as const,
      },
      tooltip: {
        mode: 'index' as const,
        intersect: false,
      },
    },
    scales: {
      x: {
        display: true,
        title: {
          display: true,
          text: 'Time',
        },
      },
      y: {
        display: true,
        title: {
          display: true,
          text: 'Value',
        },
      },
    },
  }));

  return {
    chartData,
    chartOptions,
  };
}
```

### Step 5: Dashboard Components

Create `src/components/MetricsOverview.vue`:

```vue
<template>
  <div class="metrics-overview">
    <el-row :gutter="20">
      <el-col :xs="24" :sm="12" :md="6">
        <el-card class="metric-card">
          <div class="metric-content">
            <div class="metric-icon accuracy">
              <el-icon><TrendCharts /></el-icon>
            </div>
            <div class="metric-details">
              <div class="metric-label">Accuracy</div>
              <div class="metric-value">
                {{ formatPercent(currentMetrics?.accuracy) }}
              </div>
              <div :class="['metric-trend', accuracyTrend]">
                <el-icon v-if="accuracyTrend === 'up'"><CaretTop /></el-icon>
                <el-icon v-else><CaretBottom /></el-icon>
                {{ accuracyChange }}
              </div>
            </div>
          </div>
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="12" :md="6">
        <el-card class="metric-card">
          <div class="metric-content">
            <div class="metric-icon latency">
              <el-icon><Timer /></el-icon>
            </div>
            <div class="metric-details">
              <div class="metric-label">P99 Latency</div>
              <div class="metric-value">
                {{ currentMetrics?.latency.p99 }}ms
              </div>
              <div class="metric-sublabel">
                P95: {{ currentMetrics?.latency.p95 }}ms
              </div>
            </div>
          </div>
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="12" :md="6">
        <el-card class="metric-card">
          <div class="metric-content">
            <div class="metric-icon throughput">
              <el-icon><DataLine /></el-icon>
            </div>
            <div class="metric-details">
              <div class="metric-label">Throughput</div>
              <div class="metric-value">
                {{ currentMetrics?.throughput }}/s
              </div>
              <div class="metric-sublabel">Requests per second</div>
            </div>
          </div>
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="12" :md="6">
        <el-card class="metric-card">
          <div class="metric-content">
            <div class="metric-icon error-rate">
              <el-icon><Warning /></el-icon>
            </div>
            <div class="metric-details">
              <div class="metric-label">Error Rate</div>
              <div class="metric-value">
                {{ formatPercent(currentMetrics?.errorRate) }}
              </div>
              <div :class="['metric-status', errorStatus]">
                {{ errorStatus.toUpperCase() }}
              </div>
            </div>
          </div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue';
import { storeToRefs } from 'pinia';
import { useMonitoringStore } from '@/stores/monitoring';
import {
  TrendCharts,
  Timer,
  DataLine,
  Warning,
  CaretTop,
  CaretBottom,
} from '@element-plus/icons-vue';

const store = useMonitoringStore();
const { currentMetrics } = storeToRefs(store);

const formatPercent = (value?: number) => {
  if (value === undefined) return 'N/A';
  return `${(value * 100).toFixed(2)}%`;
};

const accuracyTrend = computed(() => {
  // Calculate based on historical data
  return 'up';
});

const accuracyChange = computed(() => {
  return '+2.3%';
});

const errorStatus = computed(() => {
  const rate = currentMetrics.value?.errorRate || 0;
  if (rate < 0.01) return 'good';
  if (rate < 0.05) return 'warning';
  return 'critical';
});
</script>

<style scoped>
.metrics-overview {
  margin-bottom: 20px;
}

.metric-card {
  margin-bottom: 20px;
}

.metric-content {
  display: flex;
  align-items: center;
  gap: 15px;
}

.metric-icon {
  width: 50px;
  height: 50px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  color: white;
}

.metric-icon.accuracy {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.metric-icon.latency {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
}

.metric-icon.throughput {
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
}

.metric-icon.error-rate {
  background: linear-gradient(135deg, #fa709a 0%, #fee140 100%);
}

.metric-details {
  flex: 1;
}

.metric-label {
  font-size: 12px;
  color: #909399;
  margin-bottom: 5px;
}

.metric-value {
  font-size: 24px;
  font-weight: bold;
  color: #303133;
  margin-bottom: 3px;
}

.metric-sublabel {
  font-size: 11px;
  color: #c0c4cc;
}

.metric-trend {
  font-size: 12px;
  display: flex;
  align-items: center;
  gap: 3px;
}

.metric-trend.up {
  color: #67c23a;
}

.metric-trend.down {
  color: #f56c6c;
}

.metric-status {
  font-size: 10px;
  font-weight: bold;
  padding: 2px 8px;
  border-radius: 10px;
  display: inline-block;
  margin-top: 5px;
}

.metric-status.good {
  background: #f0f9ff;
  color: #67c23a;
}

.metric-status.warning {
  background: #fef0f0;
  color: #e6a23c;
}

.metric-status.critical {
  background: #fef0f0;
  color: #f56c6c;
}
</style>
```

Create `src/components/MetricChart.vue`:

```vue
<template>
  <el-card class="metric-chart-card">
    <template #header>
      <div class="card-header">
        <span>{{ title }}</span>
        <el-select v-model="selectedTimeRange" size="small" @change="handleTimeRangeChange">
          <el-option label="Last 1 hour" value="1h" />
          <el-option label="Last 6 hours" value="6h" />
          <el-option label="Last 24 hours" value="24h" />
          <el-option label="Last 7 days" value="7d" />
        </el-select>
      </div>
    </template>

    <div class="chart-container">
      <Line
        v-if="chartData"
        :data="chartData"
        :options="chartOptions"
      />
      <div v-else class="no-data">
        No data available
      </div>
    </div>
  </el-card>
</template>

<script setup lang="ts">
import { ref, watch, onMounted } from 'vue';
import { Line } from 'vue-chartjs';
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler,
} from 'chart.js';
import { useMonitoringStore } from '@/stores/monitoring';
import { storeToRefs } from 'pinia';
import { useChartData } from '@/composables/useChartData';

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
);

interface Props {
  title: string;
  metricName: string;
}

const props = defineProps<Props>();

const store = useMonitoringStore();
const { selectedModel, metricsHistory } = storeToRefs(store);
const selectedTimeRange = ref('1h');

const metrics = ref(metricsHistory.value.get(props.metricName) || []);
const { chartData, chartOptions } = useChartData(metrics.value);

const handleTimeRangeChange = (range: string) => {
  store.setTimeRange(range);
  if (selectedModel.value) {
    store.fetchMetricHistory(selectedModel.value, props.metricName);
  }
};

watch(() => metricsHistory.value.get(props.metricName), (newMetrics) => {
  if (newMetrics) {
    metrics.value = newMetrics;
  }
});

onMounted(() => {
  if (selectedModel.value) {
    store.fetchMetricHistory(selectedModel.value, props.metricName);
  }
});
</script>

<style scoped>
.metric-chart-card {
  height: 100%;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.chart-container {
  height: 300px;
  position: relative;
}

.no-data {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #909399;
}
</style>
```

### Step 6: Main Dashboard View

Create `src/views/DashboardView.vue`:

```vue
<template>
  <div class="dashboard-container">
    <!-- Header -->
    <div class="dashboard-header">
      <h1>Model Monitoring Dashboard</h1>
      <div class="header-controls">
        <el-select
          v-model="selectedModel"
          placeholder="Select Model"
          @change="handleModelChange"
        >
          <el-option
            v-for="model in models"
            :key="model.id"
            :label="model.name"
            :value="model.id"
          />
        </el-select>

        <el-badge :value="unacknowledgedAlerts.length" :hidden="unacknowledgedAlerts.length === 0">
          <el-button @click="showAlerts = true">
            <el-icon><Bell /></el-icon>
            Alerts
          </el-button>
        </el-badge>

        <div class="connection-status">
          <el-tag :type="isConnected ? 'success' : 'danger'" size="small">
            <el-icon><Connection /></el-icon>
            {{ isConnected ? 'Connected' : 'Disconnected' }}
          </el-tag>
        </div>
      </div>
    </div>

    <!-- Model Health Status -->
    <el-alert
      v-if="modelHealth === 'unhealthy'"
      title="Model Health Warning"
      type="error"
      :closable="false"
      show-icon
      class="health-alert"
    >
      Current model performance is below acceptable thresholds. Please investigate.
    </el-alert>

    <!-- Metrics Overview -->
    <MetricsOverview />

    <!-- Charts Grid -->
    <el-row :gutter="20">
      <el-col :xs="24" :lg="12">
        <MetricChart title="Accuracy Over Time" metric-name="accuracy" />
      </el-col>
      <el-col :xs="24" :lg="12">
        <MetricChart title="Latency (P99)" metric-name="latency_p99" />
      </el-col>
    </el-row>

    <el-row :gutter="20" style="margin-top: 20px;">
      <el-col :xs="24" :lg="12">
        <MetricChart title="Throughput" metric-name="throughput" />
      </el-col>
      <el-col :xs="24" :lg="12">
        <MetricChart title="Error Rate" metric-name="error_rate" />
      </el-col>
    </el-row>

    <!-- Alerts Drawer -->
    <el-drawer
      v-model="showAlerts"
      title="Alerts"
      direction="rtl"
      size="400px"
    >
      <AlertsList />
    </el-drawer>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { storeToRefs } from 'pinia';
import { useMonitoringStore } from '@/stores/monitoring';
import { useMetrics } from '@/composables/useMetrics';
import MetricsOverview from '@/components/MetricsOverview.vue';
import MetricChart from '@/components/MetricChart.vue';
import AlertsList from '@/components/AlertsList.vue';
import { Bell, Connection } from '@element-plus/icons-vue';

const store = useMonitoringStore();
const {
  models,
  selectedModel,
  unacknowledgedAlerts,
  modelHealth,
  isConnected,
} = storeToRefs(store);

const { startAutoRefresh, stopAutoRefresh } = useMetrics();
const showAlerts = ref(false);

const handleModelChange = (modelId: string) => {
  store.setSelectedModel(modelId);
};

onMounted(async () => {
  await store.fetchModels();
  store.connectWebSocket();
  startAutoRefresh(30000); // Refresh every 30 seconds
});

onUnmounted(() => {
  store.disconnectWebSocket();
  stopAutoRefresh();
});
</script>

<style scoped>
.dashboard-container {
  padding: 20px;
}

.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
}

.dashboard-header h1 {
  margin: 0;
  font-size: 28px;
  color: #303133;
}

.header-controls {
  display: flex;
  gap: 15px;
  align-items: center;
}

.connection-status {
  margin-left: 10px;
}

.health-alert {
  margin-bottom: 20px;
}
</style>
```

## Expected Outputs

1. **Real-time Monitoring Dashboard**:
   - Live metrics updating via WebSocket
   - Multiple visualization panels
   - Model health indicators
   - Alert notifications

2. **Interactive Charts**:
   - Time-series line charts with zoom
   - Selectable time ranges
   - Smooth animations
   - Responsive layouts

3. **Alert Management**:
   - Real-time alert notifications
   - Alert severity indicators
   - Acknowledgment functionality
   - Alert history

4. **Performance**:
   - Efficient state management
   - Optimized chart rendering
   - Proper cleanup and memory management

## Bonus Challenges

1. **Advanced Analytics**: Add statistical analysis (moving averages, anomaly detection)
2. **Model Comparison**: Side-by-side comparison of multiple models
3. **Custom Dashboards**: User-configurable dashboard layouts
4. **Export Reports**: Generate PDF/CSV reports of metrics
5. **Threshold Configuration**: User-defined alert thresholds
6. **Dark Mode**: Implement theme switching
7. **Mobile App**: Create companion mobile app with Capacitor
8. **Predictive Alerts**: ML-based alert prediction

## Resources

- [Vue 3 Documentation](https://vuejs.org/)
- [Pinia Documentation](https://pinia.vuejs.org/)
- [Element Plus](https://element-plus.org/)
- [Chart.js](https://www.chartjs.org/)
- [Socket.IO](https://socket.io/)
- [Day.js](https://day.js.org/)

## Success Criteria

### Functionality (40%)
- [ ] Real-time metrics display and update
- [ ] WebSocket connection management
- [ ] Time range selection working
- [ ] Alert system functional
- [ ] Model switching works correctly

### Code Quality (30%)
- [ ] Proper TypeScript typing
- [ ] Composables for reusable logic
- [ ] Clean component structure
- [ ] Pinia store organization
- [ ] Error handling

### User Experience (20%)
- [ ] Responsive design
- [ ] Smooth animations
- [ ] Intuitive navigation
- [ ] Clear data visualization
- [ ] Loading states

### Best Practices (10%)
- [ ] Proper cleanup on unmount
- [ ] WebSocket reconnection logic
- [ ] Memory leak prevention
- [ ] Accessibility considerations
