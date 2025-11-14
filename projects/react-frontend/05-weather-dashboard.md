# Project 5: Weather Dashboard with Charts and Maps

## Overview
Build a comprehensive weather dashboard that displays current conditions, forecasts, and historical data with interactive charts and maps. This project focuses on data visualization, geolocation APIs, and creating an informative weather application similar to Weather.com or Dark Sky.

## Difficulty Level
Advanced

## Learning Objectives
- Integrate multiple weather APIs (OpenWeatherMap, WeatherAPI)
- Create interactive charts with Recharts or Chart.js
- Implement map integration with Leaflet or Mapbox
- Handle geolocation and location search
- Visualize time-series weather data
- Create responsive dashboard layouts
- Manage API rate limiting and caching
- Display real-time weather updates
- Implement unit conversions (Celsius/Fahrenheit)

## Technical Stack
- **Framework**: React 18+ with TypeScript
- **Data Fetching**: React Query with polling for updates
- **Charts**: Recharts or Chart.js
- **Maps**: React Leaflet or Mapbox GL JS
- **Styling**: Tailwind CSS
- **Icons**: Lucide React + custom weather icons
- **Date/Time**: date-fns and date-fns-tz
- **Geolocation**: Browser Geolocation API
- **Weather API**: OpenWeatherMap API
- **Build Tool**: Vite

## Project Requirements

### 1. Current Weather Display
- **Overview Card**
  - Current temperature with feels-like
  - Weather condition (clear, cloudy, rainy, etc.)
  - Weather icon/animation
  - Location name with country
  - Last updated time
  - Sunrise/sunset times

- **Detailed Metrics**
  - Humidity percentage
  - Wind speed and direction
  - Visibility
  - UV index
  - Pressure
  - Precipitation probability
  - Dew point
  - Cloud coverage

### 2. Forecast Features
- **Hourly Forecast**
  - Next 24-48 hours
  - Temperature chart
  - Precipitation probability
  - Wind speed
  - Scrollable timeline

- **Daily Forecast**
  - 7-day or 14-day forecast
  - High/low temperatures
  - Weather conditions
  - Precipitation chance
  - Detailed day view

### 3. Data Visualization
- **Temperature Chart**
  - Line chart showing temperature trends
  - Min/max ranges
  - Toggle hourly/daily view

- **Precipitation Chart**
  - Bar chart for rainfall
  - Probability indicators

- **Wind Chart**
  - Wind speed over time
  - Wind direction compass

- **Additional Charts**
  - Humidity trends
  - Pressure changes
  - UV index timeline

### 4. Map Integration
- **Weather Map**
  - Temperature overlay
  - Precipitation radar
  - Cloud coverage
  - Wind patterns
  - Interactive zoom/pan

- **Location Features**
  - Current location detection
  - Search for cities
  - Multiple location tracking
  - Favorite locations

### 5. UI Components
- Location search with autocomplete
- Unit toggle (°C/°F, km/h, mph)
- Theme toggle (light/dark)
- Responsive dashboard grid
- Loading skeletons
- Error states
- Refresh button

## Step-by-Step Implementation

### Step 1: Project Setup
```bash
npm create vite@latest weather-dashboard -- --template react-ts
cd weather-dashboard

npm install @tanstack/react-query axios
npm install recharts
npm install react-leaflet leaflet
npm install date-fns date-fns-tz
npm install lucide-react
npm install -D tailwindcss postcss autoprefixer @types/leaflet
npx tailwindcss init -p
```

