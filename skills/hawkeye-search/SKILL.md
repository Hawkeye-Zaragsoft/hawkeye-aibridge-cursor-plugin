---
name: hawkeye-search
description: Execute fast local code searches using Hawkeye with natural language parameter extraction. Two modes: simple lookup (Hawkeye minimal) or deep analysis (Hawkeye minimal + Grep).
compatibility: 
  - claude-opus-4-6
  - claude-sonnet-4-6
  - claude-haiku-4-5-20251001
---

# Hawkeye Search Skill

## When to Use

Use this skill whenever the user wants to find anything in the codebase. This is the **default first step** for all code searches — not just explicit Hawkeye requests. Typical triggering patterns include:

- Finding specific code elements: "Where is class Player?", "Search for enum signInToken"
- Searching for functions, classes, enums, methods, variables, or constants
- Finding relationships between code: "What uses ThePlayerList?", "Who calls UpdateHealth?"
- Code review/debugging: "Is this variable used safely?", "Find potential bugs in X"
- Any request to locate, trace, explore, or analyze code

**Skip this skill if:**
- The query is empty or only contains filler words
- The user is not searching for code elements
- The query should use Claude's native knowledge instead

---

## 🎯 Quick Decision: Which Mode to Use?

**This is the most important section.** Read this first.

### Ask Yourself One Question:

**"Do I need to SEE THE CODE around the location, or just KNOW WHERE IT IS?"**

**→ Just know WHERE (the file and line number)?**
- Use **Mode 1: Location Lookup** ✅
- Returns: File paths + line numbers only
- Cost: 100 tokens
- Speed: <1 second
- Answer: "Player class is at PlayerController.cs:42"

**→ Need to SEE surrounding code/context?**
- Use **Mode 2: Code Analysis** 
- Returns: File + line + code around it
- Cost: 250-350 tokens
- Speed: 2-3 seconds
- Answer: "Player class at line 42, and here's what's around it..."

### ⚠️ DEFAULT: Always start with Mode 1

**Use Mode 1 for 80% of your searches.** Only switch to Mode 2 if you specifically need context.

```
Mode 1 (Location):     ██ 100 tokens        (~27x cheaper)
Mode 2 (Analysis):     ████████ 300 tokens
Full codebase search:  ██████████████████████████ 2,700+ tokens (WRONG)
```

---

## Two-Mode Approach

### What "Minimal" Actually Means

**"Minimal" = "Focused on exactly what you asked for"**
- NOT "limited/weak/incomplete"
- NOT "gives you less"
- ACTUALLY "gives you ONLY what you need" (and wastes zero tokens on extras)

When you ask "Where is X?", you want a location, not the entire codebase.  
`hawkeye_search_minimal` gives you **exactly that** — fast and cheap.

Choose your mode based on what you're looking for:

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
1. Try broadening query (remove type keywords)
2. Try shorter or related terms
3. If still nothing, code likely doesn't exist in codebase

---

### Mode 2: Discovery & Bug Finding
**Use for:** "Is X safe?", "Find bugs in X", "Analyze code", "Reason about X"

**Method:** `hawkeye_search_minimal` (Phase 1) + `grep` (Phase 2)

#### Phase 1: Pinpoint Locations with Hawkeye
```
hawkeye_search_minimal("validCustom")
→ Find: 22 exact locations in specific files (~100 tokens)
Result: validCustom.cpp:149, validCustom.cpp:58, LadderDefs.cpp:177, etc.
```

#### Phase 2: Get Context with Grep
```
For each critical location:
grep -C 3 "validCustom" file.cpp
→ Show lines around location with surrounding context (~150-200 tokens)
```

**Total Mode 2 cost:** ~250-350 tokens vs 2500+ for traditional Grep only

**When to use:**
- "Is validCustom used safely everywhere?"
- "Find potential bugs in GameManager usage"
- "Verify iterator access patterns are correct"
- "Analyze how this function is initialized"
- "Find unsafe pointer dereferences"

**Process:**
1. Hawkeye pinpoints exact locations (you know WHERE)
2. Grep gets context around those lines (you see HOW)
3. Combined view reveals patterns and potential issues

