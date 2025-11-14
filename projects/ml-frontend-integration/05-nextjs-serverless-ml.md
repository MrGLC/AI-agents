# Project 05: Next.js App with Serverless ML Functions

## Overview
Build a production-ready Next.js application with serverless ML inference using API routes and edge functions. This project demonstrates modern full-stack ML deployment patterns, combining server-side rendering, static generation, and serverless functions for scalable ML applications without managing infrastructure.

## Learning Objectives
- Master Next.js App Router and Server Components
- Implement serverless ML functions with API routes
- Deploy ML models to edge locations with Vercel Edge Functions
- Optimize image loading and preprocessing for ML
- Handle server-side ML inference efficiently
- Implement caching strategies for predictions
- Build hybrid rendering (SSR + SSG + CSR)
- Apply Next.js performance optimizations

## Difficulty Level
**Advanced** - Requires strong understanding of Next.js, serverless architecture, Python ML deployment, and full-stack development.

## Technical Stack
- **Frontend**: Next.js 14+ (App Router)
- **Backend**: Next.js API Routes, Vercel Serverless Functions
- **ML Runtime**: Python (with Vercel Python Runtime) or TensorFlow.js
- **State Management**: React Context + SWR
- **Database**: Vercel KV (Redis) for caching
- **Storage**: Vercel Blob for model storage
- **Deployment**: Vercel
- **Styling**: Tailwind CSS
- **Testing**: Jest, React Testing Library, Playwright

## Requirements

### Architecture Requirements
1. API Routes for ML inference endpoints
2. Edge Functions for low-latency predictions
3. Server Components for initial data fetching
4. Client Components for interactive UI
5. Middleware for request validation
6. Caching layer for predictions
7. Rate limiting for API protection

### ML Deployment Requirements
1. Multiple model support (versioning)
2. Model warmup and cold start optimization
3. Batch prediction support
4. Streaming responses for long-running predictions
5. Model monitoring and logging
6. Error handling and fallbacks
7. A/B testing infrastructure

### Performance Requirements
1. Edge function response < 50ms
2. API route response < 500ms
3. Optimized bundle sizes
4. Image optimization with Next.js Image
5. Incremental Static Regeneration (ISR)
6. Client-side caching with SWR

### Features
1. Image upload and classification
2. Text analysis and sentiment prediction
3. Batch processing interface
4. Prediction history with pagination
5. Model comparison dashboard
6. Real-time prediction status
7. Export results functionality

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create Next.js project
npx create-next-app@latest ml-serverless-app
# Choose: TypeScript, App Router, Tailwind CSS, src/ directory

cd ml-serverless-app

# Install dependencies
npm install swr axios
npm install @vercel/kv @vercel/blob
npm install sharp # For image processing
npm install zod # For validation
npm install recharts # For visualizations

# Install Python dependencies (for serverless functions)
# Create requirements.txt for Python functions
```

Create `requirements.txt`:

```txt
scikit-learn==1.3.0
numpy==1.24.3
joblib==1.3.2
pillow==10.0.0
```

### Step 2: Project Structure

```
src/
├── app/
│   ├── api/
│   │   ├── predict/
│   │   │   └── route.ts
│   │   ├── batch/
│   │   │   └── route.ts
│   │   ├── models/
│   │   │   └── route.ts
│   │   └── health/
│   │       └── route.ts
│   ├── predict/
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── history/
│   │   └── page.tsx
│   └── page.tsx
├── components/
│   ├── PredictionForm.tsx
│   ├── PredictionResults.tsx
│   ├── BatchUpload.tsx
│   └── HistoryTable.tsx
├── lib/
│   ├── ml/
│   │   ├── model.ts
│   │   ├── preprocessor.ts
│   │   └── cache.ts
│   ├── api/
│   │   └── client.ts
│   └── utils/
│       └── validation.ts
└── middleware.ts
```

### Step 3: API Route for ML Predictions

Create `src/app/api/predict/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { kv } from '@vercel/kv';
import { z } from 'zod';
import { headers } from 'next/headers';