### Step 2: Define TypeScript Types
```typescript
// src/types/weather.ts
export interface Coordinates {
  lat: number;
  lon: number;
}

export interface Location {
  name: string;
  country: string;
  state?: string;
  coordinates: Coordinates;
}

export interface CurrentWeather {
  temp: number;
  feelsLike: number;
  tempMin: number;
  tempMax: number;
  pressure: number;
  humidity: number;
  visibility: number;
  windSpeed: number;
  windDeg: number;
  clouds: number;
  uvIndex: number;
  dewPoint: number;
  sunrise: number;
  sunset: number;
  weather: {
    id: number;
    main: string;
    description: string;
    icon: string;
  };
  dt: number;
}

export interface HourlyForecast {
  dt: number;
  temp: number;
  feelsLike: number;
  pressure: number;
  humidity: number;
  windSpeed: number;
  windDeg: number;
  pop: number; // Probability of precipitation
  weather: {
    id: number;
    main: string;
    description: string;
    icon: string;
  };
}

export interface DailyForecast {
  dt: number;
  sunrise: number;
  sunset: number;
  temp: {
    day: number;
    min: number;
    max: number;
    night: number;
    eve: number;
    morn: number;
  };
  feelsLike: {
    day: number;
    night: number;
    eve: number;
    morn: number;
  };
  pressure: number;
  humidity: number;
  windSpeed: number;
  windDeg: number;
  pop: number;
  rain?: number;
  snow?: number;
  weather: {
    id: number;
    main: string;
    description: string;
    icon: string;
  };
}

export interface WeatherData {
  location: Location;
  current: CurrentWeather;
  hourly: HourlyForecast[];
  daily: DailyForecast[];
  timezone: string;
}

export type TemperatureUnit = 'celsius' | 'fahrenheit';
export type SpeedUnit = 'kmh' | 'mph';
```

### Step 3: Create Weather API Service
```typescript
// src/services/weatherApi.ts
import axios from 'axios';
import type { WeatherData, Coordinates } from '@/types/weather';

const API_KEY = import.meta.env.VITE_OPENWEATHER_API_KEY;
const BASE_URL = 'https://api.openweathermap.org/data/3.0/onecall';
const GEO_URL = 'https://api.openweathermap.org/geo/1.0';

export const weatherApi = {
  getWeatherByCoords: async (coords: Coordinates): Promise<WeatherData> => {
    const response = await axios.get(BASE_URL, {
      params: {
        lat: coords.lat,
        lon: coords.lon,
        appid: API_KEY,
        units: 'metric',
        exclude: 'minutely,alerts',
      },
    });

    return transformWeatherData(response.data, coords);
  },

  searchLocations: async (query: string) => {
    const response = await axios.get(`${GEO_URL}/direct`, {
      params: {
        q: query,
        limit: 5,
        appid: API_KEY,
      },
    });

    return response.data.map((item: any) => ({
      name: item.name,
      country: item.country,
      state: item.state,
      coordinates: {
        lat: item.lat,
        lon: item.lon,
      },
    }));
  },

  getCurrentLocation: (): Promise<Coordinates> => {
    return new Promise((resolve, reject) => {
      if (!navigator.geolocation) {
        reject(new Error('Geolocation is not supported'));
        return;
      }

      navigator.geolocation.getCurrentPosition(
        (position) => {
          resolve({
            lat: position.coords.latitude,
            lon: position.coords.longitude,
          });
        },
        (error) => {
          reject(error);
        }
      );
    });
  },
};

function transformWeatherData(data: any, coords: Coordinates): WeatherData {
  return {
    location: {
      name: data.timezone.split('/')[1].replace('_', ' '),
      country: '',
      coordinates: coords,
    },
    current: {
      temp: data.current.temp,
      feelsLike: data.current.feels_like,
      tempMin: data.daily[0].temp.min,
      tempMax: data.daily[0].temp.max,
      pressure: data.current.pressure,
      humidity: data.current.humidity,
      visibility: data.current.visibility,
      windSpeed: data.current.wind_speed,
      windDeg: data.current.wind_deg,
      clouds: data.current.clouds,
      uvIndex: data.current.uvi,
      dewPoint: data.current.dew_point,
      sunrise: data.current.sunrise,
      sunset: data.current.sunset,
      weather: data.current.weather[0],
      dt: data.current.dt,
    },
    hourly: data.hourly.slice(0, 24).map((hour: any) => ({
      dt: hour.dt,
      temp: hour.temp,
      feelsLike: hour.feels_like,
      pressure: hour.pressure,
      humidity: hour.humidity,
      windSpeed: hour.wind_speed,
      windDeg: hour.wind_deg,
      pop: hour.pop,
      weather: hour.weather[0],
    })),
    daily: data.daily.map((day: any) => ({
      dt: day.dt,
      sunrise: day.sunrise,
      sunset: day.sunset,
      temp: day.temp,
      feelsLike: day.feels_like,
      pressure: day.pressure,
      humidity: day.humidity,
      windSpeed: day.wind_speed,
      windDeg: day.wind_deg,
      pop: day.pop,
      rain: day.rain,
      snow: day.snow,
      weather: day.weather[0],
    })),
    timezone: data.timezone,
  };
}
```

