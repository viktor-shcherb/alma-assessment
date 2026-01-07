# MVP Design Overview

_Status: Draft – intentionally incomplete. Update as decisions are made._

## Goal
Enable a user to supply a PDF form URL and related documents so the system can extract structured data via OpenAI and pass it to a form-filling service.

## High-Level Flow
1. User opens the React web app.
2. App issues or reuses a `user_id` cookie that keys storage.
3. User pastes/provides a link to the publicly accessible target PDF form (hosted on S3 for now).
4. User selects supporting documents (PDFs or any format we plan to pass through to OpenAI).
5. Frontend spawns one upload request per file (concurrent) that includes the file payload + `user_id`, adds the form link metadata once the user has supplied it, and renders backend-reported status for each file.
6. Backend streams each file to S3 under `user_id/<file_slug>/value.ext` and simultaneously forwards the same bytes to OpenAI’s Files API so extraction can reference them immediately; it returns per-file status plus the S3 URL for UI “View”/“Delete” controls.
7. Backend invokes OpenAI to extract structured JSON and writes it to `user_id/<file_slug>/info.json` in S3.
8. After uploads complete and a form link exists, the user presses “Start Form Fill.” Backend downloads the blank form, stores it under the user’s namespace, and runs the form-filling workflow described below.
9. Backend stores the filled PDF in S3 and responds with a link so the UI can expose a download button.

## Components
- **Frontend (pdf-form-filling-app)**: React + MUI single page. Handles minimal validation (required fields, file size cap of 16MB), spawns concurrent uploads, surfaces status, and offers “View” as a link to S3, “Delete”, and “Start Form Fill” actions. Responsible for setting the `user_id` cookie.
- **Backend (pdf-form-filling-service)**: REST endpoint(s) that manage uploads, invoke OpenAI, call the form filler, and return status per file + final PDF links. Stateless HTTP service keyed by `user_id`.
- **Storage**: Public S3 bucket for MVP. No auth, encryption, or lifecycle rules defined yet; houses raw uploads, extracted JSON, blank forms, and filled PDFs.
- **Form Filling Engine**: External service/module (details pending) that accepts the PDF link + structured JSON and returns a completed PDF.
- **OpenAI API**: Files endpoint accepts document content so the extraction model can reference uploaded file IDs in prompts.

## Storage Layout
- Bucket prefix per visitor (supporting docs): `user_id/<original_file_name>/`.
- Raw upload object: `user_id/<original_file_name>/value.<ext>` (keep original extension).
- Extracted JSON: `user_id/<original_file_name>/info.json`.
- Blank forms: `user_id/forms/<form_slug>/source.pdf` (slug derived from URL/file name).
- Filled PDFs: `user_id/forms/<form_slug>/filled.pdf` (public link returned to frontend).
- Manifest: `user_id/manifest.json` manifest per user that lists uploaded files plus placeholder form metadata. Layout is `{ userId, updatedAt, files: [ { slug, objectKey, infoKey, fileName, s3Url, contentType, size, openaiFileId, uploadedAt, status } ], forms: {} }` so delete + pipeline steps (and the UI) can resolve assets deterministically.

## Frontend Upload UX (MVP)
- Fire one HTTP request per file to allow concurrent uploads; rely on `Promise.allSettled` to render progress bars and errors individually.
- Payload includes `userId` and the file, plus the form link when the user has already entered it. Backend responds with `{ status, slug, s3Url, openaiFileId, size }`.
- UI shows:
  - `View` button pointing at `s3Url`.
  - `Delete` button calling a backend delete endpoint that removes both `value.ext` + `info.json` and deletes the OpenAI file.
- On page load the frontend reads a `manifest_cache_<user_id>` cookie to hydrate the file list instantly and only hits `GET /api/uploads?userId=<cookie>` when the cache is missing or stale, reducing redundant S3 lookups. A “Refresh” action lets users bypass the cache and pull the newest manifest from the backend on demand.
- “Start Form Fill” button stays disabled until at least one upload succeeds and a form URL is present. Once triggered, disable uploads and show overall job status while polling for completion.

## OpenAI Integration
- During the upload phase, stream each file to both S3 and OpenAI’s Files API. Avoid writing temp files.
- Persist OpenAI file IDs alongside each document entry (location TBD) so later prompts/jobs can reference them without re-uploading.
- Structured extraction returns arbitrary JSON that we immediately store under the `info.json` path before calling the form-filling service.

## Form Filling Workflow
1. **Download & Persist Form**: Fetch the provided form URL, normalize the filename into `form_slug`, and store it as `user_id/forms/<form_slug>/source.pdf`.
2. **Context & Field Detection**:
   - Use PyMuPDF to read AcroForm metadata and extract field names, widget types, rectangles, and nearby text snippets.
   - Persist a lightweight schema describing each widget’s label, coordinates, surrounding widgets, and any derived context, but only keep entries for text widgets in this MVP.
3. **Value Selection**:
   - Prompt OpenAI (Responses/Assistants) with (a) the structured JSON extracted from supporting docs, (b) the current widget’s schema entry, and (c) a small window of surrounding widget metadata so it can decide to skip or populate the field; prompt responses stay in the form `{ fieldName, action: "skip" | "fill", value?, confidence }`.
   - Apply deterministic rules for normalization (dates, numbers) before filling any widget that OpenAI marks as `fill`.
4. **PDF Filling**:
   - Use PyMuPDF for all form editing, updating text widgets directly when AcroForm entries exist and drawing text overlays at recorded coordinates as needed.
   - Ignore non-text widgets for now; they remain blank unless re-scoped in a future iteration.
   - Store the filled artifact at `user_id/forms/<form_slug>/filled.pdf`.
5. **Status & Delivery**:
   - Update backend job status (`pending → filling → complete`).
   - Return the filled PDF S3 link to the frontend so the UI can swap in a “Download filled form” button.

## Interfaces (Draft)
- **Frontend → Backend**: `POST /api/forms` with payload `{ formUrl: string, documents: FileList }`. Backend responds with job/result identifiers (exact shape TBD).
- **Backend → OpenAI**: Text + file processing pipeline TBD (model, prompt, tokens, pricing not chosen).
- **Backend → Form Filling Service**: Likely HTTP call with `{ formUrl, payloadJson }`. Response contract undecided.

## Technical Considerations
- No authentication/authorization at MVP launch.
- Error handling limited to inline messaging; retry/backoff strategies not defined.
- Logging/monitoring unspecified; expect basic console logs only until productization.
- Deployment: root repo will expose a simplified script; service-specific defaults live in each submodule.

## Open Questions / TODO
- What constraints exist on document size, type, and number?
- Do we need virus scanning before storing files in S3?
- Which OpenAI model + prompt strategy produces reliable JSON for our forms and field mappings?
- Should the backend process uploads/filling synchronously or via background jobs?
- OpenAI file IDs now live in the manifest; where the eventual field maps belong (database vs. S3 manifest) is still undecided.
- Are per-file delete operations allowed after form filling begins?

Add answers inline as they are discovered.
