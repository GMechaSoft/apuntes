# Playwright MCP with Brave in Claude Code

Log of the setup process used to drive Brave from Claude Code through the Playwright MCP server (Ubuntu, Linux).

## Problem

The Playwright MCP server launches Google Chrome by default. With no Chrome installed, navigation fails:

```
Error: async initializeServer: Chromium distribution 'chrome' is not found at /opt/google/chrome/chrome
Run "npx playwright install chrome"
```

## Solution

Brave is Chromium-based, so the MCP server can be pointed at its executable.

### 1. Locate the Brave executable

```bash
which brave-browser
ls /opt/brave.com/brave/brave
```

Path used: `/opt/brave.com/brave/brave`.

### 2. Register the MCP server with `--executable-path`

```bash
claude mcp remove playwright
claude mcp add -s user playwright -- npx @playwright/mcp@latest --executable-path /opt/brave.com/brave/brave
```

### 3. Restart the Claude Code session

MCP servers are not hot-loaded. The Playwright tools appear only after restarting.

## Pitfall: configuration scope

`claude mcp add` defaults to `--scope local`, which binds the server to the **current project directory** only.

- Running it from `~/workspace/temp` registered the server for that project only.
- A session started in `~/workspace/HV` did not see it.
- Re-running `add` failed with `MCP server playwright already exists in local config`.

Fix: remove the local entry (`claude mcp remove playwright -s local`) and add it with `-s user`, or run `add` from the target project directory.

| Scope | Visibility | Stored in |
|-------|------------|-----------|
| `local` (default) | Current project, private | `~/.claude.json` (per-project entry) |
| `project` | Shared via `.mcp.json` in the repo | Project root |
| `user` | All projects for the user | `~/.claude.json` |

## Verification

After restarting, `browser_navigate` to the target URL worked and launched Brave.

## Notes

- Playwright opens a **fresh, empty profile**: no cookies and no existing sessions. Log in manually inside the Playwright window; do not hand credentials to the assistant.
- Opening a login-protected URL first redirects to the login page. After manual login, navigating again loads the page with the session.
- The MCP writes snapshots and console logs to `.playwright-mcp/` in the working directory. Add it to `.gitignore` in repositories to avoid committing it.
- To reuse a real Brave profile instead of an empty one, check the `--user-data-dir` option of `@playwright/mcp`. Not tested here.

## Example result

Target: `https://candidato.co.computrabajo.com/candidate/cv/edit/`

1. Unauthenticated: redirected to `secure.computrabajo.com/Account/Login`.
2. After manual login in the Playwright window: the edit route loaded without redirect.
