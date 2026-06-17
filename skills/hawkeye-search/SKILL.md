---
name: hawkeye-search
description: Replaces grep and file reading for all code and asset searches. Use hawkeye_search_minimal instead of grep, bash, Glob, or Read whenever the user wants to find, locate, analyze, or explore anything in the project. Default flow is hawkeye_search_minimal (locations) + hawkeye_expand_context for merged, server-side context retrieval in one batched call.
compatibility: 
  - claude-opus-4-6
  - claude-sonnet-4-6
  - claude-haiku-4-5-20251001
---

# Hawkeye Search Skill

## Before You Start

Proceed directly with `hawkeye_search_minimal` — do not call `hawkeye_health_check` upfront. The user has Hawkeye set up and knows what they are doing. Do not add caveats about setup, installation, or whether the project is indexed.

Only call `hawkeye_health_check` if a tool call fails or returns no response at all (i.e. the MCP is not responding). Use it to diagnose the failure, report the error, and stop.

---

## When to Use

**Do NOT use grep, bash, Glob, or bare Read calls to search for code or assets. Use `hawkeye_search_minimal` instead.** Hawkeye is the correct tool for all searches in this project.

Use this skill whenever the user wants to find anything in the project — code **or assets/content**. This is the **default first step** for all such searches — not just explicit Hawkeye requests. Typical triggering patterns include:

- Finding specific code elements: "Where is class Player?", "Search for enum signInToken"
- Searching for functions, classes, enums, methods, variables, or constants
- Finding relationships between code: "What uses ThePlayerList?", "Who calls UpdateHealth?"
- Code review/debugging: "Is this variable used safely?", "Find potential bugs in X"
- Any request to locate, trace, explore, or analyze code
- **Asset/content lookups**: "Do we have any sounds for health?", "Do we have any icons for armor?", "Is there a texture for the supply truck?", "What voice lines exist for the General?"
  - These are still Mode 1 / Mode 1.5 searches — `hawkeye_search_minimal` indexes filenames and string references (e.g. `.ini` entries, asset definitions, sound event names) just like code symbols. Search for the descriptive term (e.g. `"health sound"`, `"armor icon"`) and treat hits in `.ini`/data files the same as code hits.
  - If the goal is just "does X exist / where is it" → Mode 1 is enough.
  - If the goal is "show me how it's referenced/used" → Mode 1.5 (`hawkeye_expand_context`) to see the surrounding definition block.

**Skip this skill only if:**
- The query is empty or only contains filler words
- The user is explicitly asking about something outside the project (e.g. general language documentation)

---

## Search & Context Workflow

Mode 1.5 (search + `hawkeye_expand_context`) is the **default** for almost everything. Mode 1 alone is only for the narrow case where you genuinely just want a location, with no need to see the code.

### Mode 1: Simple Location Lookup
**Use for:** "Where is X?", "Find X", "Search for X"

**Method:** `hawkeye_search_minimal` only
- Pinpoints exact file paths and line numbers
- Fast execution, low token cost (~100 tokens)
- Perfect for simple discovery questions

**When to use:**
- "Where is class Player defined?"
- "Find all references to enum signInToken"
- "Show me where GameManager is used"
- "Search for function UpdateHealth"

**Examples:**
- Query: `"class Player"` → ~5-10 locations, 100 tokens, <1 second
- Query: `"enum signInToken"` → ~3-8 locations, 100 tokens, <1 second
- Query: `"LadderPrefMap"` → 28 locations, 100 tokens, instant

**If returns zero results:**
1. Try broadening the query (remove type keywords, try a shorter term).
2. If still nothing, the most likely cause is a group filter mismatch — not that the code doesn't exist. Call `hawkeye_get_groups` and show the user the available groups, then ask: "No results found — did you want to search in specific groups? Here are the groups available in your project: [list]. You can say something like 'search in group Code' to narrow it down, or I'll search all groups by default."
3. Do not conclude the code doesn't exist until the user has confirmed the scope they want searched.

---

### Mode 1.5: Targeted Context Extraction (default)
**Use for:** anything beyond a bare location — "Is X safe?", "Find bugs in X", "Analyze X", "Show me X with context", "What uses X?", etc.

