# Project 03: Angular App with Real-Time Predictions

## Overview
Build an enterprise-grade Angular application with real-time ML predictions using RxJS reactive programming. This project demonstrates Angular's powerful dependency injection, reactive forms, state management with NgRx, and real-time data streaming for instant ML predictions as users interact with the application.

## Learning Objectives
- Master Angular reactive forms and validation
- Implement NgRx for state management in ML applications
- Use RxJS operators for real-time data processing
- Create Angular services for ML API integration
- Implement debounced real-time predictions
- Build reusable Angular components and pipes
- Handle async operations with observables
- Apply Angular best practices and design patterns

## Difficulty Level
**Advanced** - Requires strong understanding of Angular, RxJS, NgRx, and reactive programming patterns.

## Technical Stack
- **Frontend**: Angular 16+
- **State Management**: NgRx (Store, Effects, Selectors)
- **Reactive Programming**: RxJS
- **Forms**: Angular Reactive Forms
- **HTTP**: Angular HttpClient with Interceptors
- **UI Framework**: Angular Material
- **Real-time**: Server-Sent Events (SSE) / WebSocket
- **Charts**: ngx-charts
- **Testing**: Jasmine, Karma, Cypress

## Requirements

### UI/UX Requirements
1. Material Design interface with Angular Material
2. Real-time prediction updates as user types
3. Visual feedback with loading states
4. Form validation with error messages
5. Prediction confidence visualization
6. Historical predictions timeline
7. Keyboard shortcuts for power users
8. Accessible components following WCAG 2.1

### State Management Requirements
1. NgRx store for application state
2. Effects for side effects (API calls)
3. Selectors for derived state
4. Action patterns for state mutations
5. State persistence to localStorage
6. Time-travel debugging support

### Real-Time Features
1. Debounced input for API efficiency
2. Streaming predictions via SSE/WebSocket
3. Optimistic UI updates
4. Request cancellation on new input
5. Exponential backoff for retries
6. Connection status indicators

### Performance Requirements
1. OnPush change detection strategy
2. Lazy loading for feature modules
3. Virtual scrolling for large lists
4. Memoized selectors
5. Bundle size optimization

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Install Angular CLI
npm install -g @angular/cli

# Create new Angular project
ng new ml-realtime-predictions --routing --style=scss

cd ml-realtime-predictions

# Install dependencies
ng add @ngrx/store @ngrx/effects @ngrx/store-devtools
ng add @angular/material
npm install rxjs lodash-es
npm install @swimlane/ngx-charts
```

### Step 2: Generate Core Structure

```bash
# Generate feature module
ng generate module features/prediction --route prediction --module app.module

# Generate services
ng generate service core/services/prediction-api
ng generate service core/services/realtime

# Generate NgRx store
ng generate store features/prediction/store/prediction --module features/prediction/prediction.module

# Generate components
ng generate component features/prediction/components/prediction-form
ng generate component features/prediction/components/prediction-results
ng generate component features/prediction/components/confidence-chart
ng generate component features/prediction/components/prediction-history
```

### Step 3: API Service with HttpClient

Create `src/app/core/services/prediction-api.service.ts`:

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry, timeout } from 'rxjs/operators';
import { environment } from '../../../environments/environment';

export interface PredictionInput {
  features: Record<string, any>;
}

export interface PredictionResult {
  prediction: string;
  confidence: number;
  probabilities: Record<string, number>;
  timestamp: string;
  modelVersion: string;
}

@Injectable({
  providedIn: 'root'
})
export class PredictionApiService {
  private readonly apiUrl = environment.apiUrl;

  constructor(private http: HttpClient) {}

  predict(input: PredictionInput): Observable<PredictionResult> {
    return this.http
      .post<PredictionResult>(`${this.apiUrl}/predict`, input)
      .pipe(
        timeout(30000),
        retry({
          count: 2,
          delay: 1000,
        }),
        catchError(this.handleError)
      );
  }

  batchPredict(inputs: PredictionInput[]): Observable<PredictionResult[]> {
    return this.http
      .post<PredictionResult[]>(`${this.apiUrl}/predict/batch`, {
        inputs,
      })
      .pipe(
        timeout(60000),
        catchError(this.handleError)
      );
  }

  getModelInfo(): Observable<any> {
    return this.http
      .get(`${this.apiUrl}/model/info`)
      .pipe(catchError(this.handleError));
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    let errorMessage = 'An unknown error occurred';

    if (error.error instanceof ErrorEvent) {
      // Client-side error
      errorMessage = `Error: ${error.error.message}`;
    } else {
      // Server-side error
      errorMessage = `Server Error: ${error.status} - ${error.message}`;
      if (error.error?.message) {
        errorMessage = error.error.message;
      }
    }

    console.error('API Error:', errorMessage);
    return throwError(() => new Error(errorMessage));
  }
}
```

