# RIMES API

Modern REST API for French rhyme dictionary application.

## Overview

This project provides a modern web API for the French rhyme dictionary application: 

- Checking words against phonetic rhyme patterns (`tG=F`, `tG=K`, etc.)
- Adding new words to the dictionary
- Retrieving statistics about the dictionary
- Dockerized deployment with PostgreSQL and Nginx reverse proxy

## Features

- **RESTful Endpoints**
  - `POST /rime/check` - Check if a word matches any rhyme pattern
  - `POST /rime/add` - Add a new word to the dictionary
  - `GET /rime/stats` - Get dictionary statistics
  - `GET /` - Welcome message
  - `GET /health` - Health check

- **Wildcard Support**  
  Legacy wildcards from the original VB3 app are supported:
  - `*` → one or more letters
  - `?` → exactly one letter
  - `!` → not a vowel (AEIOUY)
  - `$` → only B or D
  - `[!AE]` → exclude A and E
  - `[BD]` → only B or D

- **Multiple Storage Options**
  - SQLite (default, for development/testing)
  - PostgreSQL (for production/Docker deployments)
  - In-memory SQLite (for fast tests)

- **Docker Ready**
  - Multi-stage Docker build for minimal image size
  - docker-compose with PostgreSQL, API, and Nginx reverse proxy
  - Health checks and automatic restart policies

## Quick Start

### Prerequisites
- Python 3.12+
- Git (optional)

### 1. Clone the repository
```bash
git clone <repository-url>
cd RIMES
```

### 2. Install dependencies
```bash
pip install -r backend/requirements.txt
```

### 3. Initialize the database (SQLite)
```bash
# Creates rime_data.db and populates it with the French word list
python backend/migrate_data.py
```

### 4. Start the API server
```bash
uvicorn backend.main:app --host 0.0.0.0 --port 8080
# Or using the helper script:
python run_local.py --mode server
```

### 5. Test the API
```bash
# Check if server is running
curl http://localhost:8080/
# {"message": "Welcome to RIMES Rhyme API (Local)", "version": "1.0.0"}

# Check a word for rhyme patterns
curl -X POST http://localhost:8080/rime/check \
  -H "Content-Type: application/json" \
  -d '{"word": "FRAME"}'
# {"matches": [{"pattern": "tG=F", "count": 35, "matched_word": "FRAME"}]}

# Add a new word
curl -X POST http://localhost:8080/rime/add \
  -H "Content-Type: application/json" \
  -d '{"word": "FRANZ"}'
# {"message": "Word 'FRANZ' added successfully"}

# Get statistics
curl http://localhost:8080/rime/stats
```

## Running with Docker

### 1. Start all services
```bash
docker-compose up --build
```

### 2. Wait for services to be healthy
- PostgreSQL healthcheck runs every 10s
- API will start once database is healthy
- Nginx proxy will be available on ports 80 and 443

### 3. Run the migration (populate PostgreSQL)
```bash
# Set the DATABASE_URL environment variable
set DATABASE_URL=postgresql://rimeuser:rimepass@localhost:5432/rime_data
python backend/migrate_postgres.py
```

### 4. Test the API
```bash
# Through Nginx proxy on port 80
curl http://localhost/
curl -X POST http://localhost/rime/check -H "Content-Type: application/json" -d '{"word": "FRAME"}'
```

### 5. Stop services
```bash
docker-compose down
```

## Development

### Run tests
```bash
pytest backend/tests/ -v
```

### Run with in-memory SQLite (for fast testing)
```bash
python run_local.py --mode test   # Runs the test suite
python run_local.py --mode migrate # Populates in-memory DB
python run_local.py --mode server  # Starts API server
```

## API Documentation

Once the server is running, visit:
- Swagger UI: `http://localhost:8080/docs`
- OpenAPI JSON: `http://localhost:8080/openapi.json`

## Project Structure

```
RIMES/
├── backend/                     # Python FastAPI application
│   ├── main.py                  # FastAPI app instance
│   ├── models.py                # SQLAlchemy ORM models
│   ├── database.py              # Database engine/session setup
│   ├── api/
│   │   ├── routes.py            # API endpoint definitions
│   │   └── __init__.py
│   ├── regex_converter.py       # Legacy wildcard → regex converter
│   ├── migrate_data.py          # SQLite migration script
│   ├── migrate_postgres.py      # PostgreSQL migration script
│   ├── simulate_migration.py    # In-memory SQLite simulation
│   ├── Dockerfile               # Multi-stage Docker build
│   └── requirements.txt         # Python dependencies
├── tests/                       # Pytest test suite
│   ├── conftest.py              # Test fixtures
│   └── test_api.py              # Unit tests
├── docker-compose.yml           # Full stack orchestration
├── nginx.conf                   # Nginx reverse proxy configuration
├── init-scripts/
│   └── 01-init-schema.sql       # PostgreSQL initialization script
├── .dockerignore                # Files to exclude from Docker builds
├── extracted_words.txt          # 260,352+ French words (source data)
└── README.md                    # This file
```


This API preserves the core functionality while providing a modern, accessible interface.

## License

MIT License - see the [LICENSE](LICENSE) file for details.

- MIT License allows free use, modification, and distribution