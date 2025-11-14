# Project 01: React App with ML Prediction API Integration

## Overview
Build a comprehensive React application that integrates with a backend ML prediction API. This project demonstrates real-world patterns for connecting frontend applications to machine learning services, handling user input, managing state, and displaying predictions with confidence scores.

## Learning Objectives
- Master React hooks for ML application state management
- Implement robust API integration with error handling
- Create intuitive UI/UX for ML predictions
- Handle loading states and user feedback
- Implement form validation and input preprocessing
- Visualize prediction results with confidence scores
- Apply best practices for async operations in React

## Difficulty Level
**Intermediate** - Requires solid understanding of React, hooks, async operations, and API integration.

## Technical Stack
- **Frontend**: React 18+
- **State Management**: React Hooks (useState, useEffect, useReducer)
- **API Client**: Axios
- **Styling**: Tailwind CSS or Material-UI
- **Visualization**: Chart.js or Recharts
- **Form Handling**: React Hook Form
- **Testing**: Jest, React Testing Library
- **Backend API**: FastAPI or Flask (for ML model serving)

## Requirements

### UI/UX Requirements
1. Clean, intuitive input form with validation
2. Real-time input validation feedback
3. Loading indicators during API calls
4. Error messages with retry capability
5. Result display with confidence visualization
6. Prediction history with timestamps
7. Responsive design for mobile and desktop
8. Accessible components (ARIA labels, keyboard navigation)

### State Management Requirements
1. Input state management
2. Loading/error/success states
3. Prediction results cache
4. Form validation state
5. API call debouncing for real-time predictions

### API Integration Requirements
1. RESTful API client configuration
2. Request/response interceptors
3. Error handling and retry logic
4. Request cancellation for component unmount
5. CORS handling

### Visualization Requirements
1. Confidence score bar charts
2. Prediction probability distribution
3. Historical predictions timeline
4. Feature importance display (if available)

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create React app
npx create-react-app ml-prediction-app
cd ml-prediction-app

# Install dependencies
npm install axios react-hook-form recharts
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Step 2: API Client Setup

Create `src/api/mlApi.js`:

```javascript
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000';

// Create axios instance with default config
const mlApiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor
mlApiClient.interceptors.request.use(
  (config) => {
    // Add auth token if available
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
mlApiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response) {
      // Server responded with error
      console.error('API Error:', error.response.data);
    } else if (error.request) {
      // Request made but no response
      console.error('Network Error:', error.message);
    }
    return Promise.reject(error);
  }
);

// API functions
export const predictAPI = {
  // Single prediction
  predict: async (features) => {
    const response = await mlApiClient.post('/predict', { features });
    return response.data;
  },

  // Batch prediction
  predictBatch: async (featuresArray) => {
    const response = await mlApiClient.post('/predict/batch', {
      features: featuresArray,
    });
    return response.data;
  },

  // Get model info
  getModelInfo: async () => {
    const response = await mlApiClient.get('/model/info');
    return response.data;
  },

  // Health check
  healthCheck: async () => {
    const response = await mlApiClient.get('/health');
    return response.data;
  },
};

export default mlApiClient;
```

### Step 3: Custom Hook for Predictions

Create `src/hooks/usePrediction.js`:

```javascript
import { useState, useCallback, useRef } from 'react';
import { predictAPI } from '../api/mlApi';

export const usePrediction = () => {
  const [prediction, setPrediction] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [history, setHistory] = useState([]);
  const abortControllerRef = useRef(null);

  const predict = useCallback(async (features) => {
    // Cancel previous request if still pending
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    // Create new abort controller
    abortControllerRef.current = new AbortController();

    setLoading(true);
    setError(null);

    try {
      const result = await predictAPI.predict(features);

      const predictionData = {
        ...result,
        timestamp: new Date().toISOString(),
        features,
      };

      setPrediction(predictionData);

      // Add to history
      setHistory((prev) => [predictionData, ...prev].slice(0, 10));

      return predictionData;
    } catch (err) {
      if (err.name === 'AbortError') {
        console.log('Request cancelled');
        return null;
      }

      const errorMessage =
        err.response?.data?.message || err.message || 'Prediction failed';
      setError(errorMessage);
      throw err;
    } finally {
      setLoading(false);
    }
  }, []);

  const clearPrediction = useCallback(() => {
    setPrediction(null);
    setError(null);
  }, []);

  const clearHistory = useCallback(() => {
    setHistory([]);
  }, []);

  return {
    prediction,
    loading,
    error,
    history,
    predict,
    clearPrediction,
    clearHistory,
  };
};
```

