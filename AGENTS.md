# AGENTS GUIDE

This MVP intentionally trims scope. When contributing, keep the following constraints in mind:

## Absolute Priorities
- **Minimal feature set**: Ship only the link + document upload flow on the frontend and the form-filling pipeline on the backend.
- **No speculative architecture**: Avoid abstractions or libraries that only pay off once we scale; prefer direct, readable code.
- **Tight feedback loop**: Before adding functionality, confirm it supports the MVP goal. If unsure, ask.

## Implementation Notes
- Frontend uses React + MUI; keep layouts simple (one screen, no theming system beyond the defaults).
- Backend accepts a form URL plus arbitrary JSON extracted from uploads; no schema validation is required yet.
- Generate a `user_id` cookie for every visitor/session and include it with uploads so backend + storage can key on it deterministically.
- Storage is public S3 for this MVP. Each document and its derived JSON must live under `user_id/<original_file>/value.ext` and `user_id/<original_file>/info.json`.
- When pushing files to S3, pipe the exact payload to OpenAI’s Files API so extraction can reference the file immediately (Uploads API is out of scope for now).
- Frontend must upload each file via its own request (allowing concurrent transfers), always include the `user_id`, attach the form link metadata once available, reflect backend completion status per file, and expose “Delete” + “View on S3” affordances for each asset.
- Once uploads + form URL exist, expose a “Start Form Fill” action that triggers the backend pipeline and surfaces the resulting filled-PDF S3 link when ready.
- Deployment scripts live at the root; submodules expose only default arguments (documented locally).

## Collaboration Checklist
1. Read `README.md` and `docs/design.md` before coding.
2. Document any decision that changes scope in this repo, even if submodules already describe it.
3. Defer CI/CD, analytics, and auth work until Product explicitly schedules them.
4. Highlight missing information directly in docs so future contributors know what needs definition.

This document should evolve, but any change should reinforce the “MVP first” mindset.
