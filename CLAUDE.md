# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Conversational search application using **Vertex AI Agent Builder** (Discovery Engine API) for RAG-based generative answers grounded in document data. FastAPI backend + Streamlit frontend, deployed on GCP with multi-regional Cloud Run services behind a global load balancer with IAP.

## Essential Commands

```bash
# Install dependencies
poetry install

# Run backend locally (port 8888)
poetry run uvicorn main:app --app-dir src/answer_app --reload --host localhost --port 8888

# Run frontend locally
poetry run streamlit run src/client/streamlit_app.py

# Generate local OAuth secrets configuration
poetry run write_secrets

# Run all tests
poetry run pytest

# Run a single test file
poetry run pytest tests/test_main.py

# Run a single test function
poetry run pytest tests/test_main.py::test_answer_success -v

# Run tests with coverage
poetry run coverage run -m pytest && poetry run coverage report -m

# Deployment (bootstrap + deploy)
source ./scripts/install.sh

# Configure environment variables
source ./scripts/set_variables.sh
```

## Architecture

### Data Flow
1. User authenticates via Google OAuth in Streamlit frontend (`src/client/`)
2. Frontend sends queries to FastAPI backend `/answer` endpoint (`src/answer_app/main.py`)
3. Backend calls Vertex AI Discovery Engine via `DiscoveryEngineHandler` for grounded responses
4. Answer text + citations converted to base64-encoded markdown (`_answer_to_markdown` in `utils.py`)
5. Conversations and feedback logged to BigQuery asynchronously

### Key Backend Classes
- **`UtilHandler`** (`src/answer_app/utils.py`): Orchestrates config loading, BigQuery client, and `DiscoveryEngineHandler`. Instantiated as a **module-level singleton** (`utils = UtilHandler(...)` at bottom of file) — this means `google.auth.default()` and config loading happen at import time.
- **`DiscoveryEngineHandler`** (`src/answer_app/discoveryengine_utils.py`): Wraps `ConversationalSearchServiceAsyncClient` for answer queries, session management. Configured with location, engine ID, and preamble from `config.yaml`.
- **Pydantic models** (`src/answer_app/model.py`): Request/response types including `ClientCitation` which handles inline markdown citation links.

### API Endpoints (FastAPI)
- `POST /answer` — Main query endpoint, returns `AnswerResponse` with markdown + citations
- `POST /feedback` — Logs user thumbs-up/down feedback to BigQuery
- `GET /sessions/` — Lists user sessions
- `GET /healthz` — Health check
- `GET /get-env-variable` — Returns environment variable value (for Cloud Run)

### Infrastructure
- **Terraform** in `terraform/` — bootstrap (APIs, service accounts) → main (Cloud Run, LB, IAP)
- **Multi-regional**: Cloud Run across 3 regions (us-central1, us-west1, us-east4) with global LB
- **Config**: `src/answer_app/config.yaml` controls app name, regions, Discovery Engine IDs, BigQuery tables, LB domain, and the LLM preamble prompt

## Testing Patterns

Tests require **zero external dependencies** (no GCP credentials needed). This is achieved through a critical pattern in `tests/conftest.py`:

- **`pytest_configure` hook** patches `google.auth.default()` across all modules *before test collection starts* — this prevents the module-level `UtilHandler` singleton from calling real GCP auth during import.
- Global patches also cover `bigquery.Client`, `ConversationalSearchServiceAsyncClient`, and module-level singleton instances (`answer_app.utils.utils`, `client.utils.utils`).
- Individual test fixtures then mock specific methods on the `UtilHandler` instances (e.g., `mock_util_handler_methods` patches `answer_app.main.utils`).
- Async endpoints tested with `@pytest.mark.asyncio` and `pytest-httpx` for HTTP mocking.

When adding new modules that call `google.auth.default()` at import time, add a corresponding patch target to the `auth_patches` list in `pytest_configure`.

## Commit Conventions

This project uses **python-semantic-release** for automated versioning. Commits must follow [Conventional Commits](https://www.conventionalcommits.org/):

- `feat:` → minor version bump
- `fix:`, `perf:` → patch version bump
- Other allowed prefixes: `build`, `chore`, `ci`, `docs`, `style`, `refactor`, `test`
- Merges to `main` trigger the GitHub Actions release workflow (tag + GitHub release, no commit back to repo)

Version in `pyproject.toml` is always `0.0.0` — actual version derived dynamically from Git tags via `poetry-dynamic-versioning`.

## Required Environment Variables

- `PROJECT` — Google Cloud project ID
- `REGION` — Default compute region
- `TF_VAR_terraform_service_account` — Terraform service account
- `TF_VAR_docker_image` — Docker image for deployment
- `LOG_LEVEL` — (optional) Backend log level, defaults to INFO

## Documentation

Detailed docs are in `docs/` organized by topic: `installation/`, `development/`, `infrastructure/`, `troubleshooting/`. See `README.md` for the full index with links.
