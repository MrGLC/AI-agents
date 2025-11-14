# Project 10: Real-Time Analytics Dashboard

## Overview
Build a comprehensive real-time analytics dashboard with live data updates, interactive charts, data filtering, and customizable widgets. This project focuses on handling streaming data, creating complex visualizations, and building a performant dashboard similar to Google Analytics or Mixpanel.

## Difficulty Level
Advanced

## Learning Objectives
- Implement real-time data updates with WebSocket or polling
- Create interactive data visualizations with Recharts
- Build customizable dashboard layouts with drag-and-drop
- Handle large datasets with virtualization
- Implement advanced filtering and date range selection
- Create exportable reports
- Optimize performance for real-time updates
- Build responsive dashboard grids
- Implement data aggregation and calculations
- Handle time-series data efficiently

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Data Fetching**: React Query with polling/WebSocket
- **Charts**: Recharts or Chart.js
- **Dashboard Layout**: react-grid-layout
- **State Management**: Zustand
- **Date Handling**: date-fns and date-fns-tz
- **Tables**: TanStack Table (React Table v8)
- **Export**: jsPDF and xlsx
- **Styling**: Tailwind CSS
- **Icons**: Lucide React
- **Build Tool**: Vite

## Project Requirements

### 1. Dashboard Features
- **Main Dashboard**
  - Overview metrics (KPIs)
  - Real-time visitor count
  - Page views chart
  - User demographics
  - Traffic sources
  - Device breakdown
  - Geographic map
  - Conversion funnel

- **Customization**
  - Drag-and-drop widgets
  - Resize widgets
  - Add/remove widgets
  - Save dashboard layouts
  - Multiple dashboard templates
  - Widget settings

### 2. Data Visualizations
- **Charts**
  - Line charts (time series)
  - Bar charts (comparisons)
  - Pie charts (distributions)
  - Area charts (trends)
  - Heatmaps (activity)
  - Funnel charts (conversions)
  - Gauge charts (single metrics)

- **Chart Features**
  - Interactive tooltips
  - Zoom and pan
  - Data point selection
  - Export as image
  - Drill-down capabilities
  - Comparative views
  - Custom date ranges

### 3. Metrics and KPIs
- **User Metrics**
  - Total users
  - Active users
  - New users
  - Returning users
  - User retention rate
  - Session duration
  - Bounce rate

- **Performance Metrics**
  - Page load time
  - API response time
  - Error rate
  - Success rate
  - Throughput
  - Latency percentiles

- **Business Metrics**
  - Revenue
  - Conversion rate
  - Average order value
  - Customer lifetime value
  - Churn rate

### 4. Filtering and Controls
- **Date Range Selector**
  - Today, Yesterday, Last 7 days
  - Last 30 days, Last 90 days
  - Custom date range
  - Compare periods
  - Real-time toggle

- **Filters**
  - Country/region
  - Device type
  - Browser
  - Operating system
  - Traffic source
  - Campaign
  - User segment

- **Grouping**
  - By hour, day, week, month
  - By device, browser, location
  - Custom grouping

### 5. Data Tables
- Sortable columns
- Pagination
- Search/filter
- Column visibility
- Export to CSV/Excel
- Row selection
- Inline editing
- Expandable rows

### 6. Real-Time Features
- Live visitor count
- Real-time events stream
- Active pages
- Current conversions
- Alert notifications
- Auto-refresh intervals
- Connection status indicator

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest analytics-dashboard -- --template react-ts
cd analytics-dashboard

npm install @tanstack/react-query recharts
npm install @tanstack/react-table
npm install react-grid-layout
npm install zustand
npm install date-fns
npm install lucide-react
npm install jspdf xlsx
npm install -D tailwindcss postcss autoprefixer @types/react-grid-layout
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/analytics.ts
export interface Metric {
  name: string;
  value: number;
  change: number;
  changeType: 'increase' | 'decrease';
  trend: number[];
}

export interface TimeSeriesData {
  timestamp: string;
  value: number;
  category?: string;
}

export interface DashboardWidget {
  id: string;
  type: WidgetType;
  title: string;
  config: WidgetConfig;
  layout: {
    x: number;
    y: number;
    w: number;
    h: number;
  };
}

export type WidgetType =
  | 'metric'
  | 'line-chart'
  | 'bar-chart'
  | 'pie-chart'
  | 'area-chart'
  | 'table'
  | 'heatmap'
  | 'funnel'
  | 'map'
  | 'realtime';

