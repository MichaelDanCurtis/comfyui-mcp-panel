# OAuth in the Agent Panel — Design Spec

**Date:** 2026-07-11
**Status:** Approved (design). Ready for implementation planning.
**Repos:** `comfyui-mcp` (orchestrator: OAuth engine, backends, bridge tools — primary) · `comfyui-agent-panel` (frontend: sign-in UI in the credentials card)
**Builds on:** `chatgpt-oauth-backend.ts` (existing Codex-OAuth precedent), `code-provider-auth.ts` (existing token refresh for OpenAI/Kimi), the native credentials card (`cmcpOpenCredentialsFrame`), `panel-secrets.ts` (`~/.comfyui-mcp/panel-secrets.json`, allowlisted), provider readiness/auto-pick + the provider on/off UI in the model popup.

## Goal

Let the user sign into **Grok (xAI)**, **Codex/ChatGPT (OpenAI)**, and **GitHub Copilot** directly from the panel — a first-party OAuth login that writes the same credential files the external CLIs use, so installing those CLIs becomes optional and the panel login substitutes for them. The orchestrator runs the flow; the panel triggers it and shows status.

## Decisions (settled in design review)

1. **Scope: all three providers; Copilot is EXPERIMENTAL.** Grok's OAuth is officially sanctioned for third-party clients. Codex is gray-area but already shipped in this fork (`chatgpt-oauth-backend.ts` reads `~/.codex/auth.json` and refreshes tokens). Copilot's only working token path authenticates **as VS Code's Copilot extension client id** (`Iv1.b507a08c87ecfe98`) to obtain a `ghu_` token — this runs against GitHub's Copilot API terms and is the most likely to break or get an account flagged. Copilot ships **off by default**, visually tagged experimental with a one-line risk note, and **isolated** so a GitHub-side change cannot destabilize Grok/Codex. This is for the user's own account on their own machine.
2. **Flow runs on the ORCHESTRATOR, not the panel.** It is a Node process that can bind a loopback callback port, open a browser, poll a device endpoint, and hold client state; the panel is browser JS in ComfyUI's origin that cannot do these cleanly. Matches existing precedent.
3. **Architecture: generic OAuth engine + per-provider config** (Approach A). One `oauth-flow.ts` implements two reusable primitives (loopback-PKCE, device-code); each provider is a small config object. Avoids triplicating the security-critical PKCE/state/allowlist logic (the lockstep-drift risk seen with the dual A2UI validators).
4. **Deployment target: LOCAL orchestrator.** Assume orchestrator == browser machine; use each provider's natural flow with loopback reachable. In-panel OAuth is simply not offered when the orchestrator is remote (documented, not engineered around). No remote-pod-specific handling in v1.
5. **Token landing: native write + panel mirror ("both").** OAuth writes the provider-native file the existing backends already read (source of truth for tokens). A **status-only** mirror (no token material) is recorded in `panel-secrets.json` for the readiness UI: `{provider, account_label, obtained_at, expires_at, experimental}`.
6. **UI: per-provider sign-in rows in the native credentials card.** Signed-out → "Sign in with X"; device flow → show `user_code` + copy-verification-URL + waiting state; signed-in → account label + "Sign out". Readiness/auto-pick and the provider on/off popup treat a provider as ready only once signed in.

## Per-provider facts (from research — verify at implementation)

