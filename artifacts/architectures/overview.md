# Project Architecture Overview

## Executive Summary

The project is a QR Code Generator prototype built with Python and FastAPI. It serves as an API backend to create shortened URLs (tokens) mapped to original long URLs, and dynamically generates QR codes pointing to those shortened URLs. When a user scans the QR code or clicks the short link, the system tracks the scan event and redirects the user to the original destination.

Currently structured as a guided exercise, the codebase provides a solid boilerplate with clear separation of concerns using SQLAlchemy for ORM and Pydantic for data validation. Certain core logic functions—such as token generation, URL validation, and the redirect flow—are marked as `TODO` for developers to implement.

## File Structure

```text
/
├── pyproject.toml         # Project metadata and dependencies
├── README.md              # Documentation and exercise instructions
├── PROMPT.md              # Design challenge prompt and requirements
└── app/
    ├── main.py            # FastAPI application entry point
    ├── database.py        # SQLAlchemy database setup and session management
    ├── models.py          # ORM models (UrlMapping, ScanEvent)
    ├── schemas.py         # Pydantic validation schemas
    ├── routes.py          # API endpoints and routing logic
    ├── token_gen.py       # Token generation utility (TODOs inside)
    └── url_validator.py   # URL validation and normalization (TODOs inside)
```

## Module Functionalities

### `app/`
- **`main.py`**: The entry point for the FastAPI application. It initializes the app, registers the API router, and ensures that the database schema is created via SQLAlchemy.
- **`database.py`**: Configures the SQLite database connection using SQLAlchemy. It provides a `SessionLocal` factory and a dependency (`get_db`) to manage database sessions per request.
- **`models.py`**: Defines the data layer using SQLAlchemy declarative models. It includes `UrlMapping` for storing URL tokens and `ScanEvent` for tracking analytics.
- **`schemas.py`**: Contains Pydantic models (e.g., `CreateRequest`, `QRInfoResponse`) that define the exact shape and validation rules for the API's incoming and outgoing payloads.
- **`routes.py`**: Houses all the API endpoints (`/api/qr/*` and the redirect `/r/{token}`). It orchestrates the flow between input validation, business logic execution, database persistence, and an in-memory cache. 
- **`token_gen.py`**: A utility module designed to generate short, URL-safe Base62 tokens. It defines the logic structure, including handling hash collisions (currently a `TODO`).
- **`url_validator.py`**: A utility module responsible for sanitizing, normalizing, and verifying URLs against a blocklist to prevent abuse (currently a `TODO`).

## Architecture & Data Flow

1. **QR/Short Link Creation**:
   - The user submits a POST request to `/api/qr/create` with a long URL.
   - The input is validated by Pydantic (`schemas.py`) and sanitized/checked against a blocklist (`url_validator.py`).
   - A unique Base62 token is generated (`token_gen.py`), and the mapping is persisted to the database (`models.py`).
   - An in-memory cache is warmed with the token-to-URL mapping, and the generated short link and QR image URLs are returned.

2. **Redirection & Tracking** (Hot Path):
   - A user accesses the short link via `/r/{token}`.
   - The system uses a **cache-first strategy**: it checks the in-memory cache. If missed, it queries the database.
   - If the token is valid, a `ScanEvent` is recorded in the database, and the user is redirected (302) to the original URL.
   - Expired or deleted links return a 410 Gone; non-existent links return 404 Not Found.

3. **QR Code Image Retrieval**:
   - A GET request to `/api/qr/{token}/image` dynamically generates a PNG image of the QR code corresponding to the short link using the `qrcode` library, returning it as a streamed response.

## Dependencies & Tech Stack

- **Core Framework**: FastAPI, Uvicorn
- **Data Layer**: SQLAlchemy (ORM), SQLite (Database)
- **Data Validation**: Pydantic
- **Image Processing**: qrcode[pil]
- **Project Management**: uv

## Architectural Standards & Patterns

- **Strict Separation of Concerns**: The application strongly enforces boundaries. Routing (`routes.py`), data validation (`schemas.py`), database ORM (`models.py`), and business logic (`token_gen.py`, `url_validator.py`) are strictly decoupled.
- **Performance Optimization**: A cache-first read strategy is defined for the redirection endpoint to minimize database load on the hottest path.
- **Fail-Fast Input Validation**: All incoming requests are validated by Pydantic schemas before reaching the endpoint logic, ensuring data integrity early in the lifecycle.
