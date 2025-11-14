# Project 02: Build a RAG Chatbot with Document Retrieval

## Overview
Build a **Retrieval-Augmented Generation (RAG) chatbot** that can answer questions based on your custom documents. This system combines document retrieval with LLM generation to provide accurate, context-aware answers grounded in your data.

## Learning Objectives
- Understand RAG architecture and workflow
- Implement document loading and text splitting strategies
- Create and query vector databases for semantic search
- Build retrieval chains with source citations
- Optimize embedding and retrieval parameters
- Handle multiple document formats

## Difficulty Level
**Intermediate** - Requires understanding of embeddings, vector databases, and retrieval strategies.

## Technical Stack
- **Framework**: LangChain
- **LLM**: OpenAI GPT-4 or GPT-3.5-turbo
- **Embeddings**: OpenAI Embeddings (text-embedding-3-small)
- **Vector Store**: Chroma (local) or Pinecone (cloud)
- **Document Loaders**: PyPDF, UnstructuredLoader, TextLoader
- **Additional**: FAISS (optional), tiktoken

## Project Requirements

### RAG Pipeline Design
1. **Document Ingestion**: Load documents from various sources
2. **Text Splitting**: Chunk documents intelligently
3. **Embedding**: Convert chunks to vector embeddings
4. **Vector Storage**: Store embeddings in vector database
5. **Retrieval**: Find relevant chunks based on query
6. **Generation**: Generate answers using retrieved context

### Document Processing
- Support PDF, TXT, Markdown, and HTML files
- Implement smart chunking with overlap
- Preserve document metadata (source, page number)
- Handle large documents efficiently

### Retrieval Strategy
- Semantic similarity search
- MMR (Maximum Marginal Relevance) for diversity
- Configurable number of retrieved documents
- Re-ranking for improved relevance

### Safety and Quality
- Source attribution for all answers
- Confidence scoring
- Handling of "I don't know" scenarios
- Input sanitization

## Step-by-Step Implementation

### Step 1: Environment Setup

```python
# requirements.txt
langchain>=0.1.0
langchain-openai>=0.0.5
langchain-community>=0.0.20
chromadb>=0.4.22
pypdf>=4.0.0
unstructured>=0.12.0
tiktoken>=0.5.2
python-dotenv>=1.0.0
```

```bash
pip install -r requirements.txt
```

```python
# .env file
OPENAI_API_KEY=your_openai_api_key_here
```

### Step 2: Document Loading and Processing

```python
# document_processor.py
from langchain.document_loaders import (
    PyPDFLoader,
    TextLoader,
    DirectoryLoader,
    UnstructuredMarkdownLoader
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
from typing import List
from langchain.schema import Document
import os

class DocumentProcessor:
    """Handle document loading and chunking."""

    def __init__(self, chunk_size: int = 1000, chunk_overlap: int = 200):
        """
        Initialize document processor.

        Args:
            chunk_size: Size of each text chunk
            chunk_overlap: Overlap between chunks for context preservation
        """
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap,
            length_function=len,
            separators=["\n\n", "\n", " ", ""]
        )

    def load_pdf(self, file_path: str) -> List[Document]:
        """Load and split a PDF file."""
        loader = PyPDFLoader(file_path)
        documents = loader.load()
        return self.text_splitter.split_documents(documents)

    def load_text(self, file_path: str) -> List[Document]:
        """Load and split a text file."""
        loader = TextLoader(file_path, encoding='utf-8')
        documents = loader.load()
        return self.text_splitter.split_documents(documents)

    def load_markdown(self, file_path: str) -> List[Document]:
        """Load and split a Markdown file."""
        loader = UnstructuredMarkdownLoader(file_path)
        documents = loader.load()
        return self.text_splitter.split_documents(documents)

    def load_directory(self, directory_path: str, glob_pattern: str = "**/*.pdf") -> List[Document]:
        """
        Load all documents from a directory.

        Args:
            directory_path: Path to directory
            glob_pattern: Pattern to match files (e.g., "**/*.pdf", "**/*.txt")

        Returns:
            List of document chunks
        """
        loader = DirectoryLoader(
            directory_path,
            glob=glob_pattern,
            show_progress=True
        )
        documents = loader.load()
        return self.text_splitter.split_documents(documents)

    def load_multiple_files(self, file_paths: List[str]) -> List[Document]:
        """Load multiple files of different types."""
        all_documents = []

        for file_path in file_paths:
            ext = os.path.splitext(file_path)[1].lower()

            if ext == '.pdf':
                docs = self.load_pdf(file_path)
            elif ext == '.txt':
                docs = self.load_text(file_path)
            elif ext == '.md':
                docs = self.load_markdown(file_path)
            else:
                print(f"Skipping unsupported file type: {file_path}")
                continue

            all_documents.extend(docs)
            print(f"Loaded {len(docs)} chunks from {file_path}")

        return all_documents

# Example usage
if __name__ == "__main__":
    processor = DocumentProcessor(chunk_size=1000, chunk_overlap=200)

    # Load single PDF
    chunks = processor.load_pdf("data/document.pdf")
    print(f"Created {len(chunks)} chunks")

    # Load entire directory
    all_chunks = processor.load_directory("data/", glob_pattern="**/*.pdf")
    print(f"Total chunks: {len(all_chunks)}")
```