### Step 4: Real-Time Service with SSE

Create `src/app/core/services/realtime.service.ts`:

```typescript
import { Injectable, NgZone } from '@angular/core';
import { Observable, Subject, fromEvent } from 'rxjs';
import { map, takeUntil } from 'rxjs/operators';
import { environment } from '../../../environments/environment';

export interface StreamingPrediction {
  partialResult: any;
  complete: boolean;
  progress: number;
}

@Injectable({
  providedIn: 'root'
})
export class RealtimeService {
  private readonly sseUrl = environment.sseUrl;
  private eventSource?: EventSource;
  private destroy$ = new Subject<void>();

  constructor(private ngZone: NgZone) {}

  connectToPredictionStream(sessionId: string): Observable<StreamingPrediction> {
    return new Observable(observer => {
      this.ngZone.runOutsideAngular(() => {
        this.eventSource = new EventSource(
          `${this.sseUrl}/stream/${sessionId}`
        );

        this.eventSource.onmessage = (event) => {
          this.ngZone.run(() => {
            const data = JSON.parse(event.data);
            observer.next(data);
          });
        };

        this.eventSource.onerror = (error) => {
          this.ngZone.run(() => {
            console.error('SSE Error:', error);
            observer.error(error);
            this.disconnect();
          });
        };
      });

      return () => {
        this.disconnect();
      };
    }).pipe(takeUntil(this.destroy$));
  }

  disconnect(): void {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = undefined;
    }
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.disconnect();
  }
}
```

### Step 5: NgRx Store Setup

Create `src/app/features/prediction/store/prediction.state.ts`:

```typescript
import { PredictionResult } from '../../../core/services/prediction-api.service';

export interface PredictionState {
  currentPrediction: PredictionResult | null;
  history: PredictionResult[];
  loading: boolean;
  error: string | null;
  formValues: Record<string, any>;
  streamingProgress: number;
  isStreaming: boolean;
}

export const initialState: PredictionState = {
  currentPrediction: null,
  history: [],
  loading: false,
  error: null,
  formValues: {},
  streamingProgress: 0,
  isStreaming: false,
};
```

Create `src/app/features/prediction/store/prediction.actions.ts`:

```typescript
import { createAction, props } from '@ngrx/store';
import { PredictionInput, PredictionResult } from '../../../core/services/prediction-api.service';

// Prediction Actions
export const submitPrediction = createAction(
  '[Prediction] Submit Prediction',
  props<{ input: PredictionInput }>()
);

export const submitPredictionSuccess = createAction(
  '[Prediction] Submit Prediction Success',
  props<{ result: PredictionResult }>()
);

export const submitPredictionFailure = createAction(
  '[Prediction] Submit Prediction Failure',
  props<{ error: string }>()
);

// Real-time Actions
export const startRealtimePrediction = createAction(
  '[Prediction] Start Realtime Prediction',
  props<{ input: PredictionInput }>()
);

export const updateRealtimePrediction = createAction(
  '[Prediction] Update Realtime Prediction',
  props<{ partialResult: any; progress: number }>()
);

export const completeRealtimePrediction = createAction(
  '[Prediction] Complete Realtime Prediction',
  props<{ result: PredictionResult }>()
);

// History Actions
export const loadHistory = createAction('[Prediction] Load History');

export const loadHistorySuccess = createAction(
  '[Prediction] Load History Success',
  props<{ history: PredictionResult[] }>()
);

export const clearHistory = createAction('[Prediction] Clear History');

// Form Actions
export const updateFormValues = createAction(
  '[Prediction] Update Form Values',
  props<{ values: Record<string, any> }>()
);

export const resetForm = createAction('[Prediction] Reset Form');
```

