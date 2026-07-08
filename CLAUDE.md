# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Protobuf API contracts for SpendSense — the source of truth for the API contract shared between `SpendSense-backend` and `SpendSense-web`. Published to the Buf Schema Registry (`buf.build/xpendsense/spendsense`). See `README.md` for structure, publishing, and consumption details.

## Git workflow

`main` is production. Never commit or push directly to `main`.

- Branch work off `develop` (or commit straight to `develop` for small fixes)
- Push to `origin/develop`
- Open a PR from `develop` into `main`
- Enable auto-merge on the PR (`mergeMethod: MERGE`) so it lands once required checks pass

Phase 3 of a proto change ends with the push to GitHub that triggers BSR publishing — see the workspace root `CLAUDE.md` for the full multi-repo macro-phase workflow.