export interface WidgetConfig {
  metric?: string;
  chartType?: string;
  dataSource?: string;
  filters?: Filter[];
  groupBy?: string;
  timeRange?: TimeRange;
}

export interface Filter {
  field: string;
  operator: 'equals' | 'contains' | 'greater' | 'less';
  value: any;
}

export interface TimeRange {
  start: string;
  end: string;
  preset?: 'today' | 'yesterday' | '7d' | '30d' | '90d' | 'custom';
}

export interface AnalyticsEvent {
  id: string;
  timestamp: string;
  eventType: string;
  userId?: string;
  sessionId: string;
  properties: Record<string, any>;
  device: {
    type: 'desktop' | 'mobile' | 'tablet';
    browser: string;
    os: string;
  };
  location: {
    country: string;
    region: string;
    city: string;
  };
}

export interface UserSession {
  sessionId: string;
  userId?: string;
  startTime: string;
  endTime?: string;
  duration?: number;
  pageViews: number;
  events: number;
  device: string;
  location: string;
  referrer?: string;
  converted: boolean;
}

export interface PageView {
  id: string;
  timestamp: string;
  page: string;
  title: string;
  sessionId: string;
  duration?: number;
  exitPage: boolean;
}

export interface ConversionFunnel {
  steps: FunnelStep[];
  totalUsers: number;
  conversionRate: number;
}

export interface FunnelStep {
  name: string;
  users: number;
  percentage: number;
  dropOff: number;
}

export interface GeographicData {
  country: string;
  users: number;
  sessions: number;
  bounceRate: number;
  avgDuration: number;
}
```

### Step 3: Create Analytics Store
```typescript
// src/store/analyticsStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { DashboardWidget, TimeRange, Filter } from '@/types/analytics';

interface AnalyticsState {
  widgets: DashboardWidget[];
  timeRange: TimeRange;
  filters: Filter[];
  autoRefresh: boolean;
  refreshInterval: number;
  compareMode: boolean;
  compareTimeRange?: TimeRange;

  addWidget: (widget: DashboardWidget) => void;
  removeWidget: (widgetId: string) => void;
  updateWidget: (widgetId: string, updates: Partial<DashboardWidget>) => void;
  updateWidgetLayout: (widgetId: string, layout: DashboardWidget['layout']) => void;

  setTimeRange: (timeRange: TimeRange) => void;
  setFilters: (filters: Filter[]) => void;
  addFilter: (filter: Filter) => void;
  removeFilter: (index: number) => void;

  setAutoRefresh: (enabled: boolean) => void;
  setRefreshInterval: (interval: number) => void;
  setCompareMode: (enabled: boolean, compareTimeRange?: TimeRange) => void;
}

export const useAnalyticsStore = create<AnalyticsState>()(
  persist(
    (set) => ({
      widgets: [],
      timeRange: {
        start: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString(),
        end: new Date().toISOString(),
        preset: '7d',
      },
      filters: [],
      autoRefresh: false,
      refreshInterval: 30000, // 30 seconds
      compareMode: false,

      addWidget: (widget) => {
        set((state) => ({
          widgets: [...state.widgets, widget],
        }));
      },

      removeWidget: (widgetId) => {
        set((state) => ({
          widgets: state.widgets.filter((w) => w.id !== widgetId),
        }));
      },

      updateWidget: (widgetId, updates) => {
        set((state) => ({
          widgets: state.widgets.map((w) =>
            w.id === widgetId ? { ...w, ...updates } : w
          ),
        }));
      },

      updateWidgetLayout: (widgetId, layout) => {
        set((state) => ({
          widgets: state.widgets.map((w) =>
            w.id === widgetId ? { ...w, layout } : w
          ),
        }));
      },

      setTimeRange: (timeRange) => {
        set({ timeRange });
      },

      setFilters: (filters) => {
        set({ filters });
      },

      addFilter: (filter) => {
        set((state) => ({
          filters: [...state.filters, filter],
        }));
      },

      removeFilter: (index) => {
        set((state) => ({
          filters: state.filters.filter((_, i) => i !== index),
        }));
      },

      setAutoRefresh: (enabled) => {
        set({ autoRefresh: enabled });
      },

      setRefreshInterval: (interval) => {
        set({ refreshInterval: interval });
      },

      setCompareMode: (enabled, compareTimeRange) => {
        set({ compareMode: enabled, compareTimeRange });
      },
    }),
    {
      name: 'analytics-storage',
    }
  )
);
```

### Step 4: Create Metric Card Component
```typescript
// src/components/MetricCard.tsx
import { TrendingUp, TrendingDown } from 'lucide-react';
import { LineChart, Line, ResponsiveContainer } from 'recharts';
import type { Metric } from '@/types/analytics';