Create `src/app/features/prediction/store/prediction.reducer.ts`:

```typescript
import { createReducer, on } from '@ngrx/store';
import { initialState } from './prediction.state';
import * as PredictionActions from './prediction.actions';

export const predictionReducer = createReducer(
  initialState,

  // Submit Prediction
  on(PredictionActions.submitPrediction, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),

  on(PredictionActions.submitPredictionSuccess, (state, { result }) => ({
    ...state,
    loading: false,
    currentPrediction: result,
    history: [result, ...state.history].slice(0, 50),
    error: null,
  })),

  on(PredictionActions.submitPredictionFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),

  // Realtime Prediction
  on(PredictionActions.startRealtimePrediction, (state) => ({
    ...state,
    isStreaming: true,
    streamingProgress: 0,
    error: null,
  })),

  on(PredictionActions.updateRealtimePrediction, (state, { partialResult, progress }) => ({
    ...state,
    streamingProgress: progress,
    currentPrediction: partialResult,
  })),

  on(PredictionActions.completeRealtimePrediction, (state, { result }) => ({
    ...state,
    isStreaming: false,
    streamingProgress: 100,
    currentPrediction: result,
    history: [result, ...state.history].slice(0, 50),
  })),

  // History
  on(PredictionActions.loadHistorySuccess, (state, { history }) => ({
    ...state,
    history,
  })),

  on(PredictionActions.clearHistory, (state) => ({
    ...state,
    history: [],
  })),

  // Form
  on(PredictionActions.updateFormValues, (state, { values }) => ({
    ...state,
    formValues: values,
  })),

  on(PredictionActions.resetForm, (state) => ({
    ...state,
    formValues: {},
    currentPrediction: null,
    error: null,
  }))
);
```

Create `src/app/features/prediction/store/prediction.effects.ts`:

```typescript
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { of } from 'rxjs';
import {
  map,
  catchError,
  switchMap,
  tap,
  debounceTime,
  distinctUntilChanged,
} from 'rxjs/operators';
import { PredictionApiService } from '../../../core/services/prediction-api.service';
import { RealtimeService } from '../../../core/services/realtime.service';
import * as PredictionActions from './prediction.actions';

@Injectable()
export class PredictionEffects {
  submitPrediction$ = createEffect(() =>
    this.actions$.pipe(
      ofType(PredictionActions.submitPrediction),
      debounceTime(300), // Debounce to avoid too many API calls
      distinctUntilChanged(),
      switchMap(({ input }) =>
        this.predictionApi.predict(input).pipe(
          map((result) =>
            PredictionActions.submitPredictionSuccess({ result })
          ),
          catchError((error) =>
            of(
              PredictionActions.submitPredictionFailure({
                error: error.message,
              })
            )
          )
        )
      )
    )
  );

  startRealtimePrediction$ = createEffect(() =>
    this.actions$.pipe(
      ofType(PredictionActions.startRealtimePrediction),
      switchMap(({ input }) => {
        // Start streaming prediction
        const sessionId = this.generateSessionId();

        return this.realtimeService.connectToPredictionStream(sessionId).pipe(
          map((streamData) => {
            if (streamData.complete) {
              return PredictionActions.completeRealtimePrediction({
                result: streamData.partialResult,
              });
            } else {
              return PredictionActions.updateRealtimePrediction({
                partialResult: streamData.partialResult,
                progress: streamData.progress,
              });
            }
          }),
          catchError((error) =>
            of(
              PredictionActions.submitPredictionFailure({
                error: error.message,
              })
            )
          )
        );
      })
    )
  );

  loadHistory$ = createEffect(() =>
    this.actions$.pipe(
      ofType(PredictionActions.loadHistory),
      switchMap(() => {
        // Load from localStorage
        const history = this.loadHistoryFromStorage();
        return of(PredictionActions.loadHistorySuccess({ history }));
      })
    )
  );

  saveHistory$ = createEffect(
    () =>
      this.actions$.pipe(
        ofType(
          PredictionActions.submitPredictionSuccess,
          PredictionActions.completeRealtimePrediction
        ),
        tap(({ result }) => {
          this.saveToLocalStorage(result);
        })
      ),
    { dispatch: false }
  );

  constructor(
    private actions$: Actions,
    private predictionApi: PredictionApiService,
    private realtimeService: RealtimeService
  ) {}

  private generateSessionId(): string {
    return `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private loadHistoryFromStorage(): any[] {
    try {
      const stored = localStorage.getItem('prediction_history');
      return stored ? JSON.parse(stored) : [];
    } catch {
      return [];
    }
  }

  private saveToLocalStorage(result: any): void {
    try {
      const history = this.loadHistoryFromStorage();
      const updated = [result, ...history].slice(0, 50);
      localStorage.setItem('prediction_history', JSON.stringify(updated));
    } catch (error) {
      console.error('Failed to save to localStorage:', error);
    }
  }
}
```

Create `src/app/features/prediction/store/prediction.selectors.ts`:

```typescript
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { PredictionState } from './prediction.state';