### Step 4: Create Weather Hooks
```typescript
// src/hooks/useWeather.ts
import { useQuery } from '@tanstack/react-query';
import { weatherApi } from '@/services/weatherApi';
import type { Coordinates } from '@/types/weather';

export const useWeather = (coords?: Coordinates) => {
  return useQuery({
    queryKey: ['weather', coords],
    queryFn: () => weatherApi.getWeatherByCoords(coords!),
    enabled: !!coords,
    staleTime: 1000 * 60 * 5, // 5 minutes
    refetchInterval: 1000 * 60 * 10, // Refresh every 10 minutes
  });
};

export const useCurrentLocation = () => {
  return useQuery({
    queryKey: ['currentLocation'],
    queryFn: weatherApi.getCurrentLocation,
    staleTime: Infinity,
    retry: false,
  });
};
```

### Step 5: Create Current Weather Card
```typescript
// src/components/CurrentWeatherCard.tsx
import { format } from 'date-fns';
import { Sunrise, Sunset, Wind, Droplets, Eye, Gauge } from 'lucide-react';
import type { CurrentWeather, Location } from '@/types/weather';

interface CurrentWeatherCardProps {
  weather: CurrentWeather;
  location: Location;
  unit: 'celsius' | 'fahrenheit';
}

export const CurrentWeatherCard: React.FC<CurrentWeatherCardProps> = ({
  weather,
  location,
  unit,
}) => {
  const temp = unit === 'fahrenheit'
    ? (weather.temp * 9/5 + 32).toFixed(1)
    : weather.temp.toFixed(1);

  const feelsLike = unit === 'fahrenheit'
    ? (weather.feelsLike * 9/5 + 32).toFixed(1)
    : weather.feelsLike.toFixed(1);

  return (
    <div className="bg-gradient-to-br from-blue-500 to-blue-700 rounded-xl p-8 text-white shadow-lg">
      <div className="flex items-start justify-between mb-6">
        <div>
          <h2 className="text-3xl font-bold mb-1">{location.name}</h2>
          <p className="text-blue-100">
            {format(new Date(weather.dt * 1000), 'EEEE, MMMM d, yyyy')}
          </p>
          <p className="text-blue-100 text-sm">
            {format(new Date(weather.dt * 1000), 'h:mm a')}
          </p>
        </div>
        <div className="text-right">
          <div className="text-6xl font-bold">{temp}°</div>
          <p className="text-blue-100">Feels like {feelsLike}°</p>
        </div>
      </div>

      <div className="flex items-center gap-3 mb-6">
        <img
          src={`https://openweathermap.org/img/wn/${weather.weather.icon}@2x.png`}
          alt={weather.weather.description}
          className="w-16 h-16"
        />
        <div>
          <p className="text-xl capitalize">{weather.weather.description}</p>
          <p className="text-blue-100 text-sm">
            H: {weather.tempMax.toFixed(0)}° L: {weather.tempMin.toFixed(0)}°
          </p>
        </div>
      </div>

      <div className="grid grid-cols-3 gap-4">
        <div className="bg-white/10 rounded-lg p-3">
          <div className="flex items-center gap-2 text-blue-100 mb-1">
            <Wind className="w-4 h-4" />
            <span className="text-xs">Wind</span>
          </div>
          <p className="text-xl font-semibold">{weather.windSpeed.toFixed(1)} m/s</p>
        </div>

        <div className="bg-white/10 rounded-lg p-3">
          <div className="flex items-center gap-2 text-blue-100 mb-1">
            <Droplets className="w-4 h-4" />
            <span className="text-xs">Humidity</span>
          </div>
          <p className="text-xl font-semibold">{weather.humidity}%</p>
        </div>

        <div className="bg-white/10 rounded-lg p-3">
          <div className="flex items-center gap-2 text-blue-100 mb-1">
            <Eye className="w-4 h-4" />
            <span className="text-xs">Visibility</span>
          </div>
          <p className="text-xl font-semibold">
            {(weather.visibility / 1000).toFixed(1)} km
          </p>
        </div>

        <div className="bg-white/10 rounded-lg p-3">
          <div className="flex items-center gap-2 text-blue-100 mb-1">
            <Gauge className="w-4 h-4" />
            <span className="text-xs">Pressure</span>
          </div>
          <p className="text-xl font-semibold">{weather.pressure} hPa</p>
        </div>

        <div className="bg-white/10 rounded-lg p-3">
          <div className="flex items-center gap-2 text-blue-100 mb-1">
            <Sunrise className="w-4 h-4" />
            <span className="text-xs">Sunrise</span>
          </div>
          <p className="text-xl font-semibold">
            {format(new Date(weather.sunrise * 1000), 'h:mm a')}
          </p>
        </div>

        <div className="bg-white/10 rounded-lg p-3">
          <div className="flex items-center gap-2 text-blue-100 mb-1">
            <Sunset className="w-4 h-4" />
            <span className="text-xs">Sunset</span>
          </div>
          <p className="text-xl font-semibold">
            {format(new Date(weather.sunset * 1000), 'h:mm a')}
          </p>
        </div>
      </div>

      <div className="mt-4 pt-4 border-t border-white/20 grid grid-cols-2 gap-4 text-sm">
        <div>
          <span className="text-blue-100">UV Index:</span>
          <span className="ml-2 font-semibold">{weather.uvIndex.toFixed(1)}</span>
        </div>
        <div>
          <span className="text-blue-100">Dew Point:</span>
          <span className="ml-2 font-semibold">{weather.dewPoint.toFixed(1)}°</span>
        </div>
      </div>
    </div>
  );
};
```

### Step 6: Create Temperature Chart
```typescript
// src/components/TemperatureChart.tsx
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  Area,
  AreaChart,
} from 'recharts';
import { format } from 'date-fns';
import type { HourlyForecast } from '@/types/weather';

