---
name: hawkeye-search
description: Execute fast local code searches using Hawkeye with natural language parameter extraction
compatibility: 
  - claude-opus-4-6
  - claude-sonnet-4-6
  - claude-haiku-4-5-20251001
---

# Hawkeye Search Skill

## When to Use

Use this skill whenever the user wants to find anything in the codebase. This is the **default first step** for all code searches — not just explicit Hawkeye requests. Typical triggering patterns include:

- Finding specific code elements: "Find the Player class", "Search for armor enum", "Where is UpdateHealth defined?"
- Searching for functions, classes, enums, methods, variables, textures, or constants
- Finding relationships between code: "What uses ThePlayerList?", "Who calls UpdateHealth?"
- Locating assets, content, or string references
- Any request to locate, trace, or explore code — even without the word "search" or "Hawkeye"

**Skip this skill if:**
- The query is empty or only contains filler words
- The user is not searching for code elements
- The query should use Claude's native knowledge instead

## Fallback Strategy

Always try `hawkeye_search_minimal` first. If it returns **zero results**:
1. Try broadening the query (remove type keywords, try a shorter term)
2. If still no results, fall back to `Grep` for content search or `Glob` for file name search
3. Never pre-emptively skip Hawkeye — let it fail before falling back

If Hawkeye returns hits but you need to see the actual code, use a targeted `Read` on the specific file and line range. Do **not** run a full Grep to get context you could read directly.

## Parameter Extraction

Extract three parameters from the user's natural language query:

### Query
The search term to send to Hawkeye. 

**Extraction rules:**
1. Remove filler words: 'the', 'all', 'uses of', 'find', 'search', 'for'
2. Keep type keywords: 'class', 'enum', 'function', 'method', 'struct', etc. – these narrow searches and provide context
3. Preserve the core search terms

**Examples:**
- "Find the Player class" → `query: "Player class"` (removes "the", keeps "class")
- "Search for armor enum" → `query: "armor enum"` (removes "search for", keeps "enum")
- "Where is UpdateHealth defined?" → `query: "UpdateHealth"` (removes "where", "is", "defined")
- "Search for enum loadoutType_t" → `query: "enum loadoutType_t"` (keeps both "enum" and "loadoutType_t" for precision)
- "Find all uses of texture loader" → `query: "texture loader"` (removes "find", "all", "uses of")

### MaxResults
The maximum number of results to return.

**Extraction rules:**
1. Look for numeric values in patterns like "maxresults X" or "max results X"
2. Default to 50 if not specified
3. If the user specifies "all" or "everything", use 100

**Examples:**
- "Search for armor enum, maxresults 20" → `maxresults: 20`
- "Find Player class" → `maxresults: 50` (default)
- "Search for render function, maxresults 10" → `maxresults: 10`

### Groups
Optional filter list to restrict searches to specific code groups/modules.

**Extraction rules:**
1. Look for patterns like "using groups X,Y,Z" or "in X and Y groups"
2. Split comma-separated or "and"-separated values into an **array of strings**
3. Pass the array directly to the hawkeye_search MCP tool
4. If groups are invalid or don't exist, still search with the valid ones; don't abort

**Examples:**
- "Search for render in Rendering,Engine groups" → `groups: ["Rendering", "Engine"]`
- "Look for GameManager in Scripts and Core groups" → `groups: ["Scripts", "Core"]`
- "Find Player using groups Scripts,AI,Utilities" → `groups: ["Scripts", "AI", "Utilities"]`
- "Search for armor" → `groups: []` or undefined (no group filtering)

## Output Format

Return results in minimal, token-efficient format:
- Show only **file paths** and **line numbers**
- One result per line in the format: `filename.ext:line_number`
- Example:
  ```
  Player.cpp:45
  Player.h:12
  GameManager.cpp:189
  ```

## Error Handling

**No results found:** Display a message like:
> "No results found for 'query_term'. Please expand your search query or add more groups to filter by."

**Invalid query (empty after removing filler words):** Skip using Hawkeye and revert to Claude's native search capability.

**Invalid groups:** If the user specifies groups that don't exist, still attempt the search with valid ones and proceed normally.

## Implementation Notes

This skill calls the `hawkeye_search_minimal` MCP tool with:
- `query`: Extracted search term (string)
- `maxresults`: Maximum results (number, default 50)
- `groupMasks`: Array of group filter strings (array of strings, optional)

After receiving minimal results (file paths and line numbers), you can elaborate on findings by reading specific files and providing code context as needed.

## Important: Never Call Hawkeye Directly

- **Never** run `Hawkeye.exe`, `HawkeyeAIBridge.exe`, or `HawkeyeMcpServer.exe` via PowerShell or Bash
- **Always** use the `mcp__hawkeye__*` MCP tools to interact with Hawkeye
- `HawkeyeAIBridge.exe` is the MCP server — it runs separately and Claude communicates with it exclusively through MCP tools