### Step 3: Vector Store Setup

```python
# vector_store.py
import os
from typing import List
from dotenv import load_dotenv
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.schema import Document

load_dotenv()

class VectorStoreManager:
    """Manage vector store operations."""

    def __init__(self, collection_name: str = "rag_chatbot", persist_directory: str = "./chroma_db"):
        """
        Initialize vector store manager.

        Args:
            collection_name: Name of the collection
            persist_directory: Directory to persist the database
        """
        self.collection_name = collection_name
        self.persist_directory = persist_directory
        self.embeddings = OpenAIEmbeddings(
            model="text-embedding-3-small",
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )
        self.vectorstore = None

    def create_vectorstore(self, documents: List[Document]) -> Chroma:
        """
        Create a new vector store from documents.

        Args:
            documents: List of document chunks

        Returns:
            Chroma vector store
        """
        print(f"Creating vector store with {len(documents)} documents...")

        self.vectorstore = Chroma.from_documents(
            documents=documents,
            embedding=self.embeddings,
            collection_name=self.collection_name,
            persist_directory=self.persist_directory
        )

        print(f"Vector store created and persisted to {self.persist_directory}")
        return self.vectorstore

    def load_vectorstore(self) -> Chroma:
        """Load existing vector store from disk."""
        print(f"Loading vector store from {self.persist_directory}...")

        self.vectorstore = Chroma(
            collection_name=self.collection_name,
            embedding_function=self.embeddings,
            persist_directory=self.persist_directory
        )

        return self.vectorstore

    def add_documents(self, documents: List[Document]):
        """Add new documents to existing vector store."""
        if self.vectorstore is None:
            raise ValueError("Vector store not initialized. Call create_vectorstore or load_vectorstore first.")

        self.vectorstore.add_documents(documents)
        print(f"Added {len(documents)} documents to vector store")

    def similarity_search(self, query: str, k: int = 4) -> List[Document]:
        """
        Perform similarity search.

        Args:
            query: Search query
            k: Number of documents to return

        Returns:
            List of most relevant documents
        """
        if self.vectorstore is None:
            raise ValueError("Vector store not initialized")

        return self.vectorstore.similarity_search(query, k=k)

    def similarity_search_with_score(self, query: str, k: int = 4) -> List[tuple]:
        """
        Perform similarity search with relevance scores.

        Args:
            query: Search query
            k: Number of documents to return

        Returns:
            List of (document, score) tuples
        """
        if self.vectorstore is None:
            raise ValueError("Vector store not initialized")

        return self.vectorstore.similarity_search_with_score(query, k=k)

    def mmr_search(self, query: str, k: int = 4, fetch_k: int = 20) -> List[Document]:
        """
        Perform Maximum Marginal Relevance search for diverse results.

        Args:
            query: Search query
            k: Number of documents to return
            fetch_k: Number of documents to fetch before MMR

        Returns:
            List of diverse relevant documents
        """
        if self.vectorstore is None:
            raise ValueError("Vector store not initialized")

        return self.vectorstore.max_marginal_relevance_search(
            query,
            k=k,
            fetch_k=fetch_k
        )

    def delete_collection(self):
        """Delete the entire collection."""
        if self.vectorstore:
            self.vectorstore.delete_collection()
            print(f"Deleted collection: {self.collection_name}")
```