export const selectPredictionState =
  createFeatureSelector<PredictionState>('prediction');

export const selectCurrentPrediction = createSelector(
  selectPredictionState,
  (state) => state.currentPrediction
);

export const selectPredictionHistory = createSelector(
  selectPredictionState,
  (state) => state.history
);

export const selectIsLoading = createSelector(
  selectPredictionState,
  (state) => state.loading || state.isStreaming
);

export const selectError = createSelector(
  selectPredictionState,
  (state) => state.error
);

export const selectStreamingProgress = createSelector(
  selectPredictionState,
  (state) => state.streamingProgress
);

export const selectFormValues = createSelector(
  selectPredictionState,
  (state) => state.formValues
);

// Memoized computed selectors
export const selectPredictionWithConfidence = createSelector(
  selectCurrentPrediction,
  (prediction) => {
    if (!prediction) return null;

    return {
      ...prediction,
      confidenceLevel:
        prediction.confidence > 0.8
          ? 'high'
          : prediction.confidence > 0.6
          ? 'medium'
          : 'low',
    };
  }
);

export const selectRecentHistory = createSelector(
  selectPredictionHistory,
  (history) => history.slice(0, 10)
);
```

### Step 6: Prediction Form Component

Create `src/app/features/prediction/components/prediction-form/prediction-form.component.ts`:

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import {
  FormBuilder,
  FormGroup,
  Validators,
  AbstractControl,
} from '@angular/forms';
import { Store } from '@ngrx/store';
import { Subject } from 'rxjs';
import { debounceTime, distinctUntilChanged, takeUntil } from 'rxjs/operators';
import * as PredictionActions from '../../store/prediction.actions';
import { selectIsLoading, selectError } from '../../store/prediction.selectors';

@Component({
  selector: 'app-prediction-form',
  templateUrl: './prediction-form.component.html',
  styleUrls: ['./prediction-form.component.scss'],
})
export class PredictionFormComponent implements OnInit, OnDestroy {
  predictionForm!: FormGroup;
  isLoading$ = this.store.select(selectIsLoading);
  error$ = this.store.select(selectError);
  private destroy$ = new Subject<void>();
  enableRealtime = true;

  constructor(
    private fb: FormBuilder,
    private store: Store
  ) {}

  ngOnInit(): void {
    this.initializeForm();
    this.setupRealtimePrediction();
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private initializeForm(): void {
    this.predictionForm = this.fb.group({
      age: [
        '',
        [
          Validators.required,
          Validators.min(18),
          Validators.max(100),
          this.numberValidator,
        ],
      ],
      income: [
        '',
        [Validators.required, Validators.min(0), this.numberValidator],
      ],
      creditScore: [
        '',
        [
          Validators.required,
          Validators.min(300),
          Validators.max(850),
          this.numberValidator,
        ],
      ],
      employmentYears: [
        '',
        [Validators.required, Validators.min(0), this.numberValidator],
      ],
      loanAmount: [
        '',
        [Validators.required, Validators.min(1000), this.numberValidator],
      ],
    });
  }

  private setupRealtimePrediction(): void {
    this.predictionForm.valueChanges
      .pipe(
        debounceTime(500),
        distinctUntilChanged(),
        takeUntil(this.destroy$)
      )
      .subscribe((values) => {
        if (this.enableRealtime && this.predictionForm.valid) {
          this.submitRealtimePrediction();
        }
      });
  }

  private numberValidator(control: AbstractControl): { [key: string]: any } | null {
    const value = control.value;
    if (value === '' || value === null) return null;
    return !isNaN(parseFloat(value)) && isFinite(value) ? null : { notANumber: true };
  }

  onSubmit(): void {
    if (this.predictionForm.valid) {
      const input = {
        features: this.predictionForm.value,
      };

      this.store.dispatch(PredictionActions.submitPrediction({ input }));
    } else {
      this.predictionForm.markAllAsTouched();
    }
  }

  submitRealtimePrediction(): void {
    const input = {
      features: this.predictionForm.value,
    };

    this.store.dispatch(PredictionActions.startRealtimePrediction({ input }));
  }

  onReset(): void {
    this.predictionForm.reset();
    this.store.dispatch(PredictionActions.resetForm());
  }

  toggleRealtime(): void {
    this.enableRealtime = !this.enableRealtime;
  }

  getErrorMessage(controlName: string): string {
    const control = this.predictionForm.get(controlName);
    if (!control || !control.errors) return '';

    if (control.errors['required']) return 'This field is required';
    if (control.errors['min']) return `Minimum value is ${control.errors['min'].min}`;
    if (control.errors['max']) return `Maximum value is ${control.errors['max'].max}`;
    if (control.errors['notANumber']) return 'Please enter a valid number';

    return 'Invalid input';
  }

  hasError(controlName: string): boolean {
    const control = this.predictionForm.get(controlName);
    return !!(control && control.invalid && (control.dirty || control.touched));
  }
}
```