**Real Examples:**

```
Mode 2 Example 1: "Is LadderPrefMap used safely?"

Phase 1 - Hawkeye Minimal:
hawkeye_search_minimal("LadderPrefMap") 
→ Returns: Lines 189, 196, 210, 284, 292 in PopupHostGame.cpp

Phase 2 - Grep Context:
grep -C 2 "const LadderPrefMap\|LadderPrefMap::const_iterator" PopupHostGame.cpp
→ Shows:
  const LadderPrefMap recentLadders = ladPref.getRecentLadders();
  for (LadderPrefMap::const_iterator cit = recentLadders.begin(); ...)

Result: ✅ SAFE - confirmed const_iterator pattern with proper bounds
```

```
Mode 2 Example 2: "Find bugs in validCustom"

Phase 1 - Hawkeye Minimal:
hawkeye_search_minimal("validCustom")
→ Returns: Lines 149 (assignment), 177 (usage), 196 (usage), etc.

Phase 2 - Grep Context:
grep -C 3 "validCustom = " LadderDefs.cpp
→ Shows:
  lad->validCustom = atoi(line.str() + 7);

grep -C 2 "if.*validCustom" PopupHostGame.cpp
→ Shows:
  if (info && info->validCustom && usedLadders.find(info) == usedLadders.end())

Result: ⚠️ BUG FOUND
- Line 149: atoi() returns ANY integer, but validCustom is Bool
- Usage assumes boolean (0 or non-zero), but no range check
- Semantic bug: int-to-bool conversion without validation
```

---

## Mode Decision Guide

**Start with Mode 1. Only use Mode 2 if you explicitly need code context.**

| User Says... | What They Want | Which Mode | Cost | Speed | Example |
|---------------|---------|-----------|------|-------|---------|
| "Where is X?" | Location only | Mode 1 ✅ | 100 | <1s | "Where is Player class?" |
| "Find X" | Location only | Mode 1 ✅ | 100 | <1s | "Find enum signInToken" |
| "Show me X" | Location only | Mode 1 ✅ | 100 | <1s | "Show me UpdateHealth" |
| "Is X safe?" | Location + code analysis | Mode 2 | 250-350 | 2-3s | "Is validCustom safe?" |
| "Find bugs in X" | Location + code analysis | Mode 2 | 250-350 | 2-3s | "Find bugs in GameManager" |
| "Analyze X" | Location + code analysis | Mode 2 | 250-350 | 2-3s | "Analyze iterator patterns" |
| "What calls X?" | Location only | Mode 1 ✅ | 100 | <1s | "What calls startGame?" |
| "How is X used?" | Location + context | Mode 2 | 250-350 | 2-3s | "How is Player initialized?" |

**Key insight:** If you're just looking for a location (where, find, show, what calls), use Mode 1.  
Only switch to Mode 2 if you need to understand the code around that location.

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
Match: "Code" → mask: "0x001"
↓
Pass to search: groupMasks: ["0x001"]
```

**Examples:**
- "Search in Engine group" → Call get_groups, find Engine mask, pass to search
- "Search in Code,Tests groups" → Call get_groups, resolve both masks, pass array
- "Find in Scripts" → Call get_groups, match "Scripts", use its mask
- Invalid group → "Group 'XYZ' not found. Available groups: Code, Tests, Docs, Engine, Rendering"

**Caching Groups for Performance:**

Once `hawkeye_get_groups()` is called, **cache the groups mapping in context memory** for the remainder of the conversation:

```
First call: hawkeye_get_groups()
↓
Store in context: groups_cache = [
  {name: "Code", mask: "0x001"},
  {name: "Tests", mask: "0x002"},
  {name: "Docs", mask: "0x004"},
  {name: "Engine", mask: "0x008"},
  {name: "Rendering", mask: "0x010"}
]

Subsequent calls: Use groups_cache directly (NO new API call)
↓
User: "Now search in Engine group"
↓
Lookup: groups_cache → find "Engine" → mask: "0x008" ✓
↓
Pass to search: groupMasks: ["0x008"]
```

**Benefits:**
- ✅ Reduces API c