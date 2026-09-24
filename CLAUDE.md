# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This is the suite-wide CLAUDE.md. Every repo lives inside this folder, so this file loads in every session regardless of which repo it starts in. General personal preferences (language, code style, tooling, testing, git) live in `~/.claude/CLAUDE.md` and are not repeated here.

## Current state
The design is settled in [README.md](README.md), which is the source of truth. Each child repo (`platform`, `platform-infra`, `budget`, `events`, `admin`) currently has only a README, CLAUDE.md, and .gitignore. None has code, tooling, or commands yet. Read the README before starting any work.

New undecided questions go in the README's Open questions section. Once a question is decided, record it in the relevant section and remove it from the list.

## Layout
- This folder is the `suite` repo: README, this file, and the local-only folders.
- Inside it, as separate git repos that the `suite` repo ignores:
    - `platform`: published npm packages (`runtime`, `build`, `web`) and Terraform modules. It has no AWS access.
    - `platform-infra`: deployed shared infrastructure.
    - One repo per app: `budget`, `events`, and `admin`.
- Each repo gets its own CLAUDE.md with its real commands.

## Session notes
At the start of a session, read `memory/<repo>.md` for the repo being worked on, or `memory/suite.md` for suite-level planning, if the file exists. Before stopping, update it with what's in progress, what's next, and any open threads. These are handoff notes, not a decision log: decisions go in the README or a repo's docs. `memory/` and `local-resources/` are not tracked in git, because the repos are public.

## Rules that span repos
- **Cost:** aim for $0 on free tiers until an app has a public audience. Flag anything that costs money (e.g. Secrets Manager, NAT gateways, paid Atlas tiers) before introducing it.
- **Boundaries:** apps never read another app's database, only its API. Handlers never touch raw Lambda events or AWS SDKs directly; they go through the `runtime` library and `ctx`. The `platform` repo never gets AWS credentials.
- **Deployment:** prod is always a manual promotion of an artifact that already deployed to dev, never a rebuild.
- **DNS:** the main `sabrinabea.com` zone (in the management account) also holds the user's email records. No CI or automation ever writes to it. Changes there go only in `platform-infra/management/`, which the user applies by hand. Everything automated lives in the delegated `app.sabrinabea.com` (prod) and `dev.sabrinabea.com` (dev) zones.
- **Auth:** app code never sees Cognito directly. It only sees the `User` the runtime library builds from token claims.
