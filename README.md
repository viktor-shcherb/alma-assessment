# PDF Form Filling Automation

This repository coordinates the MVP automated form-filling experience.

## Project Overview
- Goal: let users upload supporting PDFs, capture a target form URL, and push extracted data to a filling service.
- Scope: only the minimum happy path; error states, auth, and analytics are deferred.

## Repository Layout
```
frontend/  -> Submodule containing the React app (pdf-form-filling-app)
backend/   -> Submodule hosting the form-filling service (pdf-form-filling-service)
```

Each submodule defines its own dependencies and deployment defaults. This root repo focuses on orchestration and shared documentation.

## Environment Variables
The repo ships with `.env` files containing safe defaults for local development. Update them as needed before running either service.

| Service         | File                                    | Variables                                                                                                                                                               |
|-----------------|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Backend service | `backend/pdf-form-filling-service/.env` | `S3_BUCKET_NAME`, `S3_BUCKET_REGION`, `S3_BUCKET_URL` (public base URL), `OPENAI_API_KEY`. AWS credentials come from the standard SDK chain (env vars, profiles, etc.). |
| Frontend app    | `frontend/pdf-form-filling-app/.env`    | `VITE_API_BASE_URL` – base URL for the backend HTTP API.                                                                                                                |

## Getting Started
1. `git submodule update --init --recursive`
2. Adjust the `.env` files above if your local URLs/buckets differ.
3. Follow setup instructions inside each submodule (`frontend/pdf-form-filling-app/README.md`, `backend/pdf-form-filling-service/README.md`).

## (!) Divergence between implementation and the task requirements
I noticed late in the development that the task required filling web forms, not PDFs. It does not change the overall approach, but PyMuPDF does need to be swapped out for a Puppeteer/Selenium-based solution.

The general workflow stays the same:
- User uploads documents to the app.
- Structured information gets extracted from these documents.
- This structured data is then used to fill in the form (PDF in this case).