### Step 4: Build the RAG Chain

```python
# rag_chain.py
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate
from langchain.schema import Document
from typing import List, Dict

load_dotenv()

class RAGChatbot:
    """RAG-powered chatbot for question answering."""

    def __init__(self, vectorstore_manager, model_name: str = "gpt-3.5-turbo", temperature: float = 0):
        """
        Initialize RAG chatbot.

        Args:
            vectorstore_manager: VectorStoreManager instance
            model_name: OpenAI model to use
            temperature: LLM temperature
        """
        self.vectorstore_manager = vectorstore_manager
        self.llm = ChatOpenAI(
            model=model_name,
            temperature=temperature,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )
        self.qa_chain = None
        self._setup_chain()

    def _setup_chain(self):
        """Setup the RetrievalQA chain."""

        # Custom prompt template
        template = """You are a helpful assistant that answers questions based on the provided context.
Use the following pieces of context to answer the question at the end.

If you don't know the answer based on the context, just say "I don't have enough information to answer this question."
Don't try to make up an answer.

Always cite your sources by mentioning which document the information came from.

Context:
{context}

Question: {question}

Helpful Answer:"""

        QA_PROMPT = PromptTemplate(
            template=template,
            input_variables=["context", "question"]
        )

        # Create retriever
        retriever = self.vectorstore_manager.vectorstore.as_retriever(
            search_type="similarity",
            search_kwargs={"k": 4}
        )

        # Create RetrievalQA chain
        self.qa_chain = RetrievalQA.from_chain_type(
            llm=self.llm,
            chain_type="stuff",
            retriever=retriever,
            return_source_documents=True,
            chain_type_kwargs={"prompt": QA_PROMPT}
        )

    def ask(self, question: str) -> Dict:
        """
        Ask a question and get an answer with sources.

        Args:
            question: User question

        Returns:
            Dictionary with answer and source documents
        """
        result = self.qa_chain.invoke({"query": question})

        return {
            "question": question,
            "answer": result["result"],
            "sources": self._format_sources(result["source_documents"])
        }

    def _format_sources(self, documents: List[Document]) -> List[Dict]:
        """Format source documents for display."""
        sources = []
        for i, doc in enumerate(documents, 1):
            source_info = {
                "number": i,
                "content": doc.page_content[:200] + "...",
                "metadata": doc.metadata
            }
            sources.append(source_info)
        return sources

    def ask_with_mmr(self, question: str, k: int = 4) -> Dict:
        """Ask a question using MMR search for diverse results."""

        # Create MMR retriever
        mmr_retriever = self.vectorstore_manager.vectorstore.as_retriever(
            search_type="mmr",
            search_kwargs={"k": k, "fetch_k": 20}
        )

        # Temporarily update the chain's retriever
        original_retriever = self.qa_chain.retriever
        self.qa_chain.retriever = mmr_retriever

        result = self.qa_chain.invoke({"query": question})

        # Restore original retriever
        self.qa_chain.retriever = original_retriever

        return {
            "question": question,
            "answer": result["result"],
            "sources": self._format_sources(result["source_documents"])
        }
```

### Step 5: Create the Main Application

