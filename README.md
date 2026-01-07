# Alma Assessment Platform

This repository coordinates the MVP form-filling experience.

## Project Overview
- Goal: let users upload supporting PDFs, capture a target form URL, and push extracted data to a filling service.
- Scope: only the minimum happy path; error states, auth, and analytics are deferred.
- Missing pieces: concrete API contract, hosting details, and testing strategy. These will be filled in as they are defined.
- Data handling (current): every visitor gets a generated `user_id` stored in a cookie; uploads land in public S3 under `user_id/<original_file>/value.ext`, and the structured JSON produced for that file is written to `user_id/<original_file>/info.json`. During the S3 upload stage, the same bytes are forwarded to OpenAI’s Files API so the extraction step can reference them without re-uploading.
- UX expectations:
  - Frontend issues one concurrent upload request per file, bundling form link + `user_id` metadata, and renders backend-provided status updates as each completes.
  - Users can delete uploaded assets or follow S3 links to preview them directly from the UI.
  - After all uploads and the target form link are provided, users trigger the fill process, which downloads the PDF, stores it in S3, invokes the filling engine, and finally exposes a filled-PDF link from S3.

## Repository Layout
```
frontend/  -> Submodule containing the React app (pdf-form-filling-app)
backend/   -> Submodule hosting the form-filling service (pdf-form-filling-service)
```

Each submodule defines its own dependencies and deployment defaults. This root repo focuses on orchestration and shared documentation.

## Getting Started
1. `git submodule update --init --recursive`
2. Follow setup instructions inside each submodule (`frontend/pdf-form-filling-app/README.md`, `backend/pdf-form-filling-service/README.md`).
3. Placeholder: consolidated dev tooling (TBD once requirements stabilize).

## Deployment (Placeholder)
- Root repo: will eventually provide a minimal compose/shell deployment wrapper.
- Frontend: expected to export a static build; hosting target not yet selected.
- Backend: intended to run as a single container or process with public S3 access and OpenAI credentials for the upload/extraction flow.

## Documentation Status
- `AGENTS.md`: development guardrails for this MVP.
- `docs/design.md`: evolving design overview (now includes OpenAI upload touchpoints via the Files API only, plus upload/delete/status UX and the PDF filling pipeline).
- Open questions: storage bucket naming, security requirements, OpenAI model selection.

> NOTE: Replace every `TBD` above when the corresponding decision is made.