| Provider | Flow | Endpoints | client_id | Token file | Notes |
|---|---|---|---|---|---|
| **Codex/ChatGPT** | Loopback-PKCE only (no device flow) | `auth.openai.com/oauth/authorize` + `/oauth/token` | `app_EMoamEEZ73f0CkXaXp7hrann` (public) | `~/.codex/auth.json` (`tokens.{access_token,refresh_token,account_id}`) | redirect **hard-locked** to `http://localhost:1455/auth/callback`. Token works ONLY against `chatgpt.com/backend-api/codex/responses` (already the backend's `CODEX_RESPONSES_URL`) with the `chatgpt-account-id` header; `account_id` from the `id_token` JWT `chatgpt_account_id` claim. Scopes `openid profile email offline_access`. Refresh already implemented in `code-provider-auth.ts`. |
| **Grok/xAI** | Loopback-PKCE (device endpoint also exists; v1 uses loopback per local-only decision) | OIDC discovery `auth.x.ai/.well-known/openid-configuration` → `/oauth2/authorize`, `/oauth2/token`, `/oauth2/device/code`, `/oauth2/revoke` | **RESOLVED (Task 1 GO): `b1a00492-073a-47ea-816f-4c329264a828`** — the **public Grok-CLI desktop client_id**. xAI offers no self-serve OAuth client registration (`registration_endpoint` absent from discovery), so the panel presents to xAI **as the Grok CLI** (uses the `grok-cli:access` scope). This is the same identity-reuse *mechanism* as Copilot's, but a **different risk class** — xAI sanctions third-party OAuth sign-in, so Grok is **first-class (not experimental/gated)**, unlike Copilot whose path is forbidden by GitHub's terms. Switch to an owned client_id if xAI later ships self-serve registration. | `~/.grok/auth.json` (community convention: `{access_token, refresh_token, expires_in}`) | S256 PKCE. API base `api.x.ai/v1`; confirmed scope set `openid profile email offline_access grok-cli:access api:access` returns `200` on `GET api.x.ai/v1/models` (Task 1). Endpoint allowlist: only send the bearer to `https://api.x.ai` / `*.x.ai`. |
| **GitHub Copilot** | Device flow (only path yielding a usable token) | GitHub device-code endpoints | VS Code Copilot extension id `Iv1.b507a08c87ecfe98` | Copilot token file (design: `~/.comfyui-mcp/copilot-auth.json`, our own — no external Copilot CLI convention to match) | Yields a `ghu_` token; PATs do not work. EXPERIMENTAL: authenticates as VS Code — ToS risk. Isolated behind its flag. |

## Architecture

### Files
- **NEW** `comfyui-mcp/src/services/oauth-flow.ts` — the engine. Exports `runLoopbackPKCE(cfg)`, `runDeviceCode(cfg)`, and the provider registry `OAUTH_PROVIDERS` (config objects). Owns PKCE (S256 verifier/challenge), `state` generation+check, the loopback HTTP listener (bound `127.0.0.1` only, fixed or provider-pinned port), device-code polling, and the per-provider HTTPS host allowlist for token delivery.
- **NEW** `comfyui-mcp/src/services/oauth-flow.test.ts` — vitest: PKCE pair correctness, state mismatch rejection, allowlist rejection of off-host token URLs, token-file read/write/refresh round-trips, device-code poll state machine. No live provider calls (fetch mocked).
- **MODIFY** `comfyui-mcp/src/services/code-provider-auth.ts` — extend the existing refresh machinery to cover the new token files (Grok `~/.grok/auth.json`, Copilot store); keep OpenAI/Kimi paths unchanged.
- **MODIFY** `comfyui-mcp/src/services/panel-secrets.ts` — add the status-only OAuth mirror slots (allowlisted keys; never store token material).
- **MODIFY** `comfyui-mcp/src/orchestrator/panel-tools.ts` (or the bridge command layer) — register `oauth_begin` / `oauth_status` / `oauth_signout` bridge commands.
- **MODIFY** the Grok + Codex backends — read the newly-written native files (Codex backend already does; Grok backend currently goes through the CLI/ACP — add a direct-token path that prefers `~/.grok/auth.json` when present, falling back to ACP/CLI).
- **NEW** `comfyui-mcp/src/orchestrator/copilot-backend.ts` — the experimental Copilot backend (isolated; only active when signed in + flag on).
- **MODIFY** `comfyui-agent-panel/web/js/comfyui-mcp-panel.js` — per-provider sign-in rows in the credentials card; wire `oauth_*` bridge commands; reflect login in readiness/provider-popup.

### Data flow
1. User clicks "Sign in with X" → panel sends `oauth_begin {provider}`.
2. Orchestrator runs the provider's flow:
   - **Loopback-PKCE (Codex, Grok):** bind `127.0.0.1:<port>`, generate PKCE + `state`, open the browser to the authorize URL, catch the callback, verify `state`, exchange code+verifier at the token endpoint. Returns `{mode:"loopback", opened:true}` immediately; completion arrives via `oauth_status`.
   - **Device-code (Copilot):** request a device code, return `{mode:"device", user_code, verification_url}` to the panel, poll the token endpoint until approved/expired.
3. On success: write the provider-native token file (0600), derive the account label (JWT claim / userinfo), write the status-only mirror to `panel-secrets.json`, re-probe readiness, push the updated model/provider list.
4. Panel `oauth_status {provider}` reflects signed-in (account label) / pending / signed-out; the credentials card and provider popup update.
5. `oauth_signout {provider}` deletes the native token file + clears the mirror + re-probes.
6. Ongoing: `code-provider-auth.ts` refreshes access tokens via stored refresh tokens, no browser interaction.

### Security model
- Tokens never enter the agent context or logs (existing secret discipline); the panel mirror holds **status only**, never token material.
- PKCE S256 everywhere; `state` generated + verified on every loopback exchange.
- Loopback listener bound to `127.0.0.1` exclusively, torn down immediately after the callback (or on timeout).
- Per-provider **HTTPS host allowlist**: the bearer is only ever sent to the provider's own API host(s) (e.g. Grok → `api.x.ai`/`*.x.ai`), so a spoofed/compromised OIDC discovery document can't redirect token delivery.
- Copilot isolated: its backend/flow failing (or GitHub revoking the client-id path) cannot affect Grok/Codex; it is off by default and labeled.
- Token files written `0600`.

## Sequencing & non-goals
- **v1 = LOCAL orchestrator only.** No remote-pod loopback tunneling; when the orchestrator is remote, in-panel OAuth is not offered (the external-CLI path remains).
- Grok v1 uses **loopback-PKCE**; wiring the advertised device endpoint (for future remote support) is a later cycle.
- Not in v1: provider-agnostic "add your own OAuth provider" config UI; encrypted-at-rest token storage beyond `0600`; multi-account per provider.
- Ties into the queued **AG-UI transport gateway** cycle only insofar as more backends may later want the same engine — the config-driven design accommodates that without rework.

## Open unknowns (first implementation task = spike)
- **Grok client_id** is not published in any source found. **Spike (Task 1):** obtain it (capture from a real grok-cli/Hermes loopback login, or register an xAI OAuth client), and empirically confirm which scopes the API surface needs (`api:access` vs the `openid…` set) against `api.x.ai/v1`. If neither a client_id nor self-serve registration is available, Grok in-panel OAuth is **NO-GO** and falls back to the existing CLI/ACP path — documented like the A2UI spike's off-ramp.
- Confirm the Copilot device-flow client-id path still yields a working `ghu_` token at build time (it is the most likely of the three to have changed).
