# IT PMO – Project Kanban

A single-file Kanban board for a fictitious bank's IT PMO, used as an internal demo and training tool. It is written in vanilla HTML, CSS and JavaScript, with no build step and no dependencies.

**Live site:** https://chiam168.github.io/kanban/

## Run locally

Open `index.html` in a browser.

## Notes

- Nothing is saved: refreshing the page resets the board to the seed data.
- New-task email notifications use FormSubmit. Set `FORMSUBMIT_ENDPOINT` in `index.html` to turn them on. To keep your address out of the public source, use FormSubmit's random-string alias instead of your email.
- Every push to `main` runs the constraint and secret checks, then deploys to GitHub Pages (`.github/workflows/pages.yml`).
