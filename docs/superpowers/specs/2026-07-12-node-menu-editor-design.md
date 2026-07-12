# Node Menu Editor — Design Spec

**Date:** 2026-07-12
**Status:** Approved (design). Ready for implementation planning.
**Repos:** `comfyui-agent-panel` (frontend: override extension, tree editor, panel tools — primary) · `comfyui-mcp` (orchestrator: AI cleanup tools + persona)
**Builds on:** the shipped A2UI cards (proposal review renders as an interactive before→after card), the panel's live-canvas `panel_*` tool surface + bridge, the per-workflow sessions plumbing, the existing credentials/sidebar-tab UI patterns.

## Goal

Let the user reorganize ComfyUI's add-node menus — the taxonomy that turns to chaos after installing many custom-node packs whose authors use emoji/vanity category names, junk display names, and inconsistent grouping. Two ways to edit, one shared override model: a **drag-and-drop tree editor** and a **conversational AI mass-cleanup**. Corrections never touch a pack's files and survive pack updates.

## The problem, quantified (live introspection of the user's install)

2702 registered node types, **411 distinct categories**, **38 top-level menu roots**. Clean native roots (`model`, `image`, `conditioning`) sit beside author-vanity junk: `crystools 🪛`, `🫶 ComfyAssets`, `😺dzNodes`, `大模型派对（llm_party）`, `RES4LYF`, `CivitaiAPI`, `RBG Suite`, … 449 categories carry emoji/whitespace/casing weirdness; 8 are blank. ComfyUI's built-in "bookmarks" only *add* favorite folders — they cannot re-parent or rename the real taxonomy, so this is new capability, not a reskin.

## Decisions (settled in design review)

1. **Override mechanism = `beforeRegisterNodeDef`.** A new ComfyUI extension (`app.registerExtension`) with a `beforeRegisterNodeDef(nodeType, nodeData)` hook rewrites `nodeData.category`, `nodeData.display_name`, and a hidden flag from our override map (keyed by stable node **type id**) BEFORE the node registers. Confirmed live: both menu surfaces — classic LiteGraph right-click/search AND the Vue Node Library sidebar — derive from that nodeData, so one hook fixes both. Re-applied every load ⇒ survives pack updates; a pack's files are never modified; a removed node leaves a dormant override.
2. **Persistence = ComfyUI native per-user setting** (`Comfy.NodeMenuEditor.*`), server-persisted. So the user's clean menu applies **even with the orchestrator/agent turned off** — the orchestrator is needed only for the AI cleanup, not for corrections to stick. The agent reads/writes the map via the panel bridge. Export/import makes a map portable between installs.
3. **Four operations (all selected; all reversible; all via the override layer):** recategorize (move/regroup nodes + whole categories, create/merge), rename display names + category labels, hide/banish (route off-menu to a "Hidden" shelf; still loadable in existing workflows; one-click unhide), favorites shelf (pinned top section; AI can seed it from `nodeFrequency`, which ComfyUI already tracks — confirmed live).
4. **Two editors, one shared override map:** a drag-drop tree editor (panel sidebar tab) and the conversational AI (existing agent chat). Neither is canonical — both mutate the same map and both apply live.
5. **AI review flow = targeted/conversational (primary).** The user points at specific offenders ("kill the emoji roots", "put all WAS nodes under utils", "these 3 packs are junk"); the agent previews a **scoped** before→after A2UI card (Apply / Tweak / Cancel) covering only the affected nodes, then writes the map. No mandatory whole-tree upfront review. The tree editor is the manual complement for fine tweaks.
6. **Cleanup goal is user-selected per run, not hardcoded.** Three named strategies the agent applies to a mass request: **function-first canonical** (remap every node into one coherent by-function tree, ignoring pack origin), **clean-in-place** (keep pack identity, strip vanity/emoji, normalize casing, fix blanks — no cross-pack moves), **hybrid** (general nodes → canonical tree; niche/proprietary nodes stay under a cleaned pack name). The user names the strategy in the request; the agent can also recommend one.

## Architecture

### Override model
`OverrideMap` = `{ [nodeTypeId]: { category?: string, displayName?: string, hidden?: boolean, favorite?: boolean } }` plus a top-level `{ favoritesOrder?: string[], version, updatedAt }`. Only fields that differ from the pack's original are stored (a sparse diff), so a reset is "delete the key" and an unmodified install has an empty map. The node's real type id is the stable key; category/display are presentation only — the type id and I/O contract are never touched, so existing workflows and node behavior are unaffected.

