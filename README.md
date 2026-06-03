# Hawkeye AI Bridge MCP Server

Use Hawkeye from AI coding assistants via the Model Context Protocol (MCP).

Hawkeye AI Bridge exposes Hawkeye's fast local search capabilities through MCP, making it possible for AI coding assistants to search large codebases, assets, sound files, models, text files, localization files, and project content through Hawkeye.

Hawkeye runs locally/on-premises. Your code is not uploaded by Hawkeye, and Hawkeye itself does not use AI for indexing or searching.

<img width="1371" height="841" alt="image" src="https://github.com/user-attachments/assets/2ce83c09-1302-4e07-b226-d23d14b676ee" />

---

## Table of Contents

- [What is this?](#what-is-this)
- [Why use Hawkeye AI Bridge?](#why-use-hawkeye-ai-bridge)
- [Prerequisites](#prerequisites)
- [Client Setup](#client-setup)
  - [Claude Desktop](#claude-desktop)
  - [Claude Code (CLI)](#claude-code-cli)
  - [VS Code / GitHub Copilot](#vs-code--github-copilot)
  - [Cursor](#cursor)
  - [OpenCode](#opencode)
- [Installing the Skill for Claude](#installing-the-skill-for-claude)
- [Available Tools](#available-tools)
- [Token Savings](#token-savings)
- [Example Prompts](#example-prompts)
- [Privacy and Security](#privacy-and-security)
- [Updating](#updating)
- [Removing](#removing)
- [Troubleshooting](#troubleshooting)
- [License](#license)
- [Support](#support)

---

## What is this?

This repository is the official public setup and discovery repository for using Hawkeye AI Bridge as an MCP server.

It does not contain the proprietary Hawkeye AI Bridge source code. Hawkeye and Hawkeye AI Bridge are proprietary software owned by Zaragsoft.

---

## Why use Hawkeye AI Bridge?

Hawkeye helps developers quickly search and understand large projects. With MCP support, AI coding assistants can ask Hawkeye for fast local search results, saving tokens on each call.

Typical use cases:

- Find all references to a class, function, symbol, asset, localization key, or file.
- Search large codebases without relying only on editor search.
- Help AI coding assistants understand more of the project before making changes.
- Navigate large game/software projects with less reliance on tribal knowledge.
- Keep search local/on-premises.
- ~80% token reduction vs. blind file reading.
- If you do not specify any groups then all groups will be used.

---

## Prerequisites

- **Hawkeye** installed — get it from https://www.zaragsoft.se/downloads
- **Hawkeye AI Bridge** installed — get it from https://www.zaragsoft.se/aibridge
- A local project indexed by Hawkeye
- Video walkthrough: https://www.youtube.com/watch?v=l1J-G36QSwI

**Recommended install path:** Place the `AIBridge` folder at `C:\Program Files\Hawkeye\AIBridge\`. Use this path in all examples below. If you install elsewhere, substitute your actual path throughout.

---

## Client Setup

---

### Claude Desktop

**You don't need this zip for Claude Desktop.** Download `Hawkeye.mcpb` from the release page and install it directly:

- If `.mcpb` is a recognised file type on your system: double-click the file.
- Otherwise: go to **Settings → Extensions → Install Extension** and select the file.

---

### Claude Code (CLI)

#### Quick Start (5 steps)

1. **Install Hawkeye** from https://www.zaragsoft.se/downloads if you haven't already.
2. **Move the `AIBridge` folder** to a permanent location — recommended: `C:\Program Files\Hawkeye\AIBridge\`. Don't run it from Downloads; the registered path must keep working.
3. **Register the MCP server** with Claude Code:
   ```powershell
   claude mcp add hawkeye --scope user -- "C:\Program Files\Hawkeye\AIBridge\HawkeyeAIBridge.exe"
   ```
4. **Install the Skill** so Claude knows when to use Hawkeye:
   ```powershell
   $skillDir = "$env:USERPROFILE\.claude\skills\hawkeye-search"
   New-Item -ItemType Directory -Force -Path $skillDir | Out-Null
   Copy-Item "C:\Program Files\Hawkeye\AIBridge\hawkeye-search.skill" "$skillDir\SKILL.md"
   ```
5. **Verify** — close and reopen your terminal, then run:
   ```powershell
   claude mcp list
   ```
   You should see `hawkeye` with 9 tools available.

#### Make sure Claude Code can find Hawkeye

The MCP server needs to locate `hawkeye.exe`. Pick one option:

**Option A (recommended): Set `HAWKEYE_PATH`**

```powershell
# PowerShell — persistent
[Environment]::SetEnvironmentVariable("HAWKEYE_PATH", "C:\Program Files\Hawkeye\hawkeye.exe", "User")
```

```cmd
:: Command Prompt — persistent
setx HAWKEYE_PATH "C:\Program Files\Hawkeye\hawkeye.exe"
```

> After setting an environment variable, **close and reopen your terminal** so the new value takes effect.

**Option B: Add Hawkeye to PATH via Windows GUI**

1. Press `Win`, type **Environment Variables**, open **Edit the system environment variables**.
2. Click **Environment Variables…** → under **User variables**, select **Path** → **Edit…** → **New**.
3. Add `C:\Program Files\Hawkeye` and click OK on all dialogs.
4. Restart your terminal.

> Avoid `setx PATH "%PATH%;..."` from the command line — it can truncate PATH at 1024 characters. Use the GUI instead.

#### Notes on the registration command

- **`--scope user`** makes Hawkeye available in every Claude Code session, not just the current folder.
- **The `--` separator is required.** It tells Claude Code where its own flags end and the registered command begins. Without it, registration may fail.

---

### VS Code / GitHub Copilot

Hawkeye works as an MCP server in VS Code's Copilot Agent mode. Register the exe, then open Copilot Chat in Agent mode.

#### Quick install (default path)

[![Install in VS Code](https://img.shields.io/badge/VS_Code-Add_Hawkeye_MCP-0098FF?style=flat&logo=visualstudiocode)](vscode:mcp/install?%7B%22name%22%3A%22hawkeye%22%2C%22command%22%3A%22C%3A%5C%5CProgram%20Files%5C%5CHawkeye%5C%5CAIBridge%5C%5CHawkeyeAIBridge.exe%22%7D)

Click the button above, then click **Allow** in VS Code.

#### Manual setup

Add this to your `.vscode/mcp.json` (create the file if it doesn't exist):

```json
{
  "servers": {
    "hawkeye": {
      "type": "stdio",
      "command": "C:\\Program Files\\Hawkeye\\AIBridge\\HawkeyeAIBridge.exe"
    }
  }
}
```

To make Hawkeye available across all projects, add it to your user-level config instead:

```json
// %APPDATA%\Code\User\mcp.json
{
  "servers": {
    "hawkeye": {
      "type": "stdio",
      "command": "C:\\Program Files\\Hawkeye\\AIBridge\\HawkeyeAIBridge.exe"
    }
  }
}
```

#### Using Hawkeye in Copilot Chat

1. Open Copilot Chat (`Ctrl+Alt+I`)
2. Switch to **Agent mode** (the dropdown at the top — must be Agent, not Ask or Edit)
3. Try: *"Search for findTeam in my codebase"*

> **Note:** The `hawkeye-search.skill` file is for Claude Code and Claude Desktop only — VS Code uses Copilot's own context system and doesn't need it.

---

### Cursor

Open **Settings → Tools & MCP → New MCP Server** and add:

```json
{
  "mcpServers": {
    "hawkeye": {
      "command": "cmd",
      "args": ["/c", "C:\\Program Files\\Hawkeye\\AIBridge\\HawkeyeAIBridge.exe"],
      "env": {}
    }
  }
}
```

> **Why `cmd /c`?** Cursor passes the command path to the shell without quoting it, which breaks paths containing spaces (like `C:\Program Files\`). Wrapping it with `cmd /c` works around this.

Adjust the path if Hawkeye AI Bridge is installed somewhere else.

---

### OpenCode

Add the following to your `opencode.json` (create it in your project root or home directory if it doesn't exist):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "hawkeye": {
      "type": "local",
      "command": ["C:\\Program Files\\Hawkeye\\AIBridge\\HawkeyeAIBridge.exe"]
    }
  }
}
```

Three things differ from the Claude Code config:

- The key is `"mcp"`, not `"mcpServers"`
- `command` is an **array** of strings, not a `command` + `args` split
- `"type": "local"` is required

#### Installing the Skill for OpenCode

The Skill is required for OpenCode. Without it, OpenCode will try to call `hawkeye.exe` directly on the command line rather than routing tool calls through the MCP server.

OpenCode expects skills to be extracted from the `.skill` zip archive:

```powershell
$skillDir = "$env:USERPROFILE\.opencode\skills"
New-Item -ItemType Directory -Force -Path $skillDir | Out-Null

$tempZip = "$env:TEMP\hawkeye-search.zip"
Copy-Item "C:\Program Files\Hawkeye\AIBridge\hawkeye-search.skill" $tempZip -Force
Expand-Archive -Path $tempZip -DestinationPath $skillDir -Force
Remove-Item $tempZip
```

Once both are in place, restart OpenCode and try: *"Find where PlayerController is defined."* You should see it call `hawkeye_search_minimal`.

---

## Installing the Skill for Claude

The Skill teaches Claude *when* and *how* to use the Hawkeye tools. Without it, Claude has to guess from tool descriptions alone — results will be worse.

The shipped file is called `hawkeye-search.skill` but Claude Code expects it to be named `SKILL.md` inside a named folder.

**PowerShell (one command):**

```powershell
$skillDir = "$env:USERPROFILE\.claude\skills\hawkeye-search"
New-Item -ItemType Directory -Force -Path $skillDir | Out-Null
Copy-Item "C:\Program Files\Hawkeye\AIBridge\hawkeye-search.skill" "$skillDir\SKILL.md"
```

**Manual:**

1. Open File Explorer and navigate to `%USERPROFILE%\.claude\skills\` (create the `skills` folder if it doesn't exist).
2. Create a new folder named `hawkeye-search`.
3. Copy `hawkeye-search.skill` into it.
4. Rename the file from `hawkeye-search.skill` to `SKILL.md`. (Enable "File name extensions" in File Explorer's View menu if you can't see the extension.)

> To restrict the Skill to a single project, place `SKILL.md` at `<your-project>\.claude\skills\hawkeye-search\SKILL.md` instead.

---

## Available Tools

| Tool | Description |
|------|-------------|
| `hawkeye_search_minimal` | Token-efficient search — returns only file paths and line numbers. **91% fewer tokens** than grep. Best for chat-based lookups. |
| `hawkeye_search` | Full search with metadata — returns complete result objects with context. Best for detailed analysis. |
| `hawkeye_get_groups` | List all searchable code groups configured in Hawkeye. |
| `hawkeye_get_editors` | List configured editors for opening files. |
| `hawkeye_execute_editor` | Open a file in a configured editor at a specific line number. |
| `hawkeye_health_check` | Verify Hawkeye service is running and accessible. |
| `validate_hawkeye_path` | Check that the configured Hawkeye path exists and is valid. |
| `get_settings` | Retrieve current Hawkeye configuration (path, max results, case sensitivity). |
| `save_settings` | Save Hawkeye configuration. |
| `copy_to_clipboard` | Copy text to the system clipboard. |

---

## Token Savings

| Approach | Tokens used |
|----------|------------|
| Blind file reading (no Hawkeye) | ~100,000 per session |
| With `hawkeye_search_minimal` | ~20,000 per session |
| **Savings** | **~80% reduction** |

---

## Example Prompts

Ask your AI coding assistant:

- List groups inside Hawkeye to find all the groups.
- Use Hawkeye to find all references to this class.
- Search the workspace for where this asset name is used.
- Find likely files related to the inventory system in group 4,7.
- Search for this localization key across the project.
- Find where this Blueprint or Unreal asset is referenced.
- Show me files related to feature loadout before editing code.
- Find all the sounds for explosion inside groups 1,5,7
- Use Hawkeye to find code and content references before changing this file.
- If you do not specify any groups then all groups will be used.

---

## Privacy and Security

- Hawkeye runs locally/on-premises.
- Hawkeye does not upload your source code.
- Hawkeye itself does not use AI for indexing or searching.
- The MCP server only exposes Hawkeye functionality to tools you configure locally.
- You control which MCP clients can connect to it.
- Always review AI-generated code changes before applying them.
- Our privacy guidelines are available here - https://www.zaragsoft.se/privacy

---

## Updating

When a new version of `HawkeyeAIBridge.exe` is released:

- **Same install path:** just overwrite the exe. No re-registration needed.
- **Different path:** unregister and re-add:
  ```powershell
  claude mcp remove hawkeye
  claude mcp add hawkeye --scope user -- "C:\Program Files\Hawkeye\AIBridge\HawkeyeAIBridge.exe"
  ```

When a new `hawkeye-search.skill` ships, repeat the Skill install step — `Copy-Item` will overwrite the existing `SKILL.md`.

---

## Removing

```powershell
# Unregister from Claude Code
claude mcp remove hawkeye

# Delete the Skill (optional)
Remove-Item -Recurse "$env:USERPROFILE\.claude\skills\hawkeye-search"

# Delete the install folder (optional)
Remove-Item -Recurse "C:\Program Files\Hawkeye\AIBridge"

# Remove the HAWKEYE_PATH environment variable (optional)
[Environment]::SetEnvironmentVariable("HAWKEYE_PATH", $null, "User")
```

---

## Troubleshooting

### `claude` command not found

Claude Code isn't installed or isn't on your PATH. Install from https://docs.claude.com/en/docs/build-with-claude/claude-code and restart your terminal.

### `claude mcp list` doesn't show hawkeye

You may have registered it in a different scope. Re-register with `--scope user`:

```powershell
claude mcp remove hawkeye
claude mcp add hawkeye --scope user -- "C:\Program Files\Hawkeye\AIBridge\HawkeyeAIBridge.exe"
```

### `claude mcp list` shows hawkeye but with 0 tools

The exe was registered but failed to start. Common causes:

- The `AIBridge` folder was moved or deleted after registration.
- Hawkeye itself can't be found — run `hawkeye --version` in a fresh terminal to check.
- Fix: `claude mcp remove hawkeye` then re-register.

### Hawkeye not found

```
Error: Failed to start Hawkeye process
```

1. Open a new terminal and run `hawkeye --version`.
2. If that fails, set `HAWKEYE_PATH` (see Prerequisites above) and close/reopen your terminal.
3. Environment variables only apply to terminals opened *after* you set them.

### Skill isn't being used by Claude

- Confirm the file exists at `%USERPROFILE%\.claude\skills\hawkeye-search\SKILL.md`.
- Make sure you didn't end up with `SKILL.md.skill` — Windows may hide the original `.skill` extension. Enable "File name extensions" in File Explorer's View menu to check.
- Restart your Claude Code session — Skills are loaded at session start.

### Cursor: connection failed with path error

If Cursor logs show `'C:\Program' is not recognized as an internal or external command`, the path with spaces is being passed unquoted. Use the `cmd /c` workaround shown in the Cursor section above.

### No groups returned

```
No groups available / Empty group list
```

1. Run `hawkeye --getgroups` manually to test.
2. Verify Hawkeye configuration is correct.
3. Check working directory permissions.

### Timeout errors

```
Error: Operation timed out waiting for Hawkeye
```

1. Check if the Hawkeye process is hanging.
2. Check system resources.
3. Restart Hawkeye and try again.

---

## License

**Dual License:**

- **SKILL.md and skill components**: MIT License — freely modifiable and redistributable.
- **HawkeyeAIBridge executable**: Proprietary (Zaragsoft) — free to use, cannot be modified, reverse-engineered, or redistributed.

See [LICENSE](LICENSE) for complete terms.

---

## Support

- **Hawkeye**: https://www.zaragsoft.se/downloads
- **Hawkeye AI Bridge**: https://www.zaragsoft.se/aibridge
- **Claude Code**: https://docs.claude.com/en/docs/build-with-claude/claude-code
- **Questions or feedback about Hawkeye AI Bridge**: info@zaragsoft.se