```python
# main.py
import os
from document_processor import DocumentProcessor
from vector_store import VectorStoreManager
from rag_chain import RAGChatbot

def setup_rag_system(data_directory: str, force_rebuild: bool = False):
    """
    Setup the RAG system by processing documents and creating vector store.

    Args:
        data_directory: Directory containing documents
        force_rebuild: If True, rebuild vector store from scratch

    Returns:
        RAGChatbot instance
    """
    # Initialize components
    processor = DocumentProcessor(chunk_size=1000, chunk_overlap=200)
    vectorstore_manager = VectorStoreManager(
        collection_name="rag_chatbot",
        persist_directory="./chroma_db"
    )

    # Check if vector store exists
    if os.path.exists("./chroma_db") and not force_rebuild:
        print("Loading existing vector store...")
        vectorstore_manager.load_vectorstore()
    else:
        print("Creating new vector store...")

        # Load and process documents
        documents = processor.load_directory(data_directory, glob_pattern="**/*.*")

        if not documents:
            raise ValueError(f"No documents found in {data_directory}")

        print(f"Processed {len(documents)} document chunks")

        # Create vector store
        vectorstore_manager.create_vectorstore(documents)

    # Create chatbot
    chatbot = RAGChatbot(vectorstore_manager, model_name="gpt-3.5-turbo")

    return chatbot

def main():
    """Interactive CLI for RAG chatbot."""

    print("RAG Chatbot - Document Question Answering")
    print("=" * 60)

    # Setup
    data_dir = input("Enter path to documents directory (default: ./data): ").strip()
    if not data_dir:
        data_dir = "./data"

    try:
        chatbot = setup_rag_system(data_dir)
        print("\n✓ RAG system initialized successfully!")
    except Exception as e:
        print(f"\n✗ Error setting up RAG system: {e}")
        return

    print("\nYou can now ask questions about your documents.")
    print("Type 'quit' to exit, 'mmr' to toggle MMR search.\n")

    use_mmr = False

    while True:
        question = input("\nYour question: ").strip()

        if question.lower() in ['quit', 'exit', 'q']:
            print("Goodbye!")
            break

        if question.lower() == 'mmr':
            use_mmr = not use_mmr
            print(f"MMR search {'enabled' if use_mmr else 'disabled'}")
            continue

        if not question:
            continue

        try:
            print("\n" + "=" * 60)

            if use_mmr:
                result = chatbot.ask_with_mmr(question)
            else:
                result = chatbot.ask(question)

            print(f"\n📝 Answer:\n{result['answer']}")

            print(f"\n📚 Sources ({len(result['sources'])}):")
            for source in result['sources']:
                metadata = source['metadata']
                source_name = metadata.get('source', 'Unknown')
                page = metadata.get('page', 'N/A')
                print(f"\n  [{source['number']}] {source_name} (Page: {page})")
                print(f"      {source['content']}")

            print("=" * 60)

        except Exception as e:
            print(f"\n✗ Error: {e}")

if __name__ == "__main__":
    main()
```

### Step 6: Advanced Features - Conversation Memory

```python
# conversational_rag.py
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferMemory
from langchain_openai import ChatOpenAI
import os

class ConversationalRAG:
    """RAG chatbot with conversation memory."""

    def __init__(self, vectorstore_manager, model_name: str = "gpt-3.5-turbo"):
        """Initialize conversational RAG."""

        self.llm = ChatOpenAI(
            model=model_name,
            temperature=0,
            openai_api_key=os.getenv("OPENAI_API_KEY")
        )

        # Setup memory
        self.memory = ConversationBufferMemory(
            memory_key="chat_history",
            return_messages=True,
            output_key="answer"
        )

        # Create retriever
        retriever = vectorstore_manager.vectorstore.as_retriever(
            search_kwargs={"k": 4}
        )

        # Create conversational chain
        self.chain = ConversationalRetrievalChain.from_llm(
            llm=self.llm,
            retriever=retriever,
            memory=self.memory,
            return_source_documents=True
        )

    def ask(self, question: str):
        """Ask a question with conversation context."""
        result = self.chain.invoke({"question": question})

        return {
            "answer": result["answer"],
            "sources": result["source_documents"]
        }

    def clear_memory(self):
        """Clear conversation memory."""
        self.memory.clear()
```

