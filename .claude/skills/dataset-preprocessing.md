# Dataset Preprocessing Skill

This skill helps you clean, transform, and prepare datasets for machine learning.

## Common Preprocessing Tasks

- Handling missing values
- Encoding categorical variables
- Feature scaling and normalization
- Feature engineering
- Outlier detection and treatment
- Data splitting (train/validation/test)
- Handling imbalanced datasets
- Text preprocessing
- Image preprocessing
- Time series transformations

## When to Use This Skill

- Starting a new ML project
- Data quality issues affecting model performance
- Preparing raw data for model training
- Feature engineering to improve accuracy
- Adapting data for different model types
- Creating reproducible preprocessing pipelines

## Handling Missing Values

### Detection
```python
import pandas as pd
import numpy as np

# Load data
df = pd.read_csv('data.csv')

# Check for missing values
print(df.isnull().sum())
print(f"Total missing: {df.isnull().sum().sum()}")

# Visualize missing data
import missingno as msno
msno.matrix(df)
```

### Strategies

```python
# Drop rows with any missing values
df_clean = df.dropna()

# Drop columns with more than 50% missing
threshold = len(df) * 0.5
df_clean = df.dropna(thresh=threshold, axis=1)

# Fill with mean/median/mode
df['numeric_col'].fillna(df['numeric_col'].mean(), inplace=True)
df['categorical_col'].fillna(df['categorical_col'].mode()[0], inplace=True)

# Forward fill (time series)
df['value'].fillna(method='ffill', inplace=True)

# Interpolation
df['value'].interpolate(method='linear', inplace=True)

# Using scikit-learn
from sklearn.impute import SimpleImputer

imputer = SimpleImputer(strategy='mean')
df[numeric_cols] = imputer.fit_transform(df[numeric_cols])
```

## Encoding Categorical Variables

### Label Encoding
```python
from sklearn.preprocessing import LabelEncoder

le = LabelEncoder()
df['category_encoded'] = le.fit_transform(df['category'])

# Inverse transform
df['category_original'] = le.inverse_transform(df['category_encoded'])
```

### One-Hot Encoding
```python
# Using pandas
df_encoded = pd.get_dummies(df, columns=['category'], drop_first=True)

# Using scikit-learn
from sklearn.preprocessing import OneHotEncoder

encoder = OneHotEncoder(sparse=False, drop='first')
encoded = encoder.fit_transform(df[['category']])
encoded_df = pd.DataFrame(encoded, columns=encoder.get_feature_names_out())
```

### Target Encoding
```python
# Mean encoding
target_encoding = df.groupby('category')['target'].mean()
df['category_encoded'] = df['category'].map(target_encoding)
```

## Feature Scaling

### Standardization (Z-score)
```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
df[numeric_cols] = scaler.fit_transform(df[numeric_cols])

# Mean = 0, Std = 1
```

### Min-Max Normalization
```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler(feature_range=(0, 1))
df[numeric_cols] = scaler.fit_transform(df[numeric_cols])

# Range: [0, 1]
```

### Robust Scaling
```python
from sklearn.preprocessing import RobustScaler

# Less sensitive to outliers
scaler = RobustScaler()
df[numeric_cols] = scaler.fit_transform(df[numeric_cols])
```

## Feature Engineering

### Creating New Features
```python
# Mathematical operations
df['feature_ratio'] = df['feature1'] / df['feature2']
df['feature_product'] = df['feature1'] * df['feature2']
df['feature_diff'] = df['feature1'] - df['feature2']

# Polynomial features
from sklearn.preprocessing import PolynomialFeatures

poly = PolynomialFeatures(degree=2, include_bias=False)
poly_features = poly.fit_transform(df[['feature1', 'feature2']])

# Binning
df['age_group'] = pd.cut(df['age'], bins=[0, 18, 35, 60, 100],
                          labels=['child', 'young', 'middle', 'senior'])

# Date features
df['date'] = pd.to_datetime(df['date'])
df['year'] = df['date'].dt.year
df['month'] = df['date'].dt.month
df['day_of_week'] = df['date'].dt.dayofweek
df['is_weekend'] = df['day_of_week'].isin([5, 6]).astype(int)
```

### Feature Selection
```python
from sklearn.feature_selection import SelectKBest, f_classif, RFE
from sklearn.ensemble import RandomForestClassifier

# Univariate selection
selector = SelectKBest(f_classif, k=10)
X_selected = selector.fit_transform(X, y)

# Recursive feature elimination
rf = RandomForestClassifier()
rfe = RFE(estimator=rf, n_features_to_select=10)
X_selected = rfe.fit_transform(X, y)

# Feature importance from tree models
rf = RandomForestClassifier()
rf.fit(X, y)
importances = pd.Series(rf.feature_importances_, index=X.columns)
top_features = importances.nlargest(10).index
```

## Outlier Detection