### Files
- **NEW** `comfyui-agent-panel/web/js/cmcp-node-menu.js` — the override extension: `registerExtension` + `beforeRegisterNodeDef` applying the map; load/save the map from the ComfyUI setting; the live-reapply routine (mutate already-registered types + refresh node-def store + rebuild search index; full reload as fallback); export/import; the "Hidden" shelf + favorites section wiring. Pure of orchestrator dependencies — corrections work agent-off.
- **NEW** `comfyui-agent-panel/web/js/cmcp-node-menu-tree.js` — the drag-drop tree editor sidebar tab (category tree, drag to re-parent, inline rename, right-click hide, star to favorite), reusing the sidebar-tab-guard mechanics already shipped for the chat/model-explorer tabs. Reads/writes the same OverrideMap.
- **MODIFY** `comfyui-agent-panel/web/js/comfyui-mcp-panel.js` — register the new sidebar tab; bridge handlers for the `panel_menu_*` tools (read registry + overrides, apply a proposed diff, reset); render the proposal review as an A2UI before→after card (reuse `renderA2UICard`).
- **MODIFY** `comfyui-mcp/src/orchestrator/panel-tools.ts` — register `panel_menu_read` (dump the current taxonomy + overrides + usage frequency for the agent), `panel_menu_propose` (validate + preview a scoped diff → A2UI card), `panel_menu_apply` (write the diff to the map), `panel_menu_reset`. Schema-validated at the tool layer.
- **MODIFY** `comfyui-mcp/src/orchestrator/index.ts` — persona (`panel.persona` via the editable /prompts registry) gains a Node-Menu section: the three cleanup strategies, when to use scoped vs broad proposals, that changes are reversible presentation-only overrides, and to always preview via a card before applying.

### Data flow (AI cleanup)
1. User: "clean up the emoji roots" (optionally naming a strategy). Agent calls `panel_menu_read` → gets the full `{typeId, category, displayName, pack, usageCount}[]` + current overrides.
2. Agent computes a **scoped diff** (only affected nodes) and calls `panel_menu_propose(diff, strategy)` → validated → bridge → panel renders a before→after A2UI card (per-node old→new category/name, hides, favorites) with Apply / Tweak / Cancel.
3. User taps Apply → `panel_menu_apply(diff)` merges into the OverrideMap → save to the setting → live-reapply → menus update. Tweak → the change round-trips as a chat message the agent refines. Cancel → nothing written.
4. Manual edits in the tree editor write the same map + live-reapply directly (no orchestrator needed).

### Live apply
After a map change: for each affected registered type, update its `category`/`display_name`/hidden state on the in-memory node def, refresh the Vue nodeDef store, and rebuild the search index so both surfaces reflect it without a reload. If any surface can't be refreshed in place, fall back to prompting a reload (the setting is already persisted, so a reload is lossless).

### Security / safety
- Overrides are presentation-only, keyed by stable type id; the pack's Python/JS is never edited.
- Fully reversible: "reset this node", "reset this pack", "reset all"; the Hidden shelf; export/import a map.
- Hidden nodes are removed from the *menu* only — a node already in a saved workflow still loads and runs (hiding ≠ uninstalling).
- Agent proposals always preview as a card before writing; `panel_menu_apply` validates the diff (every type id must exist in the live registry; unknown ids rejected) so a hallucinated node can't corrupt the map.
- The map is size-capped and schema-validated on load; a corrupt/oversized setting fails soft to "no overrides" rather than breaking the menu.

## Verification
1. **Override reflects in both surfaces:** set a category+display override for a known node; confirm the change in the classic add-node menu AND the Node Library sidebar.
2. **Survives reload + is agent-independent:** reload with the orchestrator stopped; overrides still applied.
3. **Live re-apply:** apply a change via the tree editor and via `panel_menu_apply`; menus update without a manual reload (or fall back to the reload prompt).
4. **Hide/unhide:** banish a node → gone from menu, still loads in an existing workflow → unhide restores it.
5. **Favorites:** star nodes → they appear in the pinned section; AI seed-from-frequency populates plausible picks.
6. **AI targeted cleanup end-to-end:** "kill the emoji roots" → scoped before→after card → Apply → emoji roots renamed/removed in the live menu; a second strategy ("function-first: move all upscaling nodes under image/upscale") produces a different, correct diff.
7. **Reset + export/import:** reset-all clears overrides; export then import on a fresh map reproduces the taxonomy.
8. **Corrupt-map safety:** a malformed setting value loads as empty overrides, menu intact.

## Sequencing & non-goals
- **First implementation task = SPIKE (the one real unknown):** confirm `beforeRegisterNodeDef` category/display/hidden overrides reflect in BOTH menu surfaces AND that a *live* re-apply (no reload) works, including hide (lean on the existing `__hidden__` root convention found live). If live re-apply is hostile, invoke the documented fallback (apply-on-next-reload) — the rest of the spec is unchanged.
- Not in v1: editing node I/O, slot names, colors, or defaults (this is menu taxonomy only); per-workflow menu profiles; sharing/importing community taxonomy presets (export/import of the user's own map IS in); auto-applying a cleanup without the preview card.
- Ties into the broader roadmap (model-council card, preview-before-apply graph edits, vision-critique loop) only by reusing the A2UI card surface; those remain separate cycles.

## Open unknowns (spike, Task 1)
- Whether the Vue Node Library sidebar picks up an in-place nodeData mutation without a store refresh call, and what that call is (nodeDef store action vs. re-register). Spike settles the exact live-reapply sequence; fallback is reload-to-apply.
- The cleanest hide mechanism (route to `__hidden__` root vs. a registry filter) — spike picks one and the design's "Hidden shelf" is built on it.