Create `src/app/features/prediction/components/prediction-form/prediction-form.component.html`:

```html
<mat-card class="prediction-form-card">
  <mat-card-header>
    <mat-card-title>Loan Prediction Form</mat-card-title>
    <mat-card-subtitle>
      Enter customer information for real-time loan approval prediction
    </mat-card-subtitle>
  </mat-card-header>

  <mat-card-content>
    <form [formGroup]="predictionForm" (ngSubmit)="onSubmit()">
      <!-- Age Field -->
      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Age</mat-label>
        <input
          matInput
          type="number"
          formControlName="age"
          placeholder="25"
        />
        <mat-icon matSuffix>person</mat-icon>
        <mat-error *ngIf="hasError('age')">
          {{ getErrorMessage('age') }}
        </mat-error>
      </mat-form-field>

      <!-- Income Field -->
      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Annual Income</mat-label>
        <input
          matInput
          type="number"
          formControlName="income"
          placeholder="50000"
        />
        <span matPrefix>$&nbsp;</span>
        <mat-icon matSuffix>attach_money</mat-icon>
        <mat-error *ngIf="hasError('income')">
          {{ getErrorMessage('income') }}
        </mat-error>
      </mat-form-field>

      <!-- Credit Score Field -->
      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Credit Score</mat-label>
        <input
          matInput
          type="number"
          formControlName="creditScore"
          placeholder="700"
        />
        <mat-icon matSuffix>score</mat-icon>
        <mat-error *ngIf="hasError('creditScore')">
          {{ getErrorMessage('creditScore') }}
        </mat-error>
      </mat-form-field>

      <!-- Employment Years Field -->
      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Years of Employment</mat-label>
        <input
          matInput
          type="number"
          formControlName="employmentYears"
          placeholder="3"
          step="0.1"
        />
        <mat-icon matSuffix>work</mat-icon>
        <mat-error *ngIf="hasError('employmentYears')">
          {{ getErrorMessage('employmentYears') }}
        </mat-error>
      </mat-form-field>

      <!-- Loan Amount Field -->
      <mat-form-field appearance="outline" class="full-width">
        <mat-label>Loan Amount</mat-label>
        <input
          matInput
          type="number"
          formControlName="loanAmount"
          placeholder="10000"
        />
        <span matPrefix>$&nbsp;</span>
        <mat-icon matSuffix>account_balance</mat-icon>
        <mat-error *ngIf="hasError('loanAmount')">
          {{ getErrorMessage('loanAmount') }}
        </mat-error>
      </mat-form-field>

      <!-- Realtime Toggle -->
      <mat-slide-toggle
        [(ngModel)]="enableRealtime"
        [ngModelOptions]="{ standalone: true }"
        color="primary"
        class="realtime-toggle"
      >
        Real-time predictions
      </mat-slide-toggle>

      <!-- Error Display -->
      <mat-error *ngIf="error$ | async as error" class="form-error">
        <mat-icon>error</mat-icon>
        {{ error }}
      </mat-error>

      <!-- Action Buttons -->
      <div class="form-actions">
        <button
          mat-raised-button
          color="primary"
          type="submit"
          [disabled]="predictionForm.invalid || (isLoading$ | async)"
        >
          <mat-icon *ngIf="!(isLoading$ | async)">send</mat-icon>
          <mat-spinner *ngIf="isLoading$ | async" diameter="20"></mat-spinner>
          {{ (isLoading$ | async) ? 'Predicting...' : 'Get Prediction' }}
        </button>

        <button
          mat-button
          type="button"
          (click)="onReset()"
        >
          <mat-icon>refresh</mat-icon>
          Reset
        </button>
      </div>
    </form>
  </mat-card-content>
</mat-card>
```

