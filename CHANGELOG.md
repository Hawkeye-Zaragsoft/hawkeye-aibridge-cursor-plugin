# Changelog

## [1.0.5] - 2026-09-28

### Added
- Unity/Unreal Engine projects now use Hawkeye search by default — the skill includes guidance for `.meta`/guid lookups, prefabs, ScriptableObjects, Blueprints, and `.uasset`/`.umap` files, on top of the existing C# code search.
- `hawkeye_reload_index` tool: triggers a headless Hawkeye index reload (no window, doesn't steal focus) when asked — e.g. "reload the index" or "it's not finding what I just added". Manual/on-request only; not triggered automatically on empty search results.

## [1.0.0] - 2026-06-04

### Initial Release
- Hawkeye AI Bridge MCP server plugin for Cursor
- Local code search with intelligent indexing
- ~91% token savings vs generic search
- Support for code groups and filters
- Integration with Cursor's agent system
- Includes hawkeye-search skill for automatic triggering