interface TemperatureChartProps {
  hourly: HourlyForecast[];
  unit: 'celsius' | 'fahrenheit';
}

export const TemperatureChart: React.FC<TemperatureChartProps> = ({
  hourly,
  unit,
}) => {
  const data = hourly.map((hour) => ({
    time: format(new Date(hour.dt * 1000), 'ha'),
    temp: unit === 'fahrenheit' ? hour.temp * 9/5 + 32 : hour.temp,
    feelsLike: unit === 'fahrenheit' ? hour.feelsLike * 9/5 + 32 : hour.feelsLike,
  }));

  return (
    <div className="bg-white rounded-xl p-6 shadow-lg">
      <h3 className="text-xl font-bold mb-4">24-Hour Temperature</h3>
      <ResponsiveContainer width="100%" height={300}>
        <AreaChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis
            dataKey="time"
            tick={{ fontSize: 12 }}
            interval="preserveStartEnd"
          />
          <YAxis
            tick={{ fontSize: 12 }}
            label={{ value: `°${unit === 'fahrenheit' ? 'F' : 'C'}`, angle: -90, position: 'insideLeft' }}
          />
          <Tooltip
            contentStyle={{
              backgroundColor: 'rgba(255, 255, 255, 0.9)',
              border: '1px solid #ccc',
              borderRadius: '8px',
            }}
          />
          <Area
            type="monotone"
            dataKey="temp"
            stroke="#3b82f6"
            fill="#93c5fd"
            strokeWidth={2}
            name="Temperature"
          />
          <Line
            type="monotone"
            dataKey="feelsLike"
            stroke="#f59e0b"
            strokeWidth={2}
            strokeDasharray="5 5"
            dot={false}
            name="Feels Like"
          />
        </AreaChart>
      </ResponsiveContainer>
    </div>
  );
};
```

### Step 7: Create Daily Forecast Component
```typescript
// src/components/DailyForecast.tsx
import { format } from 'date-fns';
import type { DailyForecast } from '@/types/weather';

