# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a personal notes repository for APR (Automatic Place and Route) / IC backend physical design study material. It is not a software project — there is no build system, linter, or test suite. Treat it as a knowledge base plus a set of reference EDA scripts/checkpoints and slide decks.

## Structure

- `text_book/` — reference PDFs covering IC backend design topics: physical design flow overview, STA (Static Timing Analysis) basics, CTS (Clock Tree Synthesis), ICC-based backend design, and APR course materials. These are read-only reference documents, not something to edit.
- `note/` — the user's own Markdown study notes, one file per topic (e.g. `floorplan.md`, `placement.md`, `routing.md`, `STA.md`). Each theory-topic note traces its content back to a specific chapter/page range of a `text_book/` PDF (cited near the top of the file) and keeps tool-specific terms (floorplan, placement, routing, etc.) in English rather than translating them. Case-study notes (e.g. `Version4_DTMF_CHIP_Innovus_flow.md`) instead reconstruct what was actually done in a `reference_design/` Innovus checkpoint tree by decompressing its `.inn.dat/inn.cmd.gz` command history and `.metric.gz` QoR data — cite the reference-design path rather than a textbook chapter.
- `reference_design/gcd/` — a reference Cadence Innovus/Encounter Tcl APR flow (`scripts/gcd_soce.tcl`) for an example "gcd" design, plus its supporting `design_data/` (`.conf`, `.io`, `.view` files). A full floorplan → power planning → placement → CTS → routing → DFM → verification → export flow kept as a runnable-script reference, not something meant to execute inside this repo (there is no EDA tool installed here).
- `reference_design/180um/` — the actual Cadence Innovus 180nm practice project (`Version4/`, `Version5/`), checked in as raw checkpoints: each stage folder (`Floorplan/`, `Placement/`, `CTS/`, `Route/`) holds `NN_stage.inn` (a small Tcl restore script) paired with `NN_stage.inn.dat/` (the saved design database — binary/gzipped state, embedded lib/lef/mmmc files, and a gzipped `inn.cmd.gz` command-history log + `DTMF_CHIP.metric.gz` QoR log). This is the source material `note/Version4_DTMF_CHIP_Innovus_flow.md` was derived from. These directories are large and mostly binary — don't `grep`/`cat`/recursively `find` through them; extract what you need with `gzip -dc <file>.inn.dat/inn.cmd.gz` (command history) or `.../DTMF_CHIP.metric.gz` (QoR metrics) instead.
- `slide/` — Marp Markdown slide decks. Convention: **one subfolder per slide deck**, containing the deck's Markdown file plus an `image/` subfolder holding the images that deck references.

## Working in this repository

- Since there is no code to build, lint, or test, focus on organizing, summarizing, or cross-referencing notes when asked.
- Do not modify the PDF files in `text_book/` or the checkpoint files under `reference_design/180um/`.
- When adding a new topic note under `note/`, follow the existing files' convention: cite the source (textbook chapter/page or reference-design path) near the top, and keep EDA-specific terms in English.
- When adding a new slide deck under `slide/`, create it as `slide/<deck-name>/<deck-name>.md` (Marp Markdown) with images placed in `slide/<deck-name>/image/` and referenced via relative paths from there.
