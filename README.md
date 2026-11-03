# SRM Worksheet Answer Generator — Guide

A local, Windows-first tool that turns your **manually downloaded** SRM worksheet files
(PDF/DOCX) into reviewed answer PDFs, uploads them to your own Google Drive, and verifies
link sharing — then stops. It never signs in to SRM for you, never fills the answer-link
field, and never clicks **UPDATE/SUBMIT**. Final submission stays a manual, human step.

## Highlights
- Folder-driven, multi-subject workflow (`input\<Subject>\Unit N\<file>.pdf`).
- Resumable and isolated: finished work is reused; one worksheet failing never aborts the batch.
- Token-aware pacing and HTTP 429 handling for free model tiers (Gemini / Groq / OpenAI-compatible).
- Targeted recovery for failed worksheets, with automatic manifest backups.
- No SRM automation, no auto-submission, secrets never printed.

## Start here
See **[how-to-use.md](how-to-use.md)** for full setup and usage: installation, `.env`
configuration, the normal workflow, command reference, recovery, troubleshooting, and
safety notes.

## Safety
Never commit or share `.env`, OAuth `credentials.json`, Drive tokens, browser profiles, or
generated answer PDFs. This repository contains documentation only — no secrets, no code.
