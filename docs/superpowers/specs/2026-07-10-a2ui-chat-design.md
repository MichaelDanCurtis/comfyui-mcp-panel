# A2UI in the Agent Panel Chat — Design Spec

**Date:** 2026-07-10
**Status:** Approved (design). Ready for implementation planning.
**Repos:** `comfyui-agent-panel` (frontend, primary) · `comfyui-mcp` (orchestrator: tools + persona)
**Builds on:** per-workflow sessions (2026-07-10 spec), the `cmcpSetChatSurface` A2UI seam, the persistent bridge client, the `/prompts` editable-persona system.

## Goal

Agents render **interactive UI inside the chat** — choice buttons, forms, cards, node-wiring
diagrams, charts — with a full interaction round-trip (user clicks → agent acts) and an
expand-to-wide/shrink-back choreography. Example mission: "help with my video workflow" → the
panel widens, a diagram shows how the VAE connects and why, the user taps one of three
options, the agent does the work, the panel shrinks back.

## Decisions (settled in design review)

1. **Protocol: Google A2UI** (open-sourced Jan 2026, Apache 2.0, v1.0-RC) — declarative JSON
   component trees, no arbitrary code by design. NOT AG-UI (that's transport, see Sequencing);
   NOT a bespoke DSL (nonstandard vocabulary nobody knows).
2. **Renderer: vendored `@a2ui/lit`** + `@a2ui/web_core` (message processor, data binding,
   state, incremental updates) as a **one-time, pinned, single-file ESM bundle** committed to
   `web/js/vendor/` — the panel stays build-free at serve time. Custom components register via
   the official custom-catalog API (extend `A2uiLitElement`, Zod-typed props).
   **Fallback plan** (if theming or size turns hostile in practice): hand-rolled vanilla
   whitelist renderer for the same subset — the emission channels and card lifecycle are
   renderer-agnostic, so swapping renderers does not change the rest of this spec.
3. **Custom catalog (ComfyUI extensions):**
   - `comfy:graph` — `{nodes:[{id,label,color?}], edges:[{from,to,label?}], direction?}` →
     WE compute layout and build the SVG (`createElementNS`, computed coordinates, labels via
     `textContent`). The showcase component for explaining node wiring.
   - `comfy:chart` — bar/line from `{series:[{label, values:[…]}], x?:[labels]}` → same
     we-draw-it pattern. For sampler comparisons, loss curves, VRAM plots.
4. **Emission: dual-channel, tool-primary.**
   - **Primary — panel tools `panel_ui_render` / `panel_ui_update`** (house `panel_*` naming;
     reach Claude/Codex/Grok/Gemini): `panel_ui_render(spec)` → validates against the schema,
     renders a card into the chat, returns `{card_id}` or a VALIDATION ERROR the agent can
     retry on. `panel_ui_update(card_id, spec)` → re-renders that card in place (incremental
     updates: progress, reactive forms).
   - **Fallback — fenced block** ` ```a2ui … ``` ` in ordinary text (Ollama-family backends,
     which never get panel tools): detected at message completion, rendered through the SAME
     pipeline. Malformed JSON degrades to a plain code block — never broken UI.
5. **Interaction round-trip: human-in-the-loop, visible.** A Button tap / form submit sends a
   **visible chat message** (human-readable line, forms serialized readably). Works identically
   on every backend, lands in thread history. After resolve the card goes **inert** (buttons
   disabled, chosen one highlighted). A ✕ dismisses without notifying the agent.
6. **Raw SVG/HTML from agents: never.** Agent-supplied markup executing in ComfyUI's origin is
   XSS-equivalent (one prompt-injected string in a quoted Civitai description → hostile code in
   the browser session). Visuals come from DATA via the comfy:* components. The sanctioned
   future escape hatch, if free-form is ever truly needed, is a **client-implemented sandboxed
   iframe portal with strict CSP** (the VS Code webview pattern) — a ComfyUI-client capability,
   never agent-originated markup. Not built in v1; documented so nobody ever "just innerHTMLs it."

## Architecture

### Files
- **NEW** `web/js/vendor/a2ui-lit.bundle.js` — pinned one-time esbuild of `@a2ui/lit`+deps
  (built by `scripts/build-a2ui-vendor.mjs`, committed; rebuild only on deliberate upgrades).
- **NEW** `web/js/cmcp-a2ui.js` — the panel-side A2UI module: schema guard + caps, the
  comfy:graph / comfy:chart custom catalog, card mount/update/resolve/dismiss lifecycle,
  theming CSS. Exports `renderA2UICard(spec, {onAction, onDismiss})` and
  `updateA2UICard(cardId, spec)`.
- `web/js/comfyui-mcp-panel.js` — fence detection in the message renderer; the
  `panel_ui_render`/`panel_ui_update` bridge-command executors (house `GRAPH_TOOL_EXECUTORS`
  pattern); thread persistence; surface choreography.
- `comfyui-mcp/src/orchestrator/panel-tools.ts` — register `panel_ui_render`/`panel_ui_update` panel tools
  (schema-validated at the tool layer; bridge command to the browser).
- `comfyui-mcp/src/orchestrator/index.ts` — persona (`PANEL_SYSTEM_APPEND`) gains an A2UI
  section: when to use cards (choices, confirmations, forms, wiring diagrams), the vocabulary,
  the fence fallback. Editable live via the `/prompts` editor (prompt-overrides).

### Data flow
1. Agent calls `panel_ui_render(spec)` (or emits a fence) → validation → bridge → panel
   executor → `renderA2UICard` appends the card to the chat log.
2. `surface:"wide"` on the spec → panel best-effort grows the ComfyUI sidebar splitter to
   ~min(60vw, 900px) (previous width remembered) via the `cmcpSetChatSurface` seam; restored
   on resolve/dismiss. Fails soft to inline if ComfyUI's DOM shape changes.
3. User taps a Button / submits → visible chat message (e.g. "→ Connect via TAESD preview";
   forms as a readable key: value block) → normal agent turn. Card → inert.
4. `panel_ui_update(card_id, spec)` → in-place re-render (web_core's incremental model).
5. Thread persistence: cards stored as `{role:"card", kind:"a2ui", spec, resolved, choice}` in
   the existing thread msgs → replay renders resolved cards inert with the choice shown.
   (Per-workflow sessions: cards belong to their workflow's thread like everything else.)

### Security model (the wall)
- Schema validation server-side at the tool layer AND client-side before render (fence path
  gets client-side only).
- Hard caps: ≤64 components/spec, ≤30 graph nodes, ≤8 series × 256 points/chart, bounded
  nesting depth (8). Over-cap → fail-soft "unsupported card" chip with expandable raw JSON.
- All agent strings render via text nodes; component types/attributes from validated enums;
  Image `src` restricted to ComfyUI-origin `/view`/`/api/view` URLs (+ existing blob pipeline).
- Unknown component types → fail-soft chip (forward-compatible with spec growth).
- The vendored renderer is pinned; upgrades are deliberate, reviewed events.

## Verification
1. **Fixtures:** a set of a2ui JSON specs (each core component, comfy:graph, comfy:chart,
   over-cap bomb, `<script>`-in-label, `javascript:` URL, unknown type) rendered live in the
   browser; assert DOM/neutralization for each.
2. **End-to-end (tool path):** ask a tool-capable agent for "3 options as a UI card" → card
   renders → click → visible round-trip message → agent responds → card inert → surface
   restored. Repeat with `surface:"wide"` for the choreography.
3. **Fallback path:** hand-inject an `a2ui` fence via a message → same render pipeline.
4. **Update path:** `panel_ui_update` on a live card changes it in place without duplication.
5. **Persistence:** reload + workflow-switch → cards replay inert with recorded choices.

## Sequencing & non-goals
- **AG-UI transport = the NEXT design cycle** (explicitly queued, high priority per user): an
  AG-UI gateway on the orchestrator translating bridge events ↔ AG-UI events — opens the panel
  to any AG-UI-speaking agent backend and the orchestrator to any AG-UI frontend. A2UI cards
  ride AG-UI events natively once it lands. Not in this build.
- Not in v1: token-level partial rendering of half-streamed cards (tearing/flicker; `panel_ui_update`
  covers live change), the sandboxed iframe portal (documented escape hatch only), theming
  beyond the panel's dark theme, A2UI audio/video components.

## Open unknowns (first implementation task = spike)
- Exact vendored bundle size and whether `@a2ui/lit` theming (shadow-DOM/CSS custom properties)
  can match the panel's dark theme with reasonable effort. **Spike task:** vendor the bundle,
  render 3 fixture cards in the panel, measure size + theming effort. If hostile → invoke the
  documented fallback renderer plan (decision 2) and proceed; the rest of the spec is unchanged.