**Why this exists:** `hawkeye_search_minimal` returns results **grouped by file**, each with a `lines` array of every hit in that file, e.g.:
```
{"file": "PartitionManager.cpp", "lines": [1289,1325,1353,...,5453]}
```
Re-searching the file with `grep -C 3 "pattern" file` re-matches by pattern — for a file with 30 hits that produces 30 context blocks, often bigger than the file itself, and it can also match occurrences Mode 1 never flagged. Reading the whole file is worse: a 5,000+ line file costs tens of thousands of tokens just to open.

**Method — call `hawkeye_expand_context` directly:**
1. Take the Mode 1 result(s) — one or more `{"file": ..., "lines": [...]}` entries.
2. Pass them straight through as `hits: [{file, lines}, ...]` to `hawkeye_expand_context`. The server does the merging *and* the reading in one call — there is no manual range math and no per-range `Read` calls.
3. **Defaults (omit unless the user asks for something different):** `contextBefore=2, contextAfter=5, mergeGap=8, maxTotalLines=1000`.
   - Asymmetric `-2/+5` matches how code reads — the lines *after* a hit usually explain "how it's used"; 2 lines before is enough for the immediate guard/declaration.
   - `mergeGap=8` merges hits within 8 lines of each other into one snippet, so adjacent `-2/+5` windows that touch/overlap collapse into a single block instead of duplicating shared lines.
4. **If the user specifies a context size N** (e.g. "show me 10 lines of context", "wider context", "just a couple lines around it") — pass `contextBefore=N, contextAfter=N, mergeGap=2N` instead.
5. **Batching for large sweeps:** the default `maxTotalLines=1000` caps a single call at roughly **~140 hits** (measured — varies with how clustered the hits are; denser clusters merge into fewer lines and raise the effective ceiling, sparse hits lower it).
   - For typical single-file or few-file lookups (well under 100 hits total), one call is enough.
   - For full-project sweeps with hundreds of hits, split the `hits` array into batches of roughly 120-140 hits (group by file so a file's hits aren't split across batches where possible) and issue one `hawkeye_expand_context` call per batch.
   - **Always check the response's `truncated` and `truncated_clusters` fields.** If `truncated: true`, the listed `truncated_clusters` were cut off — issue one more call with just those hits (with the same context settings) to get full coverage. Don't assume a clean cutoff at exactly 140 hits.
   - Rule of thumb from a measured 808-hit/168-file sweep: ~6 batched `hawkeye_expand_context` calls covered everything, vs ~11 natural chunks for an equivalent `grep -C` sweep — fewer round trips, not more, even at full-project scale.

**Example — a file with 30 hits spanning lines 1289-5453 (file is 5,500+ lines):**
- Whole-file read: ~5,500 lines ≈ tens of thousands of tokens
- Re-grep per pattern (`grep -C3` × 30 occurrences): large, duplicated, and not limited to the 30 Mode-1 lines
- `hawkeye_expand_context` with default `-2/+5`, `mergeGap=8`: 18 merged snippets, ~184 lines total (~1,950 tokens), returned in **one call** — roughly **28x cheaper than a full-file read** and far fewer round trips than 18 separate `Read` calls

**Real example: "Is LadderPrefMap used safely?"**
```
Mode 1: hawkeye_search_minimal("LadderPrefMap")
→ Returns: {"file": "PopupHostGame.cpp", "lines": [189, 196, 210, 284, 292]}

Mode 1.5: hawkeye_expand_context(hits=[{"file": "PopupHostGame.cpp", "lines": [189,196,210,284,292]}])
→ (defaults: contextBefore=2, contextAfter=5, mergeGap=8)
→ Returns 3 merged snippets covering ranges [187-201], [208-215], [282-297]

Result: ✅ SAFE - confirmed const_iterator pattern with proper bounds,
        all 5 hits covered in 1 call, no re-search needed
```

**If the default misses needed context:** the most common case is a guard/condition or comment more than 2 lines above the hit. Rather than switching the whole search to symmetric `±N`, re-call `hawkeye_expand_context` for just that file/hit with a larger `contextBefore` — keep the rest at the default.

**When NOT to use:** Mode 1.5 applies even when a file has only 1-2 hits — `hawkeye_expand_context` handles trivial cases the same way, so there's no "simpler" full-file fallback to fall back to. Skip Mode 1.5 only if the user **explicitly** asks to read the whole file or the whole function/class (e.g. "show me the entire function", "read the full file") — then do that full read directly instead.