// Validation schema
const PredictionSchema = z.object({
  features: z.object({
    age: z.number().min(18).max(100),
    income: z.number().min(0),
    credit_score: z.number().min(300).max(850),
    employment_years: z.number().min(0),
  }),
  modelVersion: z.string().optional().default('v1'),
});

// Types
interface PredictionInput {
  features: {
    age: number;
    income: number;
    credit_score: number;
    employment_years: number;
  };
  modelVersion?: string;
}

interface PredictionResult {
  prediction: string;
  confidence: number;
  probabilities: Record<string, number>;
  modelVersion: string;
  timestamp: string;
}

// In-memory model cache
let modelCache: any = null;

async function loadModel(version: string = 'v1') {
  if (modelCache) {
    return modelCache;
  }

  // In production, load from Vercel Blob or external storage
  // For demo, using mock model
  modelCache = {
    version,
    predict: (features: number[]) => {
      // Mock prediction logic
      const score =
        features[0] * 0.1 +
        features[1] * 0.0001 +
        features[2] * 0.01 +
        features[3] * 0.05;

      const probability = 1 / (1 + Math.exp(-score / 100));

      return {
        prediction: probability > 0.5 ? 'approved' : 'denied',
        probabilities: {
          approved: probability,
          denied: 1 - probability,
        },
        confidence: Math.max(probability, 1 - probability),
      };
    },
  };

  return modelCache;
}

export async function POST(request: NextRequest) {
  const startTime = Date.now();

  try {
    // Parse and validate request body
    const body = await request.json();
    const validatedData = PredictionSchema.parse(body);

    const { features, modelVersion } = validatedData;

    // Generate cache key
    const cacheKey = `prediction:${JSON.stringify(features)}:${modelVersion}`;

    // Check cache
    const cachedResult = await kv.get<PredictionResult>(cacheKey);
    if (cachedResult) {
      return NextResponse.json({
        ...cachedResult,
        cached: true,
        responseTime: Date.now() - startTime,
      });
    }

    // Load model
    const model = await loadModel(modelVersion);

    // Prepare features
    const featureArray = [
      features.age,
      features.income,
      features.credit_score,
      features.employment_years,
    ];

    // Make prediction
    const result = model.predict(featureArray);

    // Prepare response
    const predictionResult: PredictionResult = {
      ...result,
      modelVersion,
      timestamp: new Date().toISOString(),
    };

    // Cache result for 1 hour
    await kv.set(cacheKey, predictionResult, { ex: 3600 });

    // Log prediction (for monitoring)
    await logPrediction(predictionResult, features);

    return NextResponse.json({
      ...predictionResult,
      cached: false,
      responseTime: Date.now() - startTime,
    });
  } catch (error) {
    console.error('Prediction error:', error);

    if (error instanceof z.ZodError) {
      return NextResponse.json(
        {
          error: 'Invalid input',
          details: error.errors,
        },
        { status: 400 }
      );
    }

    return NextResponse.json(
      {
        error: 'Prediction failed',
        message: error instanceof Error ? error.message : 'Unknown error',
      },
      { status: 500 }
    );
  }
}

async function logPrediction(result: PredictionResult, features: any) {
  // Log to analytics/monitoring service
  const logEntry = {
    timestamp: Date.now(),
    prediction: result.prediction,
    confidence: result.confidence,
    features,
    modelVersion: result.modelVersion,
  };

  // Store in KV for history
  await kv.lpush('prediction:history', JSON.stringify(logEntry));
  await kv.ltrim('prediction:history', 0, 999); // Keep last 1000
}

export async function GET() {
  return NextResponse.json({
    status: 'healthy',
    endpoint: 'predict',
    version: '1.0.0',
  });
}
```

### Step 4: Edge Function for Ultra-Fast Predictions

Create `src/app/api/predict-edge/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';

export const runtime = 'edge';

// Lightweight model for edge deployment
class EdgeModel {
  private weights: number[];

  constructor() {
    // Simplified model weights
    this.weights = [0.1, 0.0001, 0.01, 0.05];
  }

