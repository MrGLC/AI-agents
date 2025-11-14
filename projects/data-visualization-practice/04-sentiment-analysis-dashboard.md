# Project 4: Sentiment Analysis Visualization Dashboard

## Overview
Create an interactive dashboard for sentiment analysis that visualizes text data, sentiment distributions, word clouds, topic modeling, and sentiment trends over time.

## Difficulty Level
Intermediate

## Learning Objectives
- Perform text preprocessing and cleaning
- Implement sentiment analysis models
- Visualize text data and sentiment distributions
- Create word clouds and n-gram analysis
- Track sentiment trends over time
- Build topic modeling visualizations

## Technical Stack
- **Backend**: Python
- **NLP Libraries**: NLTK, spaCy, TextBlob, or Transformers (BERT)
- **Visualization**: Plotly, WordCloud, Matplotlib
- **Dashboard**: Streamlit or Gradio
- **Data Processing**: Pandas, NumPy
- **Dataset**: Twitter data, product reviews, customer feedback, or news articles

## Project Requirements

### 1. Data Input
- Load text data from CSV, JSON, or API
- Support real-time text input for analysis
- Handle multiple text sources
- Date/time parsing for temporal analysis

### 2. Text Processing
- Clean and preprocess text
- Tokenization and lemmatization
- Remove stopwords and special characters
- Extract entities and keywords

### 3. Sentiment Analysis
- Classify sentiment (positive, negative, neutral)
- Calculate sentiment scores (-1 to +1)
- Support multiple sentiment models
- Provide confidence scores

### 4. Core Visualizations
- **Sentiment Distribution**: Pie chart or bar chart
- **Sentiment Over Time**: Line chart with trends
- **Word Cloud**: Separate clouds for each sentiment
- **Top Keywords**: Bar chart of most common words
- **N-gram Analysis**: Bigrams and trigrams
- **Sentiment Score Distribution**: Histogram
- **Entity Analysis**: Named entities by sentiment

### 5. Interactive Features
- Text input box for live analysis
- File upload for batch processing
- Date range selector
- Sentiment filter (show only positive/negative)
- Search and highlight keywords
- Export results to CSV

## Step-by-Step Implementation

### Step 1: Setup Environment
```bash
pip install streamlit pandas plotly textblob nltk wordcloud transformers torch
python -m nltk.downloader punkt stopwords vader_lexicon
```

### Step 2: Text Preprocessing
```python
import re
import nltk
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize
from nltk.stem import WordNetLemmatizer

def preprocess_text(text):
    # Lowercase
    text = text.lower()

    # Remove URLs
    text = re.sub(r'http\S+|www\S+', '', text)

    # Remove mentions and hashtags
    text = re.sub(r'@\w+|#\w+', '', text)

    # Remove special characters
    text = re.sub(r'[^a-zA-Z\s]', '', text)

    # Tokenize
    tokens = word_tokenize(text)

    # Remove stopwords
    stop_words = set(stopwords.words('english'))
    tokens = [t for t in tokens if t not in stop_words]

    # Lemmatize
    lemmatizer = WordNetLemmatizer()
    tokens = [lemmatizer.lemmatize(t) for t in tokens]

    return ' '.join(tokens)
```

### Step 3: Sentiment Analysis
```python
from textblob import TextBlob
from transformers import pipeline

# Option 1: TextBlob (simple)
def analyze_sentiment_textblob(text):
    blob = TextBlob(text)
    polarity = blob.sentiment.polarity

    if polarity > 0.1:
        sentiment = 'positive'
    elif polarity < -0.1:
        sentiment = 'negative'
    else:
        sentiment = 'neutral'

    return sentiment, polarity

# Option 2: Transformers (advanced)
sentiment_pipeline = pipeline('sentiment-analysis',
                              model='distilbert-base-uncased-finetuned-sst-2-english')

def analyze_sentiment_bert(text):
    result = sentiment_pipeline(text[:512])[0]  # BERT has 512 token limit
    return result['label'], result['score']
```