---

### Beyond this skill: cross-pattern & full-content browsing
If a request genuinely needs a *different* pattern than what Mode 1 located (e.g. "find all conditionals that test X", regex across the codebase, or browsing full match content with editor integration), don't reach for `grep` inline — that's what the **Hawkeye AI Bridge artifact** is for. It runs searches (including `hawkeye_search` full/content mode) against the same cached index, with tabs, group filters, and "open in editor" actions, without putting large result sets into this conversation's context. Point the user there for that kind of exploration instead of running grep yourself.

---

## Mode Decision Guide

| Question Type | Mode | Cost | Speed | Example |
|---------------|------|------|-------|---------|
| "Where is X?" | Mode 1 | 100 tokens | <1s | "Where is Player class?" |
| "Find X" | Mode 1 | 100 tokens | <1s | "Find enum signInToken" |
| "Search for X" | Mode 1 | 100 tokens | <1s | "Search for UpdateHealth" |
| "Show me X's locations with context" | Mode 1.5 | ~100 + ~1 batched call | <1s + 1 call | "Show me where validCustom is set" |
| "Is X safe?" | Mode 1.5 | ~100 + ~1 batched call | <1s + 1 call | "Is validCustom safe?" |
| "Find bugs in X" | Mode 1.5 | ~100 + ~1 batched call | <1s + 1 call | "Find bugs in GameManager" |
| "Analyze X" / "What uses X?" | Mode 1.5 | ~100 + ~1 batched call | <1s + 1 call | "Analyze iterator patterns" |
| Full-project sweep (100s of hits, many files) | Mode 1.5, batched ~140 hits/call | ~100 + N batched calls | <1s + N calls (N ≈ hits/140) | "Show every use of ThePlayerList with context" |
| Cross-pattern / full-content browsing | Hawkeye AI Bridge artifact | n/a (outside this skill) | n/a | "Find all conditionals that test X" |

---

## Parameter Extraction

### Query
The search term to send to Hawkeye.

**Extraction rules:**
1. Remove filler words: 'the', 'all', 'uses of', 'find', 'search', 'for'
2. Keep type keywords: 'class', 'enum', 'function', 'method', 'struct' – these narrow searches
3. Preserve the core search terms

**Examples:**
- "Where is class Player?" → Query: `"class Player"`
- "Find enum signInToken" → Query: `"enum signInToken"`
- "Search for GameManager" → Query: `"GameManager"`
- "Is validCustom used safely?" → Query: `"validCustom"`

### MaxResults
Maximum number of results to return.

**Extraction rules:**
1. Look for numeric values like "maxresults X" or "max results X"
2. Default to 50 if not specified
3. Use 100 for "all" or "everything"

**Examples:**
- "Find all references, maxresults 20" → `maxresults: 20`
- "Where is Player class?" → `maxresults: 50` (default)

### Groups
Optional filter to restrict searches to specific code groups.

**Extraction rules:**
1. Look for patterns like "using groups X,Y,Z" or "in X and Y groups"
2. **First call `hawkeye_get_groups`** to get available groups and their masks
3. Match user-provided group names against available groups
4. Extract the group mask from matching group objects
5. Pass group masks to `hawkeye_search_minimal` via `groupMasks` parameter
6. If group name not found, inform user of available groups

### hawkeye_expand_context parameters

**Extraction rules:**
1. `hits`: built from the Mode 1 result(s) — `[{file, lines}, ...]`. For large sweeps, chunk this array into ~120-140-hit batches (grouped by file) and issue one call per batch.
2. `contextBefore` / `contextAfter`: default `2` / `5`. If the user specifies a single context size N, set both to N.
3. `mergeGap`: default `8`. If using symmetric `±N` context, set to `2N`.
4. `maxTotalLines`: leave at default `1000` unless the user explicitly asks for a larger/smaller single response.
5. After each call, check `truncated` / `truncated_clusters` — re-call with just the truncated hits if `truncated: true`.

**Group Resolution Process:**
```
User input: "Search in Code group"
↓
Call: hawkeye_get_groups()
↓
Returns: [
  {name: "Code", mask: "0x001"},
  {name: "Tests", mask: "0x002"},
  {name: "Docs", mask: "0x004"}
]
↓
```