  predict(features: number[]): {
    prediction: string;
    confidence: number;
  } {
    const score = features.reduce(
      (sum, feature, idx) => sum + feature * this.weights[idx],
      0
    );

    const probability = 1 / (1 + Math.exp(-score / 100));
    const prediction = probability > 0.5 ? 'approved' : 'denied';
    const confidence = Math.max(probability, 1 - probability);

    return { prediction, confidence };
  }
}

const model = new EdgeModel();

export async function POST(request: NextRequest) {
  const startTime = Date.now();

  try {
    const body = await request.json();
    const { features } = body;

    if (!features || !Array.isArray(features) || features.length !== 4) {
      return NextResponse.json(
        { error: 'Invalid features array' },
        { status: 400 }
      );
    }

    const result = model.predict(features);

    return NextResponse.json({
      ...result,
      edge: true,
      latency: Date.now() - startTime,
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    return NextResponse.json(
      { error: 'Prediction failed' },
      { status: 500 }
    );
  }
}
```

### Step 5: Batch Prediction API Route

Create `src/app/api/batch/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';

const BatchPredictionSchema = z.object({
  predictions: z.array(
    z.object({
      id: z.string(),
      features: z.object({
        age: z.number(),
        income: z.number(),
        credit_score: z.number(),
        employment_years: z.number(),
      }),
    })
  ),
});

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { predictions } = BatchPredictionSchema.parse(body);

    // Process predictions in batches for efficiency
    const batchSize = 10;
    const results = [];

    for (let i = 0; i < predictions.length; i += batchSize) {
      const batch = predictions.slice(i, i + batchSize);

      const batchResults = await Promise.all(
        batch.map(async (item) => {
          const features = [
            item.features.age,
            item.features.income,
            item.features.credit_score,
            item.features.employment_years,
          ];

          const score =
            features[0] * 0.1 +
            features[1] * 0.0001 +
            features[2] * 0.01 +
            features[3] * 0.05;

          const probability = 1 / (1 + Math.exp(-score / 100));

          return {
            id: item.id,
            prediction: probability > 0.5 ? 'approved' : 'denied',
            confidence: Math.max(probability, 1 - probability),
          };
        })
      );

      results.push(...batchResults);
    }

    return NextResponse.json({
      results,
      total: predictions.length,
      timestamp: new Date().toISOString(),
    });
  } catch (error) {
    console.error('Batch prediction error:', error);

    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Invalid input', details: error.errors },
        { status: 400 }
      );
    }

    return NextResponse.json(
      { error: 'Batch prediction failed' },
      { status: 500 }
    );
  }
}
```

### Step 6: Client-side Data Fetching with SWR

Create `src/lib/api/client.ts`:

```typescript
import useSWR from 'swr';
import axios from 'axios';

const apiClient = axios.create({
  baseURL: '/api',
  timeout: 30000,
});

export interface PredictionInput {
  features: {
    age: number;
    income: number;
    credit_score: number;
    employment_years: number;
  };
  modelVersion?: string;
}

export interface PredictionResult {
  prediction: string;
  confidence: number;
  probabilities?: Record<string, number>;
  cached: boolean;
  responseTime: number;
  timestamp: string;
}

export async function predict(
  input: PredictionInput
): Promise<PredictionResult> {
  const response = await apiClient.post('/predict', input);
  return response.data;
}

export async function predictEdge(features: number[]) {
  const response = await apiClient.post('/predict-edge', { features });
  return response.data;
}

export async function batchPredict(predictions: any[]) {
  const response = await apiClient.post('/batch', { predictions });
  return response.data;
}

// SWR hooks
export function usePredictionHistory() {
  const { data, error, isLoading, mutate } = useSWR(
    '/history',
    async (url) => {
      const response = await apiClient.get(url);
      return response.data;
    },
    {
      refreshInterval: 10000, // Refresh every 10 seconds
      revalidateOnFocus: true,
    }
  );

  return {
    history: data || [],
    isLoading,
    error,
    refresh: mutate,
  };
}

export function useModelInfo(modelId: string) {
  const { data, error, isLoading } = useSWR(
    `/models/${modelId}`,
    async (url) => {
      const response = await apiClient.get(url);
      return response.data;
    }
  );

  return {
    modelInfo: data,
    isLoading,
    error,
  };
}
```

### Step 7: Prediction Form Component

Create `src/components/PredictionForm.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { predict } from '@/lib/api/client';

