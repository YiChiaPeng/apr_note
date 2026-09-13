# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a personal notes repository for APR (Automatic Place and Route) / IC backend physical design study material. It is not a software project — there is no build system, linter, or test suite. Treat it as a knowledge base plus a small set of reference EDA scripts and slide decks.

## Structure

- `text_book/` — reference PDFs covering IC backend design topics: physical design flow overview, STA (Static Timing Analysis) basics, CTS (Clock Tree Synthesis), ICC-based backend design, and APR course materials. These are read-only reference documents, not something to edit.
- `note/` — the user's own Markdown study notes, one file per topic (e.g. `floorplan.md`, `placement.md`, `routing.md`, `STA.md`). Each theory-topic note traces its content back to a specific chapter/page range of a `text_book/` PDF (cited near the top of the file) and keeps tool-specific terms (floorplan, placement, routing, etc.) in English rather than translating them. Case-study notes (e.g. `Version4_DTMF_CHIP_Innovus_flow.md`) instead reconstruct what was actually done in an external Innovus practice/checkpoint folder by decompressing its `.inn.dat/inn.cmd.gz` command history and `.metric.gz` QoR data — cite the external source path rather than a textbook chapter.
- `APR/` — a reference Cadence Innovus/Encounter Tcl APR flow (`scripts/gcd_soce.tcl`) for an example "gcd" design, plus its supporting `design_data/` (`.conf`, `.io`, `.view` files). This is a full floorplan → power planning → placement → CTS → routing → DFM → verification → export flow kept as a runnable-script reference, not something meant to execute inside this repo (there is no EDA tool installed here).
- `slide/` — Marp Markdown slide decks. Convention: **one subfolder per slide deck**, containing the deck's Markdown file plus an `image/` subfolder holding the images that deck references.

## Working in this repository

- Since there is no code to build, lint, or test, focus on organizing, summarizing, or cross-referencing notes when asked.
- Do not modify the PDF files in `text_book/`.
- When adding a new topic note under `note/`, follow the existing files' convention: cite the source (textbook chapter/page or external practice-case path) near the top, and keep EDA-specific terms in English.
- When adding a new slide deck under `slide/`, create it as `slide/<deck-name>/<deck-name>.md` (Marp Markdown) with images placed in `slide/<deck-name>/image/` and referenced via relative paths from there.