### Step 7: HTTP Interceptor

Create `src/app/core/interceptors/auth.interceptor.ts`:

```typescript
import { Injectable } from '@angular/core';
import {
  HttpEvent,
  HttpInterceptor,
  HttpHandler,
  HttpRequest,
  HttpErrorResponse,
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry, finalize } from 'rxjs/operators';
import { MatSnackBar } from '@angular/material/snack-bar';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  private requestCount = 0;

  constructor(private snackBar: MatSnackBar) {}

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    this.requestCount++;

    // Clone request and add auth header
    const authReq = req.clone({
      setHeaders: {
        Authorization: `Bearer ${this.getToken()}`,
        'X-Request-ID': this.generateRequestId(),
      },
    });

    return next.handle(authReq).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.snackBar.open('Authentication failed', 'Close', {
            duration: 3000,
            panelClass: ['error-snackbar'],
          });
        } else if (error.status === 500) {
          this.snackBar.open('Server error occurred', 'Close', {
            duration: 3000,
            panelClass: ['error-snackbar'],
          });
        }

        return throwError(() => error);
      }),
      finalize(() => {
        this.requestCount--;
      })
    );
  }

  private getToken(): string {
    return localStorage.getItem('authToken') || '';
  }

  private generateRequestId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

## Expected Outputs

1. **Real-time Prediction Interface**:
   - Instant predictions as user types
   - Smooth UX with debouncing
   - Material Design components

2. **State Management**:
   - Predictable state with NgRx
   - Time-travel debugging
   - Persisted history

3. **Performance**:
   - Optimized change detection
   - Lazy loading
   - Efficient API calls

4. **Enterprise Features**:
   - Comprehensive error handling
   - Request interceptors
   - Logging and monitoring

## Bonus Challenges

1. **Advanced RxJS**: Custom operators for prediction logic
2. **Progressive Web App**: Service worker for offline support
3. **Accessibility**: Full WCAG 2.1 AA compliance
4. **Internationalization**: Multi-language support with i18n
5. **Advanced Caching**: Implement complex caching strategies
6. **WebSocket Fallback**: Automatic fallback from SSE to WebSocket
7. **Performance Monitoring**: Integrate with analytics

## Resources

- [Angular Documentation](https://angular.io/docs)
- [NgRx Documentation](https://ngrx.io/)
- [RxJS Documentation](https://rxjs.dev/)
- [Angular Material](https://material.angular.io/)
- [ngx-charts](https://swimlane.gitbook.io/ngx-charts/)

## Success Criteria

### Functionality (40%)
- [ ] Real-time predictions working
- [ ] NgRx state management implemented
- [ ] Form validation functional
- [ ] History persistence working
- [ ] Error handling comprehensive

### Code Quality (30%)
- [ ] Proper TypeScript typing
- [ ] RxJS best practices
- [ ] OnPush change detection
- [ ] Clean architecture
- [ ] Unit tests >80% coverage

### User Experience (20%)
- [ ] Material Design implementation
- [ ] Smooth real-time updates
- [ ] Loading indicators
- [ ] Accessible components
- [ ] Responsive design

### Best Practices (10%)
- [ ] Proper observable cleanup
- [ ] Memoized selectors
- [ ] Lazy loading implemented
- [ ] HTTP interceptors
- [ ] Error boundaries
