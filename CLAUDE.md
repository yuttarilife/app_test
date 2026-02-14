# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository is a knowledge base for summarizing and organizing learning content. It is not a codebase; it uses AI agent skills to structure and accumulate content.

## Plan Creation

- When creating a markdown file in the `./plans` directory, use a descriptive kebab-case filename that reflects the plan content (e.g., `ai-news-catchup-setup.md`, `skill-review-and-fix.md`). Do NOT use random or auto-generated names.
- Make sure to have it reviewed by Codex using the codex-review skill.

## ExecPlans

When writing complex features or significant refactors, use an ExecPlan (as described in `.agent/PLANS.md`) from design to implementation.

## Review Gate (codex-review)

At key milestones—after updating specs/plans, after major implementation steps (≥5 files / public API / infra-config), and before commit/PR/release—run the codex-review SKILL and iterate review → fix → re-review until clean.

## Task Management

When implementing features or making code changes, use the Tasks feature to manage and track progress. Break down the work into clear steps and update task status as you proceed.

## Other

When asking for a decision, use "AskUserQuestion".