### Step 4: Create Dashboard
```python
import streamlit as st
import pandas as pd
import plotly.express as px
from wordcloud import WordCloud
import matplotlib.pyplot as plt

st.title('💬 Sentiment Analysis Dashboard')

# Sidebar
st.sidebar.header('Options')
analysis_mode = st.sidebar.radio('Mode', ['Single Text', 'Batch Analysis'])

if analysis_mode == 'Single Text':
    # Text input
    user_text = st.text_area('Enter text to analyze:', height=150)

    if st.button('Analyze'):
        if user_text:
            # Preprocess
            clean_text = preprocess_text(user_text)

            # Analyze sentiment
            sentiment, score = analyze_sentiment_textblob(user_text)

            # Display results
            col1, col2 = st.columns(2)
            col1.metric('Sentiment', sentiment.upper())
            col2.metric('Sentiment Score', f'{score:.3f}')

            # Word cloud
            wordcloud = WordCloud(width=800, height=400,
                                background_color='white').generate(clean_text)

            fig, ax = plt.subplots(figsize=(10, 5))
            ax.imshow(wordcloud, interpolation='bilinear')
            ax.axis('off')
            st.pyplot(fig)

else:
    # File upload
    uploaded_file = st.file_uploader('Upload CSV file', type=['csv'])

    if uploaded_file is not None:
        df = pd.read_csv(uploaded_file)
        st.write(f'Loaded {len(df)} records')

        # Analyze all texts
        with st.spinner('Analyzing sentiments...'):
            df['sentiment'], df['score'] = zip(*df['text'].apply(analyze_sentiment_textblob))

        # Sentiment distribution
        sentiment_counts = df['sentiment'].value_counts()

        fig1 = px.pie(values=sentiment_counts.values,
                     names=sentiment_counts.index,
                     title='Sentiment Distribution')
        st.plotly_chart(fig1)

        # Sentiment over time (if date column exists)
        if 'date' in df.columns:
            df['date'] = pd.to_datetime(df['date'])
            sentiment_time = df.groupby([pd.Grouper(key='date', freq='D'), 'sentiment']).size().reset_index(name='count')

            fig2 = px.line(sentiment_time, x='date', y='count',
                          color='sentiment',
                          title='Sentiment Trends Over Time')
            st.plotly_chart(fig2)

        # Top keywords by sentiment
        positive_texts = ' '.join(df[df['sentiment'] == 'positive']['text'])
        negative_texts = ' '.join(df[df['sentiment'] == 'negative']['text'])

        col1, col2 = st.columns(2)

        with col1:
            st.subheader('Positive Word Cloud')
            wc_pos = WordCloud(width=400, height=300,
                              background_color='white').generate(positive_texts)
            fig, ax = plt.subplots()
            ax.imshow(wc_pos, interpolation='bilinear')
            ax.axis('off')
            st.pyplot(fig)

        with col2:
            st.subheader('Negative Word Cloud')
            wc_neg = WordCloud(width=400, height=300,
                              background_color='white').generate(negative_texts)
            fig, ax = plt.subplots()
            ax.imshow(wc_neg, interpolation='bilinear')
            ax.axis('off')
            st.pyplot(fig)
```

### Step 5: Add Advanced Analytics
```python
from collections import Counter
from nltk.util import ngrams

def get_top_ngrams(texts, n=2, top_k=10):
    """Get top n-grams from texts"""
    all_ngrams = []
    for text in texts:
        tokens = word_tokenize(preprocess_text(text))
        all_ngrams.extend(list(ngrams(tokens, n)))

    ngram_counts = Counter(all_ngrams)
    return ngram_counts.most_common(top_k)

# Display top bigrams
st.subheader('Top Bigrams')
top_bigrams = get_top_ngrams(df['text'], n=2, top_k=10)
bigram_df = pd.DataFrame(top_bigrams, columns=['Bigram', 'Count'])
bigram_df['Bigram'] = bigram_df['Bigram'].apply(lambda x: ' '.join(x))

fig = px.bar(bigram_df, x='Count', y='Bigram', orientation='h',
            title='Most Common Bigrams')
st.plotly_chart(fig)
```

## Expected Outputs

1. **Single Text Analysis**:
   - Sentiment label and score
   - Word cloud visualization
   - Key entities extracted

2. **Batch Analysis Dashboard**:
   - Sentiment distribution pie chart
   - Sentiment trends over time (line chart)
   - Separate word clouds for positive/negative
   - Top keywords bar chart
   - N-gram analysis

3. **Metrics Display**:
   - Total texts analyzed
   - Average sentiment score
   - Sentiment breakdown percentages

4. **Interactive Features**:
   - Real-time text analysis
   - File upload and batch processing
   - Date range filtering
   - Export to CSV

## Bonus Challenges

- [ ] Add emotion detection (joy, anger, sadness, fear)
- [ ] Implement aspect-based sentiment analysis
- [ ] Add topic modeling with LDA
- [ ] Create sentiment comparison across sources
- [ ] Add language detection and multi-language support
- [ ] Implement sarcasm detection
- [ ] Add tweet/post classification (spam, not spam)
- [ ] Create interactive topic explorer
- [ ] Add real-time Twitter streaming analysis
- [ ] Implement sentiment prediction for new text

## Resources

- [TextBlob Documentation](https://textblob.readthedocs.io/)
- [Hugging Face Transformers](https://huggingface.co/docs/transformers)
- [NLTK Documentation](https://www.nltk.org/)
- [WordCloud Library](https://github.com/amueller/word_cloud)
- [Sentiment Analysis Guide](https://monkeylearn.com/sentiment-analysis/)

## Success Criteria

- Can analyze sentiment of single texts accurately
- Batch processing works for large datasets
- Visualizations clearly show sentiment patterns
- Word clouds are readable and informative
- Temporal trends are visible (if applicable)
- Dashboard is responsive and user-friendly
- Results can be exported
