# RAG-Based Multimodal Document Q&A System

A FastAPI-based Retrieval-Augmented Generation (RAG) system for intelligent document Q&A. This application allows users to upload PDF documents, process them with LlamaIndex, store embeddings in PGVector, and query the documents using natural language with citations and image support.

## Features

- **PDF Upload & Processing**: Upload multiple PDF documents with automatic text extraction and embedding generation
- **Vector Storage**: Uses PGVector for efficient similarity search across document embeddings
- **RAG Pipeline**: Retrieves relevant document chunks and generates contextual responses using LLMs
- **Chat History**: Supports conversation context for follow-up questions
- **Citation Support**: Returns normalized relevance scores and citations for answers
- **Image Extraction**: Extracts and stores images from PDFs with base64 encoding
- **CRUD Operations**: Full document management (upload, list, delete)
- **Docker Support**: Containerized deployment with Docker Compose

## Tech Stack

- **Backend Framework**: FastAPI
- **Database**: PostgreSQL with PGVector extension
- **Vector Store**: PGVector for embeddings
- **Document Processing**: LlamaIndex, LlamaParse
- **Embeddings**: Google Gemini / OpenAI embeddings
- **LLM**: Google Gemini / OpenAI for response generation
- **ORM**: SQLModel
- **Containerization**: Docker & Docker Compose

## Project Structure

```
folder/
├── main.py                 # FastAPI application & endpoints
├── db.py                   # Database configuration and session management
├── models.py               # SQLModel data models (PDFS, Image)
├── requirements.txt        # Python dependencies
├── Dockerfile              # Container image definition
├── docker-compose.yml      # Multi-container orchestration
├── .env_pgvector          # Environment variables (not tracked)
└── helpers/
    ├── chunker.py         # Document chunking logic
    ├── generator.py       # LLM response generation
    ├── llama_parse_pdf.py # PDF parsing with LlamaParse
    └── retriver.py        # Vector retrieval and similarity search
```

## Prerequisites

- Python 3.11+
- Docker & Docker Compose (for containerized deployment)
- OpenAI API Key or Google Gemini API Key
- LlamaParse API Key (for PDF parsing)

## Environment Variables

Create a `.env_pgvector` file in the project root:

```env
# Database Connection
DATABASE_URL=postgresql://postgres:postgres@db2:5432/dev-db

# PGVector Connection
PGVECTOR_CONNECTION_STRING=postgresql+psycopg2://postgres:postgres@db:5432/dev

# LLM API Keys
OPENAI_API_KEY=your_openai_api_key
GOOGLE_API_KEY=your_google_api_key

# LlamaParse
LLAMA_CLOUD_API_KEY=your_llama_parse_api_key
```

## Installation & Setup

### Option 1: Docker Compose (Recommended)

1. Clone the repository:
```bash
git clone <repository-url>
cd usecase
```

2. Create `.env_pgvector` with your API keys

3. Start the services:
```bash
docker-compose up --build
```

The API will be available at `http://localhost:8000`

### Option 2: Local Development

1. Install dependencies:
```bash
pip install -r requirements.txt
```

2. Start PostgreSQL with PGVector:
```bash
docker run -d \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=dev \
  -p 5432:5432 \
  ankane/pgvector
```

3. Start a second PostgreSQL instance:
```bash
docker run -d \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=dev-db \
  -p 5433:5432 \
  postgres
```

4. Run the application:
```bash
uvicorn main:app --reload
```

## API Endpoints

### Document Management

#### Upload PDFs
```http
POST /upload_pdfs
Content-Type: multipart/form-data

files: <PDF files>
```

Uploads one or more PDF files, processes them, generates embeddings, and stores them in the vector database.

**Response:**
```json
[
  {
    "filename": "document.pdf",
    "file_uuid": "abc123..."
  }
]
```

#### Get All PDFs
```http
GET /get_all_pdfs
```

Returns a list of all uploaded documents.

**Response:**
```json
[
  {
    "pdf_file_name": "document.pdf",
    "pdf_uuid": "abc123..."
  }
]
```

#### Delete PDF
```http
DELETE /delete_pdf/{pdf_uuid}
```

Deletes a specific PDF and its associated embeddings from both the database and vector store.

#### Delete All PDFs
```http
DELETE /delete_all_pdfs
```

Removes all PDFs and their embeddings from the system.

### RAG Query

#### Get Response
```http
POST /get_response
Content-Type: application/json

{
  "query": "What are the key findings?",
  "document_ids": ["abc123...", "def456..."],
  "chad_history": [
    {"role": "user", "content": "Previous question"},
    {"role": "assistant", "content": "Previous answer"}
  ]
}
```

Performs RAG query across specified documents. If `document_ids` is empty, operates in general chat mode without document context.

**Response:**
```json
{
  "response": "{\"answer\": \"...\", \"citation\": [...], \"image\": \"...\"}",
  "relevant_docs": [
    {
      "text": "relevant chunk text",
      "score": 0.85
    }
  ],
  "image_list": ["base64_encoded_image"]
}
```

## How It Works

1. **PDF Upload**: Documents are uploaded via `/upload_pdfs` endpoint
2. **Parsing**: PDFs are parsed using LlamaParse to extract text and images
3. **Chunking**: Text is split into semantic chunks
4. **Embedding**: Chunks are converted to vector embeddings
5. **Storage**: Embeddings stored in PGVector, metadata in PostgreSQL
6. **Query**: User submits a question via `/get_response`
7. **Retrieval**: Similar chunks are retrieved using vector similarity search (score ≥ 0.15)
8. **Generation**: LLM generates a response using retrieved context
9. **Response**: Returns answer with citations and normalized relevance scores

## Database Schema

### PDFS Table
```python
pdf_file_name: str (Primary Key, Indexed)
pdf_uuid: str (Indexed)
```

### Image Table
```python
id: int (Primary Key)
document_id: str (Indexed)
image_id: str (Indexed)
image_b64: str
```

## Architecture

```
┌──────────────┐
│   Client     │
└──────┬───────┘
       │
       v
┌──────────────────┐
│   FastAPI App    │
│   (main.py)      │
└────┬─────┬───────┘
     │     │
     │     └─────────────┐
     │                   │
     v                   v
┌─────────────┐   ┌──────────────┐
│ PostgreSQL  │   │  PGVector DB │
│   (db2)     │   │     (db)     │
└─────────────┘   └──────────────┘
```

## Development

### Adding New Features

- **New endpoints**: Add to [main.py](main.py)
- **Database models**: Define in [models.py](models.py)
- **Helper functions**: Add to `helpers/` directory

### Running Tests

```bash
# Install test dependencies
pip install pytest pytest-asyncio httpx

# Run tests
pytest
```

## Troubleshooting

### Connection Issues
- Ensure both PostgreSQL containers are running
- Verify `.env_pgvector` has correct connection strings
- Check ports 5432 and 5433 are not in use

### Embedding Issues
- Verify API keys are correctly set in environment variables
- Check LlamaParse subscription status
- Ensure PDF files are not corrupted

### Performance
- Adjust similarity threshold (default 0.15) in [main.py](main.py#L122)
- Consider batch processing for large document sets
- Optimize chunk sizes in `helpers/chunker.py`

## License

[Add your license here]

## Contributing

[Add contribution guidelines here]

## Contact

[Add contact information here]