### Step 4: Input Form Component

Create `src/components/PredictionForm.jsx`:

```javascript
import React from 'react';
import { useForm } from 'react-hook-form';

const PredictionForm = ({ onSubmit, loading }) => {
  const {
    register,
    handleSubmit,
    formState: { errors },
    reset,
  } = useForm({
    defaultValues: {
      age: '',
      income: '',
      creditScore: '',
      employmentYears: '',
    },
  });

  const onFormSubmit = (data) => {
    // Convert string values to numbers
    const features = {
      age: parseFloat(data.age),
      income: parseFloat(data.income),
      credit_score: parseFloat(data.creditScore),
      employment_years: parseFloat(data.employmentYears),
    };

    onSubmit(features);
  };

  return (
    <form
      onSubmit={handleSubmit(onFormSubmit)}
      className="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4"
    >
      <h2 className="text-2xl font-bold mb-6">Enter Customer Information</h2>

      {/* Age Field */}
      <div className="mb-4">
        <label
          className="block text-gray-700 text-sm font-bold mb-2"
          htmlFor="age"
        >
          Age
        </label>
        <input
          {...register('age', {
            required: 'Age is required',
            min: { value: 18, message: 'Must be at least 18' },
            max: { value: 100, message: 'Must be at most 100' },
          })}
          className={`shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline ${
            errors.age ? 'border-red-500' : ''
          }`}
          type="number"
          placeholder="25"
        />
        {errors.age && (
          <p className="text-red-500 text-xs italic">{errors.age.message}</p>
        )}
      </div>

      {/* Income Field */}
      <div className="mb-4">
        <label
          className="block text-gray-700 text-sm font-bold mb-2"
          htmlFor="income"
        >
          Annual Income ($)
        </label>
        <input
          {...register('income', {
            required: 'Income is required',
            min: { value: 0, message: 'Must be positive' },
          })}
          className={`shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline ${
            errors.income ? 'border-red-500' : ''
          }`}
          type="number"
          placeholder="50000"
        />
        {errors.income && (
          <p className="text-red-500 text-xs italic">{errors.income.message}</p>
        )}
      </div>

      {/* Credit Score Field */}
      <div className="mb-4">
        <label
          className="block text-gray-700 text-sm font-bold mb-2"
          htmlFor="creditScore"
        >
          Credit Score
        </label>
        <input
          {...register('creditScore', {
            required: 'Credit score is required',
            min: { value: 300, message: 'Minimum credit score is 300' },
            max: { value: 850, message: 'Maximum credit score is 850' },
          })}
          className={`shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline ${
            errors.creditScore ? 'border-red-500' : ''
          }`}
          type="number"
          placeholder="700"
        />
        {errors.creditScore && (
          <p className="text-red-500 text-xs italic">
            {errors.creditScore.message}
          </p>
        )}
      </div>

      {/* Employment Years Field */}
      <div className="mb-6">
        <label
          className="block text-gray-700 text-sm font-bold mb-2"
          htmlFor="employmentYears"
        >
          Years of Employment
        </label>
        <input
          {...register('employmentYears', {
            required: 'Employment years is required',
            min: { value: 0, message: 'Must be positive' },
          })}
          className={`shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline ${
            errors.employmentYears ? 'border-red-500' : ''
          }`}
          type="number"
          step="0.1"
          placeholder="3"
        />
        {errors.employmentYears && (
          <p className="text-red-500 text-xs italic">
            {errors.employmentYears.message}
          </p>
        )}
      </div>

      {/* Submit Buttons */}
      <div className="flex items-center justify-between">
        <button
          className="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline disabled:opacity-50 disabled:cursor-not-allowed"
          type="submit"
          disabled={loading}
        >
          {loading ? (
            <span className="flex items-center">
              <svg
                className="animate-spin -ml-1 mr-3 h-5 w-5 text-white"
                xmlns="http://www.w3.org/2000/svg"
                fill="none"
                viewBox="0 0 24 24"
              >
                <circle
                  className="opacity-25"
                  cx="12"
                  cy="12"
                  r="10"
                  stroke="currentColor"
                  strokeWidth="4"
                ></circle>
                <path
                  className="opacity-75"
                  fill="currentColor"
                  d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                ></path>
              </svg>
              Predicting...
            </span>
          ) : (
            'Get Prediction'
          )}
        </button>
        <button
          className="bg-gray-500 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
          type="button"
          onClick={() => reset()}
        >
          Clear
        </button>
      </div>
    </form>
  );
};

export default PredictionForm;
```

