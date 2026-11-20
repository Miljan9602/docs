# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Mintlify documentation site ("Saphyre Docs") for two products on Sei Network (chain ID `1329`):
- **RFQ** — the Nima RFQ (Request for Quote) protocol: coordinator API at `https://rfq.saphyre.xyz`, client (frontend) integration, solver integration (HTTP + MessagePack WSS).
- **DEX** — Dragonswap: contract addresses/ABIs and the read-only DEX API at `https://sei-api.dragonswap.app`.

There is no application code — only MDX pages, `docs.json`, and static JSON/YAML assets.

## Commands

Yarn is the package manager (`yarn.lock`, `packageManager` field), despite the README saying `npm install`.

```bash
yarn install
yarn dev            # mintlify dev → http://localhost:3000
yarn validate       # mintlify validate — run after any change
yarn check-links    # mintlify broken-links — run after adding/moving/renaming pages
```

The `objc ... GNotificationCenterDelegate` warning from sharp on macOS is noise; check for `success` at the end.

## Structure and how pieces connect

- `docs.json` is the single source of navigation. Two `products` (DEX, RFQ), each with tabs → groups → page paths (no `.mdx` extension). A new page is invisible until it is added here.
- DEX is the default product. Mintlify has no per-product `default` flag; the product shown at `/` is whichever one contains the `index` page in its nav, so `index` lives under DEX and DEX is listed first.
- API reference pages use Mintlify's `api:` frontmatter for the interactive playground, not OpenAPI:
  - Relative paths (`api: "GET /api/v1/quote"`) resolve against `api.baseUrl` in `docs.json` (the RFQ coordinator). Used under `frontend/api/`.
  - Solver pages (`solver/api/`) use an absolute placeholder host (`https://your-solver.example.com`) because they document endpoints solvers implement.
  - DEX pages (`contracts/dex-api/`) use absolute `https://sei-api.dragonswap.app/...` URLs. One endpoint per page.
- `api-reference/openapi.yaml` is not wired into `docs.json` and does not cover all endpoints (e.g. no analytics). Don't treat it as authoritative.
- Branding mirrors saphyre.xyz's CSS tokens: brand `#155DFC`, dark background `#0A0A0A`, links `#60A5FA`, font Supreme. Fonts are self-hosted in `fonts/` as woff2 because Mintlify only accepts woff/woff2 (the site ships TTF). `images/logo-dark.svg` (white wordmark) is the dark-mode logo; `logo-light.svg` is the same mark with dark text.
- `style.css` (auto-loaded by Mintlify) recolors HTTP method badges, path-param pills and the "Try it" button to brand blue for every method; it relies on Mintlify's `method-pill`, `method-nav-pill` and `tryit-button` hooks.
- `concepts/` holds protocol-level explanations (architecture, EIP-712 signing, Permit2, fees, TTLs, error codes) shared by both RFQ integration tabs.

## AI skill pages (keep in sync)

`skills/` contains pairs:
- `<name>-SKILL.md` — raw Claude Code skill file (YAML `name`/`description` frontmatter). Users install it via `curl` from `https://raw.githubusercontent.com/Miljan9602/docs/main/skills/<name>-SKILL.md`.
- `<name>.mdx` — rendered docs page: Mintlify frontmatter + an `<Info>` install block, then the skill body.

For `client-integration` and `solver-integration`, the body after `</Info>` is identical to the `-SKILL.md` body — apply every edit to both files. `dex-api-integration.mdx` is a richer Mintlify-component version of a condensed `dex-api-integration-SKILL.md`; edit both, but they are not identical.

When an RFQ API behavior changes, the established pattern (see git log) is to update the API page, any affected guide (`*/integration-guide.mdx`, `solver/websocket-guide.mdx`), and both skill files in the same commit.

## Gotchas

- Raw GitHub URLs to `skills/*-SKILL.md` and `contracts/abi/**/*.json` are public install/download links. Renaming or moving those files breaks external users.
- In MDX prose and headings, escape literal braces: `/api/v1/pools/\{address\}`. Plain `-SKILL.md` files must not escape them.
- Constants such as the Permit2 address, chain ID, and quote TTLs are repeated across many pages; grep for every occurrence when changing one.

## Commits

Conventional commits with a docs scope, e.g. `docs(skills): ...`, `docs(solver): ...`.