### Statistical Methods
```python
# IQR method
Q1 = df['feature'].quantile(0.25)
Q3 = df['feature'].quantile(0.75)
IQR = Q3 - Q1
lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR

outliers = df[(df['feature'] < lower_bound) | (df['feature'] > upper_bound)]

# Z-score method
from scipy import stats
z_scores = np.abs(stats.zscore(df[numeric_cols]))
outliers = df[(z_scores > 3).any(axis=1)]

# Remove outliers
df_clean = df[(z_scores <= 3).all(axis=1)]
```

### ML-Based Detection
```python
from sklearn.ensemble import IsolationForest

iso_forest = IsolationForest(contamination=0.1, random_state=42)
outliers = iso_forest.fit_predict(df[numeric_cols])
df_clean = df[outliers == 1]
```

## Data Splitting

```python
from sklearn.model_selection import train_test_split

# Simple split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Stratified split (for classification)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

# Time series split (no shuffling)
split_index = int(len(df) * 0.8)
train_df = df[:split_index]
test_df = df[split_index:]

# K-fold cross-validation
from sklearn.model_selection import KFold

kf = KFold(n_splits=5, shuffle=True, random_state=42)
for train_idx, val_idx in kf.split(X):
    X_train, X_val = X[train_idx], X[val_idx]
    y_train, y_val = y[train_idx], y[val_idx]
```

## Handling Imbalanced Data

### Resampling
```python
from imblearn.over_sampling import SMOTE, RandomOverSampler
from imblearn.under_sampling import RandomUnderSampler

# Oversampling minority class
ros = RandomOverSampler(random_state=42)
X_resampled, y_resampled = ros.fit_resample(X, y)

# SMOTE (Synthetic Minority Over-sampling)
smote = SMOTE(random_state=42)
X_resampled, y_resampled = smote.fit_resample(X, y)

# Undersampling majority class
rus = RandomUnderSampler(random_state=42)
X_resampled, y_resampled = rus.fit_resample(X, y)
```

### Class Weights
```python
from sklearn.utils.class_weight import compute_class_weight

# Compute balanced class weights
class_weights = compute_class_weight('balanced', classes=np.unique(y), y=y)
class_weight_dict = dict(enumerate(class_weights))

# Use in model
model = RandomForestClassifier(class_weight='balanced')
```

## Text Preprocessing

```python
import re
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize
from nltk.stem import PorterStemmer, WordNetLemmatizer

def preprocess_text(text):
    # Lowercase
    text = text.lower()

    # Remove URLs
    text = re.sub(r'http\S+|www\S+', '', text)

    # Remove mentions and hashtags
    text = re.sub(r'@\w+|#\w+', '', text)

    # Remove punctuation and numbers
    text = re.sub(r'[^a-zA-Z\s]', '', text)

    # Tokenize
    tokens = word_tokenize(text)

    # Remove stopwords
    stop_words = set(stopwords.words('english'))
    tokens = [t for t in tokens if t not in stop_words]

    # Lemmatization
    lemmatizer = WordNetLemmatizer()
    tokens = [lemmatizer.lemmatize(t) for t in tokens]

    return ' '.join(tokens)

df['text_clean'] = df['text'].apply(preprocess_text)
```

## Creating Preprocessing Pipelines

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer

# Define transformers for different column types
numeric_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='mean')),
    ('scaler', StandardScaler())
])

categorical_transformer = Pipeline(steps=[
    ('imputer', SimpleImputer(strategy='constant', fill_value='missing')),
    ('onehot', OneHotEncoder(handle_unknown='ignore'))
])

# Combine transformers
preprocessor = ColumnTransformer(
    transformers=[
        ('num', numeric_transformer, numeric_features),
        ('cat', categorical_transformer, categorical_features)
    ])

# Full pipeline with model
from sklearn.ensemble import RandomForestClassifier

pipeline = Pipeline(steps=[
    ('preprocessor', preprocessor),
    ('classifier', RandomForestClassifier())
])

# Fit and predict
pipeline.fit(X_train, y_train)
predictions = pipeline.predict(X_test)
```

## Best Practices

- **Understand your data first**: EDA before preprocessing
- **Document transformations**: Keep track of all changes
- **Avoid data leakage**: Fit only on training data
- **Use pipelines**: Ensure reproducibility
- **Save preprocessors**: For consistent production inference
- **Validate assumptions**: Check distributions after transformations
- **Handle new categories**: In production data
- **Version your preprocessing**: Track changes over time
- **Test edge cases**: Null values, outliers, new categories

## Common Pitfalls

- Fitting scalers on entire dataset (including test set)
- Dropping too many rows/columns with missing values
- Not handling unseen categories in production
- Over-engineering features without validation
- Ignoring domain knowledge in feature creation
- Not preserving data types after transformations
- Forgetting to apply same preprocessing at inference time