### Step 5: Results Display Component

Create `src/components/PredictionResults.jsx`:

```javascript
import React from 'react';
import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
  Cell,
} from 'recharts';

const PredictionResults = ({ prediction, error }) => {
  if (error) {
    return (
      <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative">
        <strong className="font-bold">Error: </strong>
        <span className="block sm:inline">{error}</span>
      </div>
    );
  }

  if (!prediction) {
    return null;
  }

  const {
    prediction: result,
    confidence,
    probability,
    probabilities,
    timestamp,
  } = prediction;

  // Prepare data for confidence chart
  const confidenceData = probabilities
    ? Object.entries(probabilities).map(([label, prob]) => ({
        label,
        probability: (prob * 100).toFixed(2),
      }))
    : [{ label: 'Confidence', probability: (confidence * 100).toFixed(2) }];

  // Determine color based on prediction
  const resultColor = result === 'approved' ? 'green' : 'red';

  return (
    <div className="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
      <h2 className="text-2xl font-bold mb-6">Prediction Results</h2>

      {/* Main Result */}
      <div className="mb-6">
        <div
          className={`bg-${resultColor}-100 border-l-4 border-${resultColor}-500 text-${resultColor}-700 p-4`}
        >
          <p className="font-bold text-xl">
            Prediction: {result.toUpperCase()}
          </p>
          <p className="text-sm">
            Confidence: {(confidence * 100).toFixed(2)}%
          </p>
          <p className="text-xs text-gray-600 mt-2">
            Generated at: {new Date(timestamp).toLocaleString()}
          </p>
        </div>
      </div>

      {/* Confidence Visualization */}
      <div className="mb-6">
        <h3 className="text-lg font-semibold mb-3">Confidence Breakdown</h3>
        <ResponsiveContainer width="100%" height={300}>
          <BarChart data={confidenceData}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="label" />
            <YAxis domain={[0, 100]} />
            <Tooltip formatter={(value) => `${value}%`} />
            <Legend />
            <Bar dataKey="probability" name="Probability (%)">
              {confidenceData.map((entry, index) => (
                <Cell
                  key={`cell-${index}`}
                  fill={entry.label === result ? '#10b981' : '#ef4444'}
                />
              ))}
            </Bar>
          </BarChart>
        </ResponsiveContainer>
      </div>

      {/* Probability Details */}
      {probabilities && (
        <div className="bg-gray-50 p-4 rounded">
          <h3 className="text-lg font-semibold mb-3">Class Probabilities</h3>
          <div className="grid grid-cols-2 gap-4">
            {Object.entries(probabilities).map(([label, prob]) => (
              <div key={label} className="flex justify-between items-center">
                <span className="font-medium">{label}:</span>
                <span className="text-gray-600">{(prob * 100).toFixed(2)}%</span>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
};

export default PredictionResults;
```

