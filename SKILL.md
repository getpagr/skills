---
name: pagr
description: Publish, update, list, delete, or manage HTML files with the Pagr CLI or remote MCP server. Use when publishing HTML reports, dashboards, documents, or planning artifacts, or configuring Pagr in a project.
---

# Pagr CLI Skill

Use this skill when the user wants to publish, update, list, delete, or manage HTML files with the Pagr CLI or through the remote MCP server.

## When To Use This Skill

- Publishing generated HTML reports, dashboards, documents, or plans.
- Updating existing published Pagr pages while keeping the same public URL.
- Creating shareable URLs for local HTML artifacts.
- Listing or deleting Pagr files from a user account.
- Setting up Pagr in a project for agent workflows.
- Publishing HTML artifacts from CI.
- Using the remote MCP server to publish HTML reports.

## Core Rules

- Use `pagr upload <file>` for HTML files.
- Never invent a Pagr URL. Always return the URL printed by the CLI.
- If a file contains `<meta name="pagr-id">` or `<meta name="pagr-slug">`, `pagr upload <file>` may update the existing page instead of creating a new one.
- Updates replace the current HTML in place. Pagr does not store previous HTML versions or rollback history.
- Prefer `PAGR_API_KEY` in CI instead of writing `~/.pagr/config.json`.
- Do not print or expose full API keys unless the user explicitly asks to configure authentication.
- If upload, update, list, or delete fails, surface the exact CLI/API error.

## Remote MCP Server (SSE)

You can connect to Pagr's remote Model Context Protocol (MCP) server over SSE (Server-Sent Events) to allow any MCP-compatible agent to publish files directly.

The remote MCP server endpoint is:
```text
https://mcp.getpagr.co/mcp?PAGR_API_KEY=pagr_tW2NdRvtqZq2CIb0oJ137rfh75ystZoeoOQiSd7aMDc
```

When using an MCP-compatible client (like Claude.ai Web App Custom Connectors, Cursor IDE, or others), configure the Server URL pointing to the endpoint above. This allows the client to call tools like `upload_file` and publish pages seamlessly.

## Installation

```bash
npm install -g @getpagr/cli
```

Pagr CLI requires Node.js 20 or newer. The installed command is `pagr`.

## Authentication

Browser login:

```bash
pagr login
```

`pagr login` starts a browser login flow, authenticates with Pagr, receives a CLI API key through a local callback, and saves it to `~/.pagr/config.json`.

Manual login for SSH, headless, or locked-down environments:

```bash
pagr login --manual
```

Save a key without prompting:

```bash
pagr login --key pagr_...
```

Set a key directly in local config:

```bash
pagr config set-key pagr_...
```

API keys must start with `pagr_`.

Config location:

```text
~/.pagr/config.json
```

Auth lookup order:

1. `PAGR_API_KEY`
2. `~/.pagr/config.json`
3. Error if neither exists

## Upload New HTML

```bash
pagr upload report.html
```

Behavior:

- Reads the local HTML file as UTF-8.
- Uploads raw HTML to Pagr.
- Prints the public URL.
- Copies the URL to the clipboard when supported.
- When `CI` is set, also prints:

```text
PAGR_SLUG=...
PAGR_URL=...
```

## Upload With Title And Slug

Set a page title:

```bash
pagr upload report.html --title "Weekly report"
pagr upload report.html -t "Weekly report"
```

Request a stable readable slug:

```bash
pagr upload report.html --slug weekly-report
pagr upload report.html -s weekly-report
```

Use both:

```bash
pagr upload report.html --slug weekly-report --title "Weekly report"
```

`--title` sends `X-Pagr-Title`. `--slug` requests a stable readable URL. Custom slugs can be plan-gated.

## Update Existing Pages

```bash
pagr upload report.html --update --slug weekly-report
pagr upload report.html -u -s weekly-report
```

Behavior:

- Checks slug ownership first.
- Fails if the slug is missing or not owned by the authenticated user.
- Sends replacement HTML to the existing page.
- Keeps the public URL stable.
- Replaces current content in place; this is not versioning.

## Auto-Update With Meta Tags

```html
<meta name="pagr-id" content="...">
<meta name="pagr-slug" content="weekly-report">
```

Update priority:

1. `pagr-id` updates by stable document ID.
2. `pagr-slug` updates by slug.
3. `--update --slug` forces update by slug.
4. Otherwise a new file is uploaded.

Inject Pagr metadata into the source file:

```bash
pagr upload report.html --inject
```

`--inject` writes Pagr metadata into the local HTML source after upload. If a new upload has no `pagr-slug`, the CLI auto-injects metadata for future updates. This modifies the local source HTML file.

## List Files

```bash
pagr ls
pagr ls --limit 20
```

Default limit is 50. Output includes slug, views, plan, title, and URL. If no files exist, the CLI prints `No files found.`

## Delete Files

```bash
pagr rm weekly-report
pagr rm weekly-report --yes
pagr rm weekly-report -y
```

Without `--yes`, the CLI asks for confirmation and accepts `y` or `yes`. Delete is not reversible.

## Config Commands

```bash
pagr config show
pagr config set-key pagr_...
pagr config set-url https://pagr.link
```

`pagr config show` masks the API key. `set-key` requires a `pagr_` prefix. `set-url` overrides the Worker API URL in local config. Environment variables override local config.

## Init Command

```bash
pagr init
pagr init --no-claude
pagr init --no-mcp
```

`pagr init` writes:

- `SKILL.md`
- `.claude/skills/pagr/SKILL.md` unless `--no-claude`
- `.agents/skills/pagr/SKILL.md`
- `.claude/settings.json` MCP server config unless `--no-mcp`

MCP config:

```json
{
  "mcpServers": {
    "pagr": {
      "command": "npx",
      "args": ["-y", "@getpagr/mcp"]
    }
  }
}
```

## CI Usage

Use `PAGR_API_KEY` in CI. Do not write `~/.pagr/config.json` in CI.

```yaml
- name: Publish HTML report
  run: pagr upload output/report.html --title "Nightly report"
  env:
    PAGR_API_KEY: ${{ secrets.PAGR_API_KEY }}
```

When `CI` is set, use the machine-readable `PAGR_SLUG` and `PAGR_URL` output lines.

## Environment Variables

- `PAGR_API_KEY`: overrides the locally saved key.
- `PAGR_WORKER_URL`: overrides the Worker API URL. Default is `https://pagr.link`.
- `PAGR_APP_URL`: overrides the app URL used by `pagr login`. Default is `https://app.getpagr.co`.
- `CI`: makes upload print machine-readable `PAGR_SLUG` and `PAGR_URL`.

## Error Handling

Common failures:

- Missing API key: run `pagr login`, run `pagr login --manual`, or set `PAGR_API_KEY`.
- Invalid API key format: keys must start with `pagr_`.
- Cannot read file: confirm the path exists and points to an HTML file.
- Slug not found or not owned during update: upload without `--update` or use the correct slug.
- Upload/update/list/delete API errors: surface the exact CLI error.
- Clipboard failure does not fail upload; the URL is still printed.

If browser login fails, use `pagr login --manual`. If upload fails due to plan or file limit, surface the CLI error and tell the user to check account plan and file size.

## Agent Workflow

1. Confirm the target file exists and is HTML.
2. If the user wants a stable URL, use `--slug`.
3. If the file already has `pagr-id` or `pagr-slug`, use plain `pagr upload <file>` unless the user asks for a new page.
4. If the user explicitly wants to replace an existing page, use:

```bash
pagr upload <file> --update --slug <slug>
```

5. Return the URL printed by the CLI.
6. In CI, capture `PAGR_URL`.