export default function PredictionForm() {
  const [formData, setFormData] = useState({
    age: '',
    income: '',
    credit_score: '',
    employment_years: '',
  });

  const [result, setResult] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      const features = {
        age: parseFloat(formData.age),
        income: parseFloat(formData.income),
        credit_score: parseFloat(formData.credit_score),
        employment_years: parseFloat(formData.employment_years),
      };

      const prediction = await predict({ features });
      setResult(prediction);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Prediction failed');
    } finally {
      setLoading(false);
    }
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value,
    });
  };

  return (
    <div className="max-w-2xl mx-auto p-6">
      <form onSubmit={handleSubmit} className="space-y-6">
        <div>
          <label className="block text-sm font-medium text-gray-700">
            Age
          </label>
          <input
            type="number"
            name="age"
            value={formData.age}
            onChange={handleChange}
            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
            required
            min="18"
            max="100"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700">
            Annual Income ($)
          </label>
          <input
            type="number"
            name="income"
            value={formData.income}
            onChange={handleChange}
            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
            required
            min="0"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700">
            Credit Score
          </label>
          <input
            type="number"
            name="credit_score"
            value={formData.credit_score}
            onChange={handleChange}
            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
            required
            min="300"
            max="850"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700">
            Years of Employment
          </label>
          <input
            type="number"
            name="employment_years"
            value={formData.employment_years}
            onChange={handleChange}
            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
            required
            min="0"
            step="0.1"
          />
        </div>

        <button
          type="submit"
          disabled={loading}
          className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 disabled:opacity-50"
        >
          {loading ? 'Predicting...' : 'Get Prediction'}
        </button>
      </form>

      {error && (
        <div className="mt-4 p-4 bg-red-50 border border-red-200 rounded-md">
          <p className="text-red-700">{error}</p>
        </div>
      )}

      {result && (
        <div className="mt-6 p-6 bg-white border rounded-lg shadow-sm">
          <h3 className="text-lg font-semibold mb-4">Prediction Result</h3>
          <div className="space-y-2">
            <p>
              <span className="font-medium">Decision:</span>{' '}
              <span
                className={`font-bold ${
                  result.prediction === 'approved'
                    ? 'text-green-600'
                    : 'text-red-600'
                }`}
              >
                {result.prediction.toUpperCase()}
              </span>
            </p>
            <p>
              <span className="font-medium">Confidence:</span>{' '}
              {(result.confidence * 100).toFixed(2)}%
            </p>
            <p>
              <span className="font-medium">Response Time:</span>{' '}
              {result.responseTime}ms
            </p>
            <p>
              <span className="font-medium">Cached:</span>{' '}
              {result.cached ? 'Yes' : 'No'}
            </p>
          </div>
        </div>
      )}
    </div>
  );
}
```

### Step 8: Middleware for Rate Limiting

Create `src/middleware.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { kv } from '@vercel/kv';

const RATE_LIMIT_WINDOW = 60; // 1 minute
const MAX_REQUESTS = 20; // Max requests per window

export async function middleware(request: NextRequest) {
  // Only apply to API routes
  if (!request.nextUrl.pathname.startsWith('/api')) {
    return NextResponse.next();
  }

  // Get client identifier (IP address)
  const ip = request.ip || request.headers.get('x-forwarded-for') || 'unknown';
  const key = `rate-limit:${ip}`;

  try {
    // Get current count
    const current = await kv.get<number>(key);

    if (current && current >= MAX_REQUESTS) {
      return NextResponse.json(
        {
          error: 'Rate limit exceeded',
          retryAfter: RATE_LIMIT_WINDOW,
        },
        { status: 429 }
      );
    }

    // Increment counter
    const count = current ? current + 1 : 1;
    await kv.set(key, count, { ex: RATE_LIMIT_WINDOW });

    // Add rate limit headers
    const response = NextResponse.next();
    response.headers.set('X-RateLimit-Limit', MAX_REQUESTS.toString());
    response.headers.set('X-RateLimit-Remaining', (MAX_REQUESTS - count).toString());

    return response;
  } catch (error) {
    // If rate limiting fails, allow the request
    console.error('Rate limiting error:', error);
    return NextResponse.next();
  }
}