### Step 6: Main App Component

Update `src/App.js`:

```javascript
import React, { useEffect } from 'react';
import PredictionForm from './components/PredictionForm';
import PredictionResults from './components/PredictionResults';
import { usePrediction } from './hooks/usePrediction';
import './App.css';

function App() {
  const {
    prediction,
    loading,
    error,
    history,
    predict,
    clearPrediction,
    clearHistory,
  } = usePrediction();

  const handlePredict = async (features) => {
    try {
      await predict(features);
    } catch (err) {
      console.error('Prediction error:', err);
    }
  };

  return (
    <div className="min-h-screen bg-gray-100 py-6 px-4 sm:px-6 lg:px-8">
      <div className="max-w-7xl mx-auto">
        {/* Header */}
        <div className="text-center mb-8">
          <h1 className="text-4xl font-bold text-gray-900 mb-2">
            ML Prediction Dashboard
          </h1>
          <p className="text-gray-600">
            Credit Approval Prediction System
          </p>
        </div>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
          {/* Left Column: Form */}
          <div>
            <PredictionForm onSubmit={handlePredict} loading={loading} />
          </div>

          {/* Right Column: Results */}
          <div>
            <PredictionResults prediction={prediction} error={error} />
          </div>
        </div>

        {/* History Section */}
        {history.length > 0 && (
          <div className="mt-8">
            <div className="bg-white shadow-md rounded px-8 pt-6 pb-8">
              <div className="flex justify-between items-center mb-4">
                <h2 className="text-2xl font-bold">Prediction History</h2>
                <button
                  onClick={clearHistory}
                  className="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded text-sm"
                >
                  Clear History
                </button>
              </div>
              <div className="overflow-x-auto">
                <table className="min-w-full">
                  <thead className="bg-gray-50">
                    <tr>
                      <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Timestamp
                      </th>
                      <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Prediction
                      </th>
                      <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Confidence
                      </th>
                      <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                        Features
                      </th>
                    </tr>
                  </thead>
                  <tbody className="bg-white divide-y divide-gray-200">
                    {history.map((item, index) => (
                      <tr key={index}>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {new Date(item.timestamp).toLocaleString()}
                        </td>
                        <td className="px-6 py-4 whitespace-nowrap">
                          <span
                            className={`px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${
                              item.prediction === 'approved'
                                ? 'bg-green-100 text-green-800'
                                : 'bg-red-100 text-red-800'
                            }`}
                          >
                            {item.prediction}
                          </span>
                        </td>
                        <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {(item.confidence * 100).toFixed(2)}%
                        </td>
                        <td className="px-6 py-4 text-sm text-gray-500">
                          Age: {item.features.age}, Income: $
                          {item.features.income}
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

export default App;
```

### Step 7: Backend API Example (FastAPI)

Create `backend/main.py`:

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from typing import Dict, List
import joblib

app = FastAPI(title="ML Prediction API")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load model (mock for demonstration)
# model = joblib.load('model.pkl')

class PredictionInput(BaseModel):
    features: Dict[str, float]

class PredictionOutput(BaseModel):
    prediction: str
    confidence: float
    probabilities: Dict[str, float]

@app.post("/predict", response_model=PredictionOutput)
async def predict(input_data: PredictionInput):
    try:
        features = input_data.features

        # Extract features in correct order
        X = np.array([[
            features['age'],
            features['income'],
            features['credit_score'],
            features['employment_years']
        ]])

        # Mock prediction (replace with actual model.predict)
        # prediction = model.predict(X)[0]
        # probabilities = model.predict_proba(X)[0]

        # Mock logic for demonstration
        credit_score = features['credit_score']
        income = features['income']

        if credit_score > 650 and income > 40000:
            prediction = "approved"
            prob_approved = 0.85
        else:
            prediction = "denied"
            prob_approved = 0.25

        return PredictionOutput(
            prediction=prediction,
            confidence=prob_approved if prediction == "approved" else 1 - prob_approved,
            probabilities={
                "approved": prob_approved,
                "denied": 1 - prob_approved
            }
        )

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