interface DailyForecastProps {
  daily: DailyForecast[];
  unit: 'celsius' | 'fahrenheit';
}

export const DailyForecastComponent: React.FC<DailyForecastProps> = ({
  daily,
  unit,
}) => {
  const convertTemp = (temp: number) => {
    return unit === 'fahrenheit' ? (temp * 9/5 + 32).toFixed(0) : temp.toFixed(0);
  };

  return (
    <div className="bg-white rounded-xl p-6 shadow-lg">
      <h3 className="text-xl font-bold mb-4">7-Day Forecast</h3>
      <div className="space-y-3">
        {daily.map((day, index) => (
          <div
            key={day.dt}
            className="flex items-center justify-between p-3 hover:bg-gray-50 rounded-lg transition-colors"
          >
            <div className="flex items-center gap-4 flex-1">
              <span className="font-medium w-24">
                {index === 0 ? 'Today' : format(new Date(day.dt * 1000), 'EEEE')}
              </span>
              <img
                src={`https://openweathermap.org/img/wn/${day.weather.icon}.png`}
                alt={day.weather.description}
                className="w-12 h-12"
              />
              <span className="text-gray-600 capitalize flex-1">
                {day.weather.description}
              </span>
            </div>

            <div className="flex items-center gap-4">
              <div className="text-blue-500 text-sm">
                {(day.pop * 100).toFixed(0)}%
              </div>
              <div className="flex gap-2 w-24 justify-end">
                <span className="font-semibold">
                  {convertTemp(day.temp.max)}°
                </span>
                <span className="text-gray-400">
                  {convertTemp(day.temp.min)}°
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

## Expected Outputs

1. **Complete Weather Dashboard** with:
   - Current weather with detailed metrics
   - Hourly and daily forecasts
   - Interactive charts
   - Weather map
   - Location search

2. **Data Visualizations**:
   - Temperature trend charts
   - Precipitation graphs
   - Wind direction compass
   - Humidity and pressure trends

3. **User Experience**:
   - Responsive design
   - Unit conversions
   - Auto-refresh
   - Location detection
   - Loading states

4. **Performance**:
   - Fast initial load
   - Efficient API caching
   - Smooth animations
   - Optimized re-renders

## Bonus Challenges

- [ ] Add weather alerts and warnings
- [ ] Implement air quality index display
- [ ] Add historical weather data comparison
- [ ] Create weather radar animation
- [ ] Add multiple location comparison
- [ ] Implement weather widgets
- [ ] Add severe weather notifications
- [ ] Create hourly precipitation chart
- [ ] Add moon phase information
- [ ] Implement pollen count display
- [ ] Add weather-based activity suggestions
- [ ] Create shareable weather cards
- [ ] Add offline support with cached data
- [ ] Implement weather trends analysis

## Resources

- [OpenWeatherMap API](https://openweathermap.org/api)
- [Recharts Documentation](https://recharts.org/)
- [React Leaflet](https://react-leaflet.js.org/)
- [Weather Icons](https://erikflowers.github.io/weather-icons/)
- [Geolocation API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API)

## Success Criteria

- Weather data displays accurately
- Charts render correctly with proper scales
- Map shows location and weather overlays
- Location search works with autocomplete
- Unit conversion toggles work properly
- Auto-refresh updates data periodically
- Loading states are informative
- Error handling covers all edge cases
- Mobile responsive on all screen sizes
- API rate limits are respected
- TypeScript types are comprehensive
- Performance is optimized (no unnecessary renders)