interface MetricCardProps {
  metric: Metric;
}

export const MetricCard: React.FC<MetricCardProps> = ({ metric }) => {
  const isPositive = metric.changeType === 'increase';

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <div className="flex items-start justify-between mb-4">
        <div>
          <p className="text-sm text-gray-600 mb-1">{metric.name}</p>
          <p className="text-3xl font-bold text-gray-900">
            {metric.value.toLocaleString()}
          </p>
        </div>
        <div
          className={`flex items-center gap-1 px-2 py-1 rounded ${
            isPositive
              ? 'bg-green-100 text-green-700'
              : 'bg-red-100 text-red-700'
          }`}
        >
          {isPositive ? (
            <TrendingUp className="w-4 h-4" />
          ) : (
            <TrendingDown className="w-4 h-4" />
          )}
          <span className="text-sm font-semibold">
            {Math.abs(metric.change)}%
          </span>
        </div>
      </div>

      {/* Trend sparkline */}
      <div className="h-12">
        <ResponsiveContainer width="100%" height="100%">
          <LineChart data={metric.trend.map((value, index) => ({ value, index }))}>
            <Line
              type="monotone"
              dataKey="value"
              stroke={isPositive ? '#10b981' : '#ef4444'}
              strokeWidth={2}
              dot={false}
            />
          </LineChart>
        </ResponsiveContainer>
      </div>
    </div>
  );
};
```

### Step 5: Create Time Series Chart Component
```typescript
// src/components/TimeSeriesChart.tsx
import {
  LineChart,
  Line,
  AreaChart,
  Area,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  Legend,
} from 'recharts';
import { format } from 'date-fns';
import type { TimeSeriesData } from '@/types/analytics';

interface TimeSeriesChartProps {
  data: TimeSeriesData[];
  title: string;
  type?: 'line' | 'area';
  dataKey?: string;
  compareData?: TimeSeriesData[];
}

export const TimeSeriesChart: React.FC<TimeSeriesChartProps> = ({
  data,
  title,
  type = 'line',
  dataKey = 'value',
  compareData,
}) => {
  const Chart = type === 'area' ? AreaChart : LineChart;
  const DataComponent = type === 'area' ? Area : Line;

  const formatXAxis = (timestamp: string) => {
    return format(new Date(timestamp), 'MMM dd');
  };

  const formatTooltip = (value: number) => {
    return value.toLocaleString();
  };

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      <ResponsiveContainer width="100%" height={300}>
        <Chart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />
          <XAxis
            dataKey="timestamp"
            tickFormatter={formatXAxis}
            stroke="#9ca3af"
          />
          <YAxis stroke="#9ca3af" />
          <Tooltip
            formatter={formatTooltip}
            labelFormatter={(label) =>
              format(new Date(label), 'MMM dd, yyyy HH:mm')
            }
            contentStyle={{
              backgroundColor: 'rgba(255, 255, 255, 0.95)',
              border: '1px solid #e5e7eb',
              borderRadius: '0.5rem',
            }}
          />
          <Legend />
          <DataComponent
            type="monotone"
            dataKey={dataKey}
            stroke="#3b82f6"
            fill="#93c5fd"
            strokeWidth={2}
            name="Current Period"
          />
          {compareData && (
            <DataComponent
              type="monotone"
              data={compareData}
              dataKey={dataKey}
              stroke="#9ca3af"
              fill="#d1d5db"
              strokeWidth={2}
              strokeDasharray="5 5"
              name="Previous Period"
            />
          )}
        </Chart>
      </ResponsiveContainer>
    </div>
  );
};
```

### Step 6: Create Real-Time Events Component
```typescript
// src/components/RealTimeEvents.tsx
import { useEffect, useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Circle, User, Globe, Monitor } from 'lucide-react';
import { formatDistanceToNow } from 'date-fns';
import type { AnalyticsEvent } from '@/types/analytics';