@app.get("/model/info")
async def model_info():
    return {
        "model_name": "Credit Approval Classifier",
        "version": "1.0.0",
        "features": ["age", "income", "credit_score", "employment_years"]
    }
```

## Expected Outputs

1. **Functional Web Application**:
   - Clean, responsive UI with form inputs
   - Real-time validation feedback
   - Loading states during predictions
   - Results displayed with visualizations

2. **API Integration**:
   - Successful API calls to backend
   - Error handling with user feedback
   - Request cancellation on component unmount

3. **Visualizations**:
   - Confidence score bar charts
   - Probability distributions
   - Prediction history table

4. **User Experience**:
   - Smooth transitions and loading states
   - Clear error messages
   - Accessible form controls
   - Mobile-responsive design

## Bonus Challenges

1. **Add Real-time Predictions**:
   - Implement debounced real-time predictions as user types
   - Show confidence scores updating in real-time

2. **Feature Importance**:
   - Request and display feature importance from backend
   - Visualize which features contributed most to prediction

3. **Export Functionality**:
   - Add ability to export predictions as CSV/JSON
   - Generate PDF reports of predictions

4. **A/B Testing**:
   - Integrate multiple models
   - Compare predictions from different models side-by-side

5. **Advanced State Management**:
   - Implement Redux or Zustand for complex state
   - Add middleware for logging and analytics

6. **Offline Support**:
   - Implement service workers for PWA
   - Cache predictions for offline viewing

7. **Authentication**:
   - Add user login/registration
   - Implement JWT token management
   - User-specific prediction history

8. **WebSocket Integration**:
   - Real-time updates for long-running predictions
   - Live model retraining notifications

## Resources

### Documentation
- [React Hooks](https://react.dev/reference/react)
- [React Hook Form](https://react-hook-form.com/)
- [Axios](https://axios-http.com/docs/intro)
- [Recharts](https://recharts.org/en-US/)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [FastAPI](https://fastapi.tiangolo.com/)

### Tutorials
- [React Query for API State](https://tanstack.com/query/latest)
- [Error Boundaries in React](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [Async Operations Best Practices](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous)

### Tools
- [Postman](https://www.postman.com/) - API testing
- [React DevTools](https://react.dev/learn/react-developer-tools)
- [Lighthouse](https://developers.google.com/web/tools/lighthouse) - Performance auditing

## Success Criteria

### Functionality (40%)
- [ ] Form accepts valid inputs with validation
- [ ] API calls successfully connect to backend
- [ ] Predictions display correctly with confidence scores
- [ ] Error handling works for all failure scenarios
- [ ] Prediction history persists during session

### Code Quality (30%)
- [ ] Clean component structure with separation of concerns
- [ ] Custom hooks for reusable logic
- [ ] Proper error handling and loading states
- [ ] Type safety (PropTypes or TypeScript)
- [ ] No console errors or warnings

### User Experience (20%)
- [ ] Intuitive, accessible interface
- [ ] Responsive design works on mobile and desktop
- [ ] Loading indicators provide clear feedback
- [ ] Error messages are user-friendly
- [ ] Smooth transitions and animations

### Best Practices (10%)
- [ ] Code follows React best practices
- [ ] API client properly configured with interceptors
- [ ] Component cleanup (abort controllers, cleanup functions)
- [ ] Proper form validation and sanitization
- [ ] Accessible HTML with ARIA labels

### Bonus Points
- Unit tests with >80% coverage
- Integration tests for API calls
- End-to-end tests with Cypress
- Performance optimization (memoization, code splitting)
- Documentation and comments
