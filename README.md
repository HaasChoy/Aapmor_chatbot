# RAG API Server with Autonomous Content Synchronization

## Key Features
  - **Dual Ingestion Pipelines**: Ingest knowledge from both local files (`.pdf`, `.docx`, `.txt`) and live web pages.
  - **Intelligent Web Scraping**: Uses Selenium to render JavaScript, ensuring content from modern, dynamic websites is captured accurately.
  - **Autonomous Content Synchronization**: A sophisticated background scheduling system (`APScheduler`) automatically:
      - Polls sitemaps to discover new and deleted pages.
      - Uses a hybrid strategy (HTTP headers & content hashing) to efficiently detect changes on existing pages.
      - Triggers re-scraping and re-indexing of modified content.
      - Cleans up data from deleted pages.
  - **Advanced RAG Core**: Leverages a powerful SentenceTransformer model for embeddings, **Qdrant** for vector search, and the **Groq API** for high-speed, accurate generation.
  - **Robust API**: Built with FastAPI, offering asynchronous performance, automatic documentation (`/docs`), and dependency injection.
  - **Comprehensive Logging & Monitoring**: Detailed logging to both console and MongoDB, with administrative API endpoints to monitor and control the background synchronization tasks.
  - **Containerized**: Fully containerized with Docker Compose for easy setup and deployment of the application and its databases (Qdrant, MongoDB).

## System Architecture

    subgraph RAG API Server (FastAPI)
        direction LR
        API[API Endpoints <br> (/chat, /query, /scrape, ...)]
        RAGChatbot[ RAGChatbot]
        WebScraper[🕷 WebScraper]
        DocProcessor[ DocumentProcessor]
        Scheduler[ APScheduler]
    end
    
    subgraph Data Stores
        Qdrant[( Qdrant DB)]
        MongoDB[( MongoDB)]
    end

    subgraph External Services
        Groq[ Groq API (LLM)]
        Websites[Target Websites]
    end

    User -- "Sends Question / Scrape Request" --> API
    API -- "Processes Request" --> RAGChatbot
    API -- "Triggers Ingestion" --> WebScraper
    API -- "Triggers Ingestion" --> DocProcessor

    RAGChatbot -- "1. Embeds Question" --> SentenceTransformer( Embedding Model)
    RAGChatbot -- "2. Searches for Context" --> Qdrant
    RAGChatbot -- "3. Augments & Sends Prompt" --> Groq
    Groq -- "4. Returns Answer" --> RAGChatbot
    RAGChatbot -- "5. Logs Interaction" --> MongoDB
    RAGChatbot -- "Returns Answer" --> API
    API -- "Sends Response" --> User

    WebScraper -- "Scrapes Pages" --> Websites
    WebScraper -- "Stores Embeddings & Metadata" --> Qdrant & MongoDB
    DocProcessor -- "Stores Embeddings & Metadata" --> Qdrant & MongoDB

    Scheduler -- "Triggers on Schedule" --> ContentSynchronizer( Content Synchronizer)
    ContentSynchronizer -- "Checks Sitemaps & Pages" --> Websites
    ContentSynchronizer -- "Updates URL Tracker" --> MongoDB
    ContentSynchronizer -- "Flags URLs for..." --> WebScraper
```

## Tech Stack

  - **Backend**: FastAPI, Uvicorn
  - **Vector Database**: Qdrant
  - **Metadata/Logging Database**: MongoDB
  - **LLM Provider**: Groq
  - **Core AI/ML**: SentenceTransformers, PyTorch, Langchain
  - **Web Scraping**: Selenium, BeautifulSoup, HTTPX
  - **Scheduling**: APScheduler
  - **Deployment**: Docker & Docker Compose

## Getting Started

### Prerequisites

  - Docker and Docker Compose
  - Python 3.12+
  - An API key from [Groq](https://console.groq.com/keys)

### Installation & Setup

1.  **Clone the repository:**

    ```bash
    git clone https://github.com/your-username/your-repo.git
    cd your-repo
    ```

2.  **Create the environment file:**
    Copy the example environment file and edit it with your specific settings.

    ```bash
    cp .env.example .env
    ```

    Now, open the `.env` file and fill in the required values:

    **File: `.env`**

    ```env
    # --- Required Settings ---
    GROQ_API_KEY="gsk_YourGroqApiKeyHere"

    # --- Autonomous Synchronization Settings ---
    # Add a comma-separated list of sitemaps to monitor
    TARGET_COMPANY_SITEMAPS="https://www.example.com/sitemap.xml,https://www.another-site.com/sitemap_index.xml"

    # You can override other settings from app_config.py here if needed
    # e.g., LOG_LEVEL=DEBUG
    ```

3.  **Run the Application with Docker Compose (Recommended):**
    This is the easiest way to start the entire system, including the databases.

    ```bash
    docker compose up -d
    ```

    The first time you run this, Docker will build the `backend` image, which may take a few minutes. Your RAG API Server will be available at `http://localhost:8000`.

### Local Development (Alternative)

If you want to run the FastAPI server directly on your host for development:

1.  **Start Databases**: First, start the Qdrant and MongoDB services using Docker Compose.
    ```bash
    docker compose up -d qdrant mongo
    ```
2.  **Create Virtual Environment**:
    ```bash
    python -m venv venv
    source venv/bin/activate
    ```
3.  **Install Dependencies**:
    ```bash
    pip install -r requirements.txt
    ```
4.  **Run the Server**:
    ```bash
    uvicorn main:app --reload
    ```

## API Usage Examples

### Ingesting Data

**Scrape URLs:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{
  "urls": ["https://www.promptior.ai/about", "https://www.promptior.ai/solutions"],
  "replace_existing": true
}' http://127.0.0.1:8000/api/scraper/scrape
```

**Upload a File:**

```bash
# Make sure you have a file named 'my_document.pdf' in the same directory
curl -X POST -F "file=@my_document.pdf" http://127.0.0.1:8000/api/documents/upload
```

### Querying the RAG System

**Get a detailed answer with context:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{
  "question": "What kind of cloud transformation services does the company provide?"
}' http://127.0.0.1:8000/api/query
```

**Get a simple chat response:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{
  "question": "Who can I contact for more information?"
}' http://127.0.0.1:8000/api/chat/ask
```

### Administration & Monitoring

**Check the status of background jobs:**

```bash
curl http://127.0.0.1:8000/scheduler/status
```

**Manually trigger the sitemap poll:**

```bash
curl -X POST http://127.0.0.1:8000/api/admin/trigger-sitemap-poll
```

## Configuration

The application is configured via environment variables defined in the `.env` file. See `app_config.py` for a full list of available settings. Key variables include:

| Variable                               | Description                                                               | Default             |
| -------------------------------------- | ------------------------------------------------------------------------- | ------------------- |
| `GROQ_API_KEY`                         | **Required.** Your API key for the Groq service.                          | `...`               |
| `TARGET_COMPANY_SITEMAPS`              | Comma-separated list of sitemap URLs to monitor.                          | `[]`                |
| `SITEMAP_POLL_INTERVAL_SECONDS`        | How often to check sitemaps for new/deleted URLs.                         | `86400` (24h)       |
| `CONTENT_CHECK_INTERVAL_SECONDS`       | How often to check existing URLs for content changes.                     | `21600` (6h)        |
| `LOG_LEVEL`                            | The logging level for the application (`INFO`, `DEBUG`, etc.).            | `INFO`              |
| `QDRANT_COLLECTION`                    | The name of the main Qdrant collection for document embeddings.           | `documents`         |