export const RealTimeEvents: React.FC = () => {
  const [events, setEvents] = useState<AnalyticsEvent[]>([]);

  // Simulate real-time events (replace with actual WebSocket)
  useEffect(() => {
    const interval = setInterval(() => {
      const newEvent: AnalyticsEvent = {
        id: Date.now().toString(),
        timestamp: new Date().toISOString(),
        eventType: ['page_view', 'click', 'signup', 'purchase'][
          Math.floor(Math.random() * 4)
        ],
        sessionId: Math.random().toString(36),
        properties: {},
        device: {
          type: ['desktop', 'mobile', 'tablet'][Math.floor(Math.random() * 3)] as any,
          browser: 'Chrome',
          os: 'Windows',
        },
        location: {
          country: ['US', 'UK', 'CA', 'DE'][Math.floor(Math.random() * 4)],
          region: 'CA',
          city: 'San Francisco',
        },
      };

      setEvents((prev) => [newEvent, ...prev].slice(0, 50));
    }, 2000);

    return () => clearInterval(interval);
  }, []);

  const getEventIcon = (eventType: string) => {
    switch (eventType) {
      case 'page_view':
        return <Globe className="w-4 h-4" />;
      case 'signup':
        return <User className="w-4 h-4" />;
      default:
        return <Circle className="w-4 h-4" />;
    }
  };

  const getEventColor = (eventType: string) => {
    switch (eventType) {
      case 'page_view':
        return 'text-blue-600';
      case 'signup':
        return 'text-green-600';
      case 'purchase':
        return 'text-purple-600';
      default:
        return 'text-gray-600';
    }
  };

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <div className="flex items-center justify-between mb-4">
        <h3 className="text-lg font-semibold">Real-Time Events</h3>
        <div className="flex items-center gap-2 text-green-600">
          <div className="w-2 h-2 bg-green-600 rounded-full animate-pulse" />
          <span className="text-sm font-medium">Live</span>
        </div>
      </div>

      <div className="space-y-3 max-h-96 overflow-y-auto">
        {events.map((event) => (
          <div
            key={event.id}
            className="flex items-start gap-3 p-3 hover:bg-gray-50 rounded-lg transition-colors"
          >
            <div className={`mt-1 ${getEventColor(event.eventType)}`}>
              {getEventIcon(event.eventType)}
            </div>
            <div className="flex-1 min-w-0">
              <div className="flex items-center justify-between mb-1">
                <p className="font-medium text-sm capitalize">
                  {event.eventType.replace('_', ' ')}
                </p>
                <span className="text-xs text-gray-500">
                  {formatDistanceToNow(new Date(event.timestamp), {
                    addSuffix: true,
                  })}
                </span>
              </div>
              <div className="flex items-center gap-3 text-xs text-gray-600">
                <span className="flex items-center gap-1">
                  <Monitor className="w-3 h-3" />
                  {event.device.type}
                </span>
                <span className="flex items-center gap-1">
                  <Globe className="w-3 h-3" />
                  {event.location.country}
                </span>
              </div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );
};
```

### Step 7: Create Dashboard Layout
```typescript
// src/pages/Dashboard.tsx
import { useState } from 'react';
import GridLayout from 'react-grid-layout';
import 'react-grid-layout/css/styles.css';
import 'react-resizable/css/styles.css';
import { MetricCard } from '@/components/MetricCard';
import { TimeSeriesChart } from '@/components/TimeSeriesChart';
import { RealTimeEvents } from '@/components/RealTimeEvents';
import { useAnalyticsStore } from '@/store/analyticsStore';
import { Calendar, Filter, Download, RefreshCw } from 'lucide-react';

// Mock data
const mockMetrics = [
  {
    name: 'Total Users',
    value: 12543,
    change: 12.5,
    changeType: 'increase' as const,
    trend: [100, 120, 115, 140, 135, 160, 155],
  },
  {
    name: 'Sessions',
    value: 28901,
    change: 8.2,
    changeType: 'increase' as const,
    trend: [200, 220, 210, 240, 235, 260, 255],
  },
  {
    name: 'Bounce Rate',
    value: 42,
    change: -3.1,
    changeType: 'decrease' as const,
    trend: [50, 48, 46, 44, 43, 42, 42],
  },
  {
    name: 'Avg. Duration',
    value: 245,
    change: 15.3,
    changeType: 'increase' as const,
    trend: [200, 210, 215, 225, 230, 240, 245],
  },
];

const mockTimeSeriesData = Array.from({ length: 30 }, (_, i) => ({
  timestamp: new Date(Date.now() - (29 - i) * 24 * 60 * 60 * 1000).toISOString(),
  value: Math.floor(Math.random() * 1000) + 500,
}));

export const Dashboard: React.FC = () => {
  const { timeRange, setTimeRange, autoRefresh, setAutoRefresh } =
    useAnalyticsStore();
  const [isRefreshing, setIsRefreshing] = useState(false);

  const handleRefresh = async () => {
    setIsRefreshing(true);
    // Simulate data refresh
    await new Promise((resolve) => setTimeout(resolve, 1000));
    setIsRefreshing(false);
  };

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <header className="bg-white shadow-sm">
        <div className="max-w-7xl mx-auto px-4 py-4">
          <div className="flex items-center justify-between">
            <h1 className="text-2xl font-bold text-gray-900">
              Analytics Dashboard
            </h1>

            <div className="flex items-center gap-4">
              {/* Date Range Picker */}
              <button className="flex items-center gap-2 px-4 py-2 border rounded-lg hover:bg-gray-50">
                <Calendar className="w-4 h-4" />
                <span className="text-sm">Last 7 days</span>
              </button>

              {/* Filters */}
              <button className="flex items-center gap-2 px-4 py-2 border rounded-lg hover:bg-gray-50">
                <Filter className="w-4 h-4" />
                <span className="text-sm">Filters</span>
              </button>

              {/* Auto Refresh */}
              <button
                onClick={() => setAutoRefresh(!autoRefresh)}
                className={`flex items-center gap-2 px-4 py-2 border rounded-lg ${
                  autoRefresh ? 'bg-blue-50 border-blue-500' : 'hover:bg-gray-50'
                }`}
              >
                <RefreshCw
                  className={`w-4 h-4 ${autoRefresh ? 'animate-spin' : ''}`}
                />
                <span className="text-sm">Auto Refresh</span>
              </button>

              {/* Export */}
              <button className="flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700">
                <Download className="w-4 h-4" />
                <span className="text-sm">Export</span>
              </button>
            </div>
          </div>
        </div>
      </header>

      {/* Dashboard Content */}
      <main className="max-w-7xl mx-auto px-4 py-8">
        {/* Metrics Grid */}
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
          {mockMetrics.map((metric) => (
            <MetricCard key={metric.name} metric={metric} />
          ))}
        </div>

        {/* Charts Grid */}
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-8">
          <TimeSeriesChart
            data={mockTimeSeriesData}
            title="Page Views"
            type="area"
          />
          <TimeSeriesChart
            data={mockTimeSeriesData}
            title="User Sessions"
            type="line"
          />
        </div>

        {/* Real-Time Events */}
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
          <div className="lg:col-span-2">
            <TimeSeriesChart
              data={mockTimeSeriesData}
              title="Traffic Sources"
              type="area"
            />
          </div>
          <RealTimeEvents />
        </div>
      </main>
    </div>
  );
};
```

## Expected Outputs

1. **Comprehensive Analytics Dashboard** with:
   - Real-time metrics and KPIs
   - Interactive charts and visualizations
   - Customizable layout
   - Live data updates

2. **Data Visualization**:
   - Multiple chart types
   - Time series analysis
   - Comparison views
   - Interactive tooltips

3. **Filtering and Controls**:
   - Date range selection
   - Advanced filters
   - Auto-refresh toggle
   - Export functionality

4. **Performance**:
   - Smooth real-time updates
   - Optimized rendering
   - Efficient data handling
   - Fast interactions

## Bonus Challenges

- [ ] Add custom dashboard templates
- [ ] Implement data export to PDF
- [ ] Add email report scheduling
- [ ] Create custom alert rules
- [ ] Implement cohort analysis
- [ ] Add A/B test tracking
- [ ] Create user flow visualization
- [ ] Add predictive analytics
- [ ] Implement data retention policies
- [ ] Add custom metrics builder
- [ ] Create API for external integrations
- [ ] Add team collaboration features
- [ ] Implement role-based access control
- [ ] Add data sampling for large datasets

## Resources

- [Recharts Documentation](https://recharts.org/)
- [TanStack Table](https://tanstack.com/table/latest)
- [React Grid Layout](https://github.com/react-grid-layout/react-grid-layout)
- [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [date-fns](https://date-fns.org/)

## Success Criteria

- Real-time data updates work smoothly
- All charts render correctly
- Dashboard layout is customizable
- Filters apply to all widgets
- Export functionality works
- Auto-refresh toggles correctly
- Performance is optimized (60fps)
- Data is accurate and consistent
- UI is responsive on all screens
- Loading states are clear
- Error handling is comprehensive
- TypeScript provides full type safety
- Accessibility standards are met
- Code is well-organized and maintainable