## Expected Outputs

### Example 1: Simple Question
```
Your question: What is the main topic of the document?

📝 Answer:
The main topic of the document is artificial intelligence and machine learning
applications in healthcare. The document discusses various use cases including
diagnostic imaging, patient monitoring, and treatment recommendations.

📚 Sources (4):
  [1] healthcare_ai.pdf (Page: 1)
      Artificial intelligence is transforming healthcare by enabling more
      accurate diagnoses and personalized treatment plans...

  [2] healthcare_ai.pdf (Page: 5)
      Machine learning models have shown promising results in detecting
      diseases from medical imaging data...
```

### Example 2: Multi-hop Reasoning
```
Your question: Compare the costs mentioned in section 3 with the benefits in section 5.

📝 Answer:
According to the document, section 3 outlines implementation costs of $500,000
for the AI system, while section 5 reports annual benefits of $2M through
improved diagnostic accuracy and reduced readmission rates, showing a positive
ROI within the first year.

📚 Sources (4):
  [showing relevant excerpts from sections 3 and 5]
```

## Bonus Challenges

1. **Multi-Modal RAG**:
   - Add image processing capabilities
   - Extract text from images (OCR)
   - Handle tables and charts

2. **Hybrid Search**:
   - Combine semantic and keyword search
   - Implement BM25 + vector search
   - Add re-ranking layer

3. **Advanced Chunking**:
   - Implement semantic chunking
   - Use sentence-window retrieval
   - Add parent-child document relationships

4. **Query Enhancement**:
   - Query expansion with synonyms
   - Multi-query generation
   - Hypothetical document embeddings (HyDE)

5. **Evaluation Framework**:
   - Create test question/answer pairs
   - Measure retrieval accuracy
   - Calculate answer quality metrics

6. **Web Interface**:
   - Build Streamlit or Gradio UI
   - Add document upload functionality
   - Visualize retrieved chunks

7. **Production Features**:
   - Implement caching layer
   - Add usage analytics
   - Set up A/B testing for prompts

## Resources

### Documentation
- [LangChain RAG Tutorial](https://python.langchain.com/docs/use_cases/question_answering/)
- [ChromaDB Documentation](https://docs.trychroma.com/)
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)

### Research Papers
- [Retrieval-Augmented Generation (RAG)](https://arxiv.org/abs/2005.11401)
- [Dense Passage Retrieval](https://arxiv.org/abs/2004.04906)

### Tools
- [LlamaIndex](https://www.llamaindex.ai/) - Alternative RAG framework
- [Pinecone](https://www.pinecone.io/) - Managed vector database
- [Weaviate](https://weaviate.io/) - Open-source vector database

## Success Criteria

- [ ] Successfully loads and chunks various document types
- [ ] Creates and persists vector store
- [ ] Retrieves relevant documents for queries
- [ ] Generates accurate answers with source citations
- [ ] Handles "I don't know" scenarios appropriately
- [ ] Supports multiple retrieval strategies (similarity, MMR)
- [ ] Provides clear source attribution
- [ ] Responds within acceptable time limits (<5s)
- [ ] Code is modular and well-documented
- [ ] Error handling for edge cases

## Testing Checklist

- [ ] Test with different document types (PDF, TXT, MD)
- [ ] Test with various chunk sizes
- [ ] Test with single vs. multiple documents
- [ ] Test retrieval accuracy with known questions
- [ ] Test with ambiguous queries
- [ ] Test with out-of-domain questions
- [ ] Verify source citations are accurate
- [ ] Test with large document sets (>100 docs)
- [ ] Measure retrieval latency
- [ ] Test vector store persistence and loading

## Next Steps

After completing this project:
1. Move on to Project 03: Multi-Agent System
2. Experiment with different embedding models
3. Try different vector databases (Pinecone, Weaviate)
4. Implement evaluation metrics
5. Deploy as a web service
