# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Kanban board for a fictitious bank's IT PMO (internal demo/training tool). The whole app is `index.html` (markup, `<style>`, `<script>`). There is no build, bundler, package manager, linter or test suite; run it by opening the file in a browser (`open index.html`).

## Hard constraints (from the original brief — do not break)

- Vanilla HTML/CSS/JS only: no frameworks, CDNs, web fonts or image files. System font stack, Unicode glyphs/inline SVG for icons.
- No persistence: no `localStorage`, `sessionStorage`, IndexedDB or cookies. A refresh must reset the board to seed data (the UI says so).
- The only network call is the FormSubmit AJAX endpoint (`FORMSUBMIT_ENDPOINT`). The user's email appears only in that constant.
- No `!important`, no `alert()`/`confirm()` (delete confirmation and validation errors are inline).
- No real bank branding: neutral "IT PMO" wordmark, blue palette. Task IDs are `UOB-ITPM-####` and the email subject prefix is `[UOB IT PMO]` by explicit spec; don't add logos or imitate official systems.

Quick check after edits: `grep -nEi "localStorage|sessionStorage|indexedDB|document\.cookie|https?://|!important" index.html` should only hit the FormSubmit URL.

## Architecture

Everything flows from `state = { tasks: [], filters: {...} }` plus a small `ui` object for transient flags (`pendingDelete`, `moveOpen`, `draggingId`). Actions (`addTask`, `moveTask`, `deleteTask`, filter handlers) mutate `state`/`ui` and then call `renderBoard()`, which rebuilds the four columns and the header summary via `innerHTML`. Card DOM is produced only by `renderCard()`; never edit card elements directly.

Things that are easy to get wrong:

- **Event delegation**: because the board is re-rendered wholesale, listeners live on `#board` (click via `data-action` attributes, plus drag events). Add new card controls as `data-action` buttons handled in `onBoardClick`, not per-element listeners.
- **Drag and drop**: `onDragStart` only toggles a CSS class; re-rendering during `dragstart` cancels the drag. The re-render happens in `drop`/`dragend`.
- **Escaping**: every user-supplied string must go through `escapeHtml()` before entering an HTML string.
- **Filters vs. summary**: column counts respect filters; the header summary (total, per-status, overdue) always reflects all tasks.
- **Dates**: use the local-date helpers (`todayISO`, `toISODate`), not `toISOString()`, to avoid UTC off-by-one. Seed due dates are relative to today so the demo never goes stale.
- **Submit flow** (`handleSubmit`): validate → `addTask` (optimistic render) → close modal + success toast → `notifyNewTask` in try/catch; a failure only shows a warning toast. The board must never break on a FormSubmit failure.
- **FormSubmit activation**: the first submission to a new address triggers a confirmation email; delivery only works after the link is clicked. Placeholder `YOUR_EMAIL@example.com` is expected to fail.