export const config = {
  matcher: '/api/:path*',
};
```

### Step 9: Server Component for SSR

Create `src/app/page.tsx`:

```typescript
import { Suspense } from 'react';
import PredictionForm from '@/components/PredictionForm';
import { kv } from '@vercel/kv';

async function getRecentPredictions() {
  try {
    const history = await kv.lrange('prediction:history', 0, 4);
    return history.map((item) => JSON.parse(item as string));
  } catch (error) {
    console.error('Failed to fetch recent predictions:', error);
    return [];
  }
}

export default async function Home() {
  const recentPredictions = await getRecentPredictions();

  return (
    <main className="min-h-screen bg-gray-50 py-12 px-4">
      <div className="max-w-7xl mx-auto">
        <h1 className="text-4xl font-bold text-center mb-2">
          Serverless ML Predictions
        </h1>
        <p className="text-center text-gray-600 mb-12">
          Fast, scalable ML inference with Next.js and Vercel
        </p>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
          <div>
            <h2 className="text-2xl font-semibold mb-4">Make Prediction</h2>
            <Suspense fallback={<div>Loading form...</div>}>
              <PredictionForm />
            </Suspense>
          </div>

          <div>
            <h2 className="text-2xl font-semibold mb-4">Recent Predictions</h2>
            <div className="space-y-4">
              {recentPredictions.map((pred: any, idx: number) => (
                <div
                  key={idx}
                  className="bg-white p-4 rounded-lg shadow-sm border"
                >
                  <div className="flex justify-between items-center">
                    <span className="font-medium">{pred.prediction}</span>
                    <span className="text-sm text-gray-500">
                      {(pred.confidence * 100).toFixed(0)}%
                    </span>
                  </div>
                  <div className="text-xs text-gray-400 mt-2">
                    {new Date(pred.timestamp).toLocaleString()}
                  </div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </main>
  );
}

export const revalidate = 60; // Revalidate every 60 seconds
```

## Expected Outputs

1. **Serverless ML Application**:
   - API routes for predictions
   - Edge functions for low latency
   - Server-side rendering
   - Client-side interactivity

2. **Performance**:
   - Edge latency < 50ms
   - Cached responses instant
   - Optimized bundle sizes
   - ISR for static content

3. **Production Features**:
   - Rate limiting
   - Caching layer
   - Error handling
   - Monitoring hooks

4. **Deployment**:
   - One-click Vercel deployment
   - Automatic scaling
   - Global CDN distribution

## Bonus Challenges

1. **Streaming Responses**: Implement streaming for long predictions
2. **A/B Testing**: Deploy multiple model versions
3. **Analytics**: Add prediction analytics dashboard
4. **WebSocket**: Real-time prediction updates
5. **Authentication**: Add user authentication with NextAuth
6. **Database**: Store predictions in PostgreSQL
7. **Monitoring**: Integrate with Vercel Analytics
8. **Multi-model**: Support switching between models

## Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [Vercel Functions](https://vercel.com/docs/functions)
- [Vercel KV](https://vercel.com/docs/storage/vercel-kv)
- [SWR Documentation](https://swr.vercel.app/)
- [Edge Runtime](https://edge-runtime.vercel.app/)

## Success Criteria

### Functionality (40%)
- [ ] API routes working
- [ ] Edge functions deployed
- [ ] Caching implemented
- [ ] Rate limiting active
- [ ] SSR/ISR configured

### Performance (30%)
- [ ] Edge latency < 50ms
- [ ] Bundle size optimized
- [ ] Image optimization
- [ ] Efficient caching
- [ ] Fast page loads

### Code Quality (20%)
- [ ] TypeScript typing
- [ ] Error handling
- [ ] Validation with Zod
- [ ] Clean architecture
- [ ] Testing coverage

### Deployment (10%)
- [ ] Vercel deployment
- [ ] Environment variables
- [ ] Production-ready
- [ ] Monitoring setup
- [ ] Documentation
