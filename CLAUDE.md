# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A pregnancy monitor ("Gestação do Davi") for logging blood pressure/heart rate and weight. It is a mobile-first web app (max width 430px, iOS home-screen meta tags). The UI text is Brazilian Portuguese (`pt-BR`), so keep new strings, dates (`toLocaleDateString('pt-BR')`) and code comments in Portuguese.

## Structure and commands

The whole app is one file, [index.html](index.html), with inline CSS and JS. There is no build step, package manager, linter or test suite. To run it locally, serve the directory statically, e.g. `python3 -m http.server 8000`, then open `http://localhost:8000`. The only external dependency is Chart.js 4.4.1, loaded from cdnjs.

The code is written in a dense, minified-looking style: short identifiers, one-line functions and HTML built from template strings. Match that style when editing.

## Architecture

**Backend: Supabase over raw REST.** The app does not use the supabase-js SDK. The `sbGet/sbInsert/sbUpdate/sbDelete` helpers call PostgREST (`/rest/v1/<table>`) with `fetch`, using a publishable key hardcoded at the top of the script. Tables:
- `registros`: blood-pressure readings `{id, ts, s, d, b, n}` (systolic, diastolic, bpm, note)
- `pesos`: weight entries `{id, ts, p, s, n}` (kg, gestational week or null, note)
- `sintomas`: symptom entries `{id, ts, t, i, n}` (symptom name, intensity 1=leve 2=moderada 3=forte, note)
- `config`: key/value rows `{id, valor}`, upserted with `Prefer: resolution=merge-duplicates`. The keys are `dpp` (due date `YYYY-MM-DD`) and `dpp_cal` (JSON `{ts, sem}` week calibration)

`ts` is epoch milliseconds, set by the client (`Date.now()`). `sbGet` always appends `order=ts.desc`, so `R` and `PW` are stored newest first. Charts re-sort ascending with `[...R].sort(...)`.

**State and rendering.** The global arrays `R` (readings) and `PW` (weights) are the in-memory source of truth. Every change updates Supabase, patches the local array, then calls a full re-render (`render()` / `renderPeso()`), which rebuilds the history lists with `innerHTML` and destroys and recreates the Chart.js instances (`wC, fC, cC, pwC, pfC`). Charts use `responsive:false`, and canvas width is set by hand from the parent's `clientWidth`, so a chart drawn in a hidden container gets width 0. This is why `renderPeso()` runs when that page is shown.

**Edit/delete state is tracked by array index** (`pedit`, `pdel`, `peditPeso`, `pdelPeso`), not by record `id`.

**Sync.** `fetchAll()` runs on load and every 30s. It replaces `R`/`PW` with the server data and re-renders, so multiple devices stay in sync. The header dot (`setSyncDot`) shows the status. Because of this polling, an open inline edit form can be re-rendered while the user is typing, and the index-based edit state can end up pointing at a different record.

**DPP (due date) card.** `dpp` and `dpp_cal` are written to both `localStorage` and the Supabase `config` table. `fetchAll` copies the server values into `localStorage`, and `renderDpp()` reads only from `localStorage`. The gestational week comes from the calibration if one exists; otherwise it is estimated as 40 weeks minus the days left.

**Navigation.** There are three "pages" (`#page-pressao`, `#page-peso`, `#page-sintomas`), switched by `goPage()` from the fixed bottom nav. The pressure page has three sub-tabs (`goTab`): Geral (single-metric chart, switched with `swMode` b/s/d), Por período (filter by time of day and date range) and Comparar (compare two date ranges). Time of day (`per()`) is manhã 5–12h, tarde 12–18h, noite otherwise.

**Color thresholds** for BP and BPM are in `col(v, m)`. Line charts get a gradient stroke from `mkGrad`/`mkPesoGrad`, colored by each point's value.
