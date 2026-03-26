# ThinkBrowse — AI Frontend Testing for Claude Code & Cursor

Stop the manual screenshot loop. ThinkBrowse gives your AI coding agent direct access to your real Chrome browser — with your cookies, your sessions, your extensions. Navigate, click, screenshot, and QA without switching tabs.

Works with **Claude Code**, **Cursor**, **Cline**, **Windsurf**, and any MCP client.

## Install

```bash
npx @thinkbrowse/mcp
```

Or install the CLI globally:

```bash
npm install -g @thinkbrowse/cli
```

The CLI automatically installs the native host binary, which lets AI tools control Chrome.

## Add to your MCP config

```json
{
  "mcpServers": {
    "thinkbrowse": {
      "command": "npx",
      "args": ["@thinkbrowse/mcp"]
    }
  }
}
```

Config location:
- **Claude Code**: `~/.claude/settings.json`
- **Cursor**: `.cursor/mcp.json`
- **Cline**: VS Code settings → Cline MCP Servers

Then ask your agent: *"navigate to localhost:3000 and screenshot the checkout page"*

## Why ThinkBrowse

| | ThinkBrowse | Playwright MCP |
|---|---|---|
| Uses your real Chrome | ✅ | ❌ headless only |
| Keeps your cookies & sessions | ✅ | ❌ fresh browser every time |
| No "browser already in use" error | ✅ | ❌ common conflict |
| Works with auth-gated pages | ✅ | ❌ no cookies |
| Runs alongside your browser | ✅ | ❌ fights for Chrome profile |

## Agent skills

Skills for Claude Code, Cursor, Codex, and Gemini CLI are in `.claude/skills/`, `.cursor/skills/`, `.codex/skills/`, and `.gemini/skills/`.

## Manual binary install

If you need to install the native host without the CLI, download the binary for your platform from the [latest release](https://github.com/dundas/thinkbrowse/releases/latest) and run:

```bash
chmod +x thinkbrowse-host-* && ./thinkbrowse-host-* --install
```

## More

- [thinkbrowse.io](https://thinkbrowse.io)
- [Privacy Policy](https://thinkbrowse.io/privacy)
