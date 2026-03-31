# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repo is a research workspace for planning the adoption of [OpenClaw](https://github.com/openclaw/openclaw) — a self-hosted personal AI agent gateway. It contains notes, specs, and setup guides rather than runnable code.

## What is OpenClaw?

OpenClaw runs as a persistent daemon that connects LLMs (Claude, OpenAI, local models) to messaging platforms (WhatsApp, Telegram, iMessage, Discord, etc.) and can automate tasks via skills, browser control, file management, and scheduled jobs. It installs via a CLI onboarding wizard and requires Node.js 22+.

## Research Structure

- `research/` — findings on hardware, installation, configuration, and use cases
  - `mac-mini-specs-2026-03-30.md` — Mac Mini spec comparison, recommendation, and session 1 conclusions
  - `getting-started-openclaw-2026-03-30.md` — OpenClaw install guide (older, manual steps)
  - `setup-guide-2026-03-30.md` — OpenClaw setup guide (newer, more complete)

## Key Decisions Made

- **Hardware target:** Mac Mini M4, 24GB RAM, 512GB SSD (~$999) — allows cloud API usage and local model experiments (8B–34B via Ollama)
