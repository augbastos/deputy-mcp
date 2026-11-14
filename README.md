# Deputy MCP

<!-- mcp-name: io.github.augbastos/deputy-mcp -->

[![CI](https://github.com/augbastos/deputy-mcp/actions/workflows/ci.yml/badge.svg)](https://github.com/augbastos/deputy-mcp/actions/workflows/ci.yml)
[![Security](https://github.com/augbastos/deputy-mcp/actions/workflows/security.yml/badge.svg)](https://github.com/augbastos/deputy-mcp/actions/workflows/security.yml)
[![Release](https://img.shields.io/github/v/release/augbastos/deputy-mcp)](https://github.com/augbastos/deputy-mcp/releases)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Ask Claude *"when do I work next?"* and get the answer from your
[Deputy](https://www.deputy.com) roster.

Deputy MCP is a local [MCP](https://modelcontextprotocol.io) server that lets AI agents
read your shifts, timesheets and team schedule, and, only if you turn it on, act on them.
It runs on your machine, connects to Deputy, and works with a regular
employee account.

## What you can ask

- "When do I work next?"
- "How many hours did I work last week?"
- "Who's on shift right now?" *(manager access)*
- "Clock me in." *(writes enabled)*
- "Take the open shift on Saturday." *(writes enabled; asks you to confirm first)*

## Requirements

- [uv](https://docs.astral.sh/uv/)
- Python 3.11 or newer
- A Deputy account (see the access modes below)

## Install

Choose your access mode first:

| Mode | Who it's for | Start here |
|---|---|---|
| **OAuth** | Any employee | [Employee sign-in](#employee-oauth) |
| **API token** | Deputy admins | [Admin setup](#admin-api-token) |
| **Calendar feed** | No API access | Set `DEPUTY_CALENDAR_URL` using the [client setup](docs/DESIGN.md#other-mcp-clients); your own roster only, read-only |

### Employee OAuth

Create a personal app at <https://once.deputy.com/my/oauth_clients> with the redirect
URI `http://localhost:8823/callback`. Create it there, not under your install's
Settings → Developer: an app made inside an install fails sign-in with
`invalid_client`. In Bash, replace the placeholders, sign in, then register the
server with Claude Code:

```bash
export DEPUTY_OAUTH_CLIENT_ID='your-client-id'
export DEPUTY_OAUTH_CLIENT_SECRET='your-client-secret'
uvx deputy-mcp login
claude mcp add deputy \
  -e DEPUTY_OAUTH_CLIENT_ID="$DEPUTY_OAUTH_CLIENT_ID" \
  -e DEPUTY_OAUTH_CLIENT_SECRET="$DEPUTY_OAUTH_CLIENT_SECRET" \
  -- uvx deputy-mcp
```

Login opens your browser and stores OAuth tokens in your OS keychain by default.
Run login and Claude Code as the same OS user. No terminal (for example, the Claude
Desktop bundle)? Set the two OAuth values and ask the assistant to sign you in: the
`deputy_login` tool opens the same browser sign-in.

### Admin API token

With your Deputy API token and install URL, register directly; no OAuth login is needed:

```bash
claude mcp add deputy \
  -e DEPUTY_API_TOKEN=your-deputy-token \
  -e DEPUTY_BASE_URL=https://your-company.eu.deputy.com \
  -- uvx deputy-mcp
```

### Codex CLI

For your own roster, replace `...` with your private calendar feed URL:

```bash
codex mcp add deputy --env DEPUTY_CALENDAR_URL=... -- uvx deputy-mcp
```

The first `uvx` run downloads dependencies; if Codex reports the tools as unavailable,
set `startup_timeout_sec = 60` under `[mcp_servers.deputy]` in `~/.codex/config.toml`.

### Claude Desktop

Download the `.mcpb` bundle from the [Releases page](https://github.com/augbastos/deputy-mcp/releases)
and drag it onto Settings → Extensions (double-clicking may not open it on Windows). Fill
in one access mode under Configure, then ask Claude to sign you in.

Other MCP clients and Docker: see [other ways to install](docs/DESIGN.md#other-ways-to-install).
All settings are listed in [`.env.example`](.env.example).

## Tools

- **Your data** (any account): `deputy_whoami`, `deputy_get_my_roster`,
  `deputy_next_shift`, `deputy_get_my_timesheets`, `deputy_get_my_colleagues`,
  `deputy_get_my_calendar_url`, `deputy_get_areas`
- **Team** (manager access): `deputy_get_team_roster`, `deputy_who_is_working`,
  `deputy_get_employee_info`, `deputy_search_shifts`
- **Sign-in** (OAuth credentials set): `deputy_login`
- **Actions** (only with `DEPUTY_ALLOW_WRITES=true`): `deputy_clock_in`,
  `deputy_clock_out`, `deputy_set_unavailability`,
  `deputy_claim_open_shift`

## Safe by default

- **Read-only** unless you set `DEPUTY_ALLOW_WRITES=true`; until then the action tools
  don't exist for the agent.
- **You confirm** before a shift is claimed, since that skips Deputy's approval step.
- **No guessing**: if a name matches several people, you get the list, not the first one.
- **Credential protection**: by default, tokens go only to `https://*.deputy.com`
  and OAuth tokens live in your OS keychain; errors are scrubbed of secrets before
  an agent sees them.

## Command line

The same client works as a command-line tool. With no subcommand, `deputy-mcp` starts
the MCP server (stdio by default; local HTTP is described in the
[design notes](docs/DESIGN.md#local-http-transport)).

```bash
deputy-mcp whoami            # who the credentials belong to
deputy-mcp roster [--team]   # your roster, or everyone's with manager access
deputy-mcp next [--employee NAME_OR_ID]
deputy-mcp timesheets | who | areas
deputy-mcp login [--no-browser] | logout
```

Add `--json` for machine output.

## Develop and test

```bash
uv sync                  # runtime and dev dependencies
uv run pytest            # no network needed
uv run ruff check .
uv run mypy
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full list of checks.

The unified CI workflow reports `Safety` and `CI passed`, the required merge checks.
It also runs lint, type checks, tests on Linux and Windows, builds, and security scans.
The Security workflow repeats the scans weekly.

## Layout

| Path | Contents |
|---|---|
| `src/deputy_mcp/` | The package: `client/` (Deputy API), `server/` (MCP tools, resources, prompts), CLI, OAuth |
| `tests/` | Unit, client, server and agent-interface tests; `live/` is opt-in and never runs in CI |
| `docs/` | [Design notes](docs/DESIGN.md) |
| `examples/` | A mock Deputy server (`mock_deputy.py`) |
| `.github/workflows/` | CI, security scans and release |

## Releases

Pushing a `v*` tag runs the release workflow (`.github/workflows/release.yml`). Release
notes are in the [changelog](CHANGELOG.md).

## Status

The everyday read tools and OAuth sign-in were tested against a real Deputy install with
an employee account. Manager tools, actions and the OAuth token refresh are covered by
automated tests but not yet verified on a live install. Not affiliated with Deputy.

## More

- [How it's built and why](docs/DESIGN.md)
- [Contributing](CONTRIBUTING.md) · [Security](SECURITY.md) · [Changelog](CHANGELOG.md) · [Roadmap](ROADMAP.md)

## License

[MIT](LICENSE)
