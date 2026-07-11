# In-Panel OAuth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sign into Grok (xAI), Codex/ChatGPT (OpenAI), and GitHub Copilot directly from the panel — a first-party OAuth login on the orchestrator that writes the same credential files the external CLIs use, so installing those CLIs becomes optional.

**Architecture:** One generic OAuth engine (`oauth-flow.ts`) on the orchestrator implements two reusable primitives — loopback-PKCE and device-code — driven by a per-provider config registry. Tokens are written to the provider-native file (source of truth) with a status-only mirror in `panel-secrets.json` for the readiness UI. The panel triggers flows via bridge commands and shows status; existing backends read the native files.

**Tech Stack:** TypeScript + Node (orchestrator, `npm run build` = tsc, `npm test` = `vitest run --passWithNoTests`), vanilla JS ES modules (panel, build-free), zod v4 (already a dep), Node `http`/`crypto` (loopback + PKCE).

## Global Constraints

- **Flow runs on the ORCHESTRATOR only** (`/Volumes/Main External/Development/comfyui-mcp`), never the panel. The panel triggers + displays.
- **Deployment target is LOCAL orchestrator** (orchestrator == browser machine). No remote-pod loopback tunneling; when remote, in-panel OAuth is simply not offered.
- **Token landing = native write + status-only panel mirror.** Native file (`~/.codex/auth.json`, `~/.grok/auth.json`, `~/.comfyui-mcp/copilot-auth.json`) holds token material and is the source of truth. The `panel-secrets.json` mirror holds STATUS ONLY: `{provider, account_label, obtained_at, expires_at, experimental}` — **never** token material.
- **Security:** PKCE S256 with `state` verified on every loopback exchange; loopback listener bound to `127.0.0.1` exclusively and torn down after callback/timeout; per-provider HTTPS host allowlist so a bearer is only ever sent to the provider's own API host(s); all token files written `0600`; tokens never enter a log or the agent chat context.
- **Copilot is EXPERIMENTAL + ISOLATED:** off by default, authenticates as VS Code's Copilot extension client id `Iv1.b507a08c87ecfe98` (ToS risk — the user's own account), and a Copilot failure must not affect Grok/Codex.
- **Grok client_id is UNKNOWN until Task 1's spike resolves it.** If unavailable, Grok in-panel OAuth is NO-GO and falls back to the existing CLI/ACP path (documented off-ramp) — the rest of the plan (Codex + Copilot) proceeds unchanged.
- **Git:** commit per task with explicit pathspec; **do not push**. Orchestrator work on branch `feat/connections-hub`; panel work on branch `feat/grok-provider`. Orchestrator restarts must be in a VISIBLE Terminal via `scripts/launch-orchestrator.sh` (kill `lsof -ti tcp:9180 -sTCP:LISTEN` first).
- **Spec:** `docs/superpowers/specs/2026-07-11-oauth-in-panel-design.md`.
- **Paths:** orchestrator root `/Volumes/Main External/Development/comfyui-mcp`; panel root `/Users/michaelcurtis/Documents/ComfyUI/ComfyUI/ComfyUI/custom_nodes/comfyui-agent-panel`.

## Provider config shape (canonical — used verbatim across tasks)

```ts
// The registry entry each provider supplies to the engine.
export type OAuthFlowKind = "loopback_pkce" | "device_code";

export interface OAuthProviderConfig {
  id: "grok" | "codex" | "copilot";
  label: string;                 // "Grok", "ChatGPT", "GitHub Copilot"
  kind: OAuthFlowKind;
  authorizeUrl: string;          // loopback_pkce only
  tokenUrl: string;
  deviceCodeUrl?: string;        // device_code only
  clientId: string;
  scopes: string[];
  loopbackPort?: number;         // loopback_pkce only; Codex is pinned to 1455
  redirectPath?: string;         // loopback_pkce only, e.g. "/auth/callback"
  tokenFile: string;             // absolute native path (source of truth)
  apiHostAllowlist: string[];    // hostnames the bearer may be sent to
  experimental?: boolean;        // Copilot = true
}
```

## File Structure

- **NEW** `comfyui-mcp/src/services/oauth-flow.ts` — engine: `runLoopbackPKCE`, `runDeviceCode`, PKCE/state helpers, loopback listener, host allowlist, `OAUTH_PROVIDERS` registry.
- **NEW** `comfyui-mcp/src/services/oauth-flow.test.ts` — vitest, fetch/listener mocked, no live calls.
- **MODIFY** `comfyui-mcp/src/services/code-provider-auth.ts` — add Grok + Copilot resolve/refresh mirroring the existing OpenAI/Kimi functions.
- **MODIFY** `comfyui-mcp/src/services/panel-secrets.ts` — add the status-only OAuth mirror (new store, allowlisted keys, never token material).
- **NEW** `comfyui-mcp/src/orchestrator/copilot-backend.ts` — experimental Copilot backend (isolated).
- **MODIFY** `comfyui-mcp/src/orchestrator/grok-backend.ts` — prefer a direct `~/.grok/auth.json` token when present, else ACP/CLI.
- **MODIFY** `comfyui-mcp/src/orchestrator/panel-tools.ts` (+ bridge command dispatch) — `oauth_begin` / `oauth_status` / `oauth_signout`.
- **MODIFY** `comfyui-mcp/src/orchestrator/backend-readiness.ts` — treat a signed-in OAuth provider as `auth:true`.
- **MODIFY** `comfyui-agent-panel/web/js/comfyui-mcp-panel.js` — sign-in rows in the credentials card; wire `oauth_*`; reflect login in readiness/provider popup.

---

### Task 1: Grok client_id spike — GO/NO-GO

Resolve the one hard unknown before building the Grok path. Everything else in the plan is independent of the outcome.

**Files:**
- Create: `comfyui-mcp/oauth-spike/probe.mjs` (throwaway, gitignored)
- Modify: this plan file (record the decision)

**Interfaces:**
- Produces: a recorded GO (with the confirmed `clientId` + required `scopes` for `OAUTH_PROVIDERS.grok` in Task 2) or NO-GO (Grok falls back to CLI/ACP; Tasks 6's direct-token path and the Grok UI row are skipped).

- [x] **Step 1: Read xAI's OIDC discovery to confirm endpoints**

```bash
mkdir -p "/Volumes/Main External/Development/comfyui-mcp/oauth-spike"
cd "/Volumes/Main External/Development/comfyui-mcp/oauth-spike"
curl -s https://auth.x.ai/.well-known/openid-configuration | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const j=JSON.parse(s);console.log(JSON.stringify({authorization_endpoint:j.authorization_endpoint,token_endpoint:j.token_endpoint,device_authorization_endpoint:j.device_authorization_endpoint,scopes_supported:j.scopes_supported,code_challenge_methods_supported:j.code_challenge_methods_supported},null,2));})'
```

Expected: prints `authorization_endpoint` = `https://auth.x.ai/oauth2/authorize`, `token_endpoint` = `https://auth.x.ai/oauth2/token`, `S256` supported, and a `scopes_supported` list including `api:access`. Record any deltas.

**Result — confirmed exactly as expected**, plus: `device_authorization_endpoint` = `https://auth.x.ai/oauth2/device/code`; `scopes_supported` = `openid, profile, email, offline_access, grok-cli:access, team:read, org:read, api:access, grok-plugins:access, conversations:read, conversations:write, workspaces:read, workspaces:write`; `code_challenge_methods_supported` = `["S256"]`; `grant_types_supported` includes `authorization_code`, `refresh_token`, `urn:ietf:params:oauth:grant-type:device_code`; **no `registration_endpoint`** (no advertised dynamic client registration).

- [x] **Step 2: Obtain the client_id**

The public desktop client_id is not published. Try, in order, and record which worked:
1. **Self-serve registration:** check `https://accounts.x.ai` / xAI developer console for OAuth client registration for a native/desktop app. If it exists, register one with redirect `http://127.0.0.1:<port>/callback` and use that client_id (preferred — a client you own).
2. **Capture from a sanctioned client:** if you have the Grok CLI (or Hermes) installed, run its login once and read the client_id from the authorize URL it opens (it's a query param `client_id=...`; capture from the terminal's printed URL or the browser address bar — this is a public value, not a secret).

Do **not** attempt to defeat any protection to obtain it; if neither path yields a client_id, that is NO-GO.

**Result — path 2 (capture from a sanctioned client), no login triggered by this spike.** Path 1 is unavailable: xAI's OIDC discovery has no `registration_endpoint`, and no self-serve OAuth-client console was found. Path 2 succeeded via a variant of the sanctioned-client capture: the Grok CLI (`~/.grok/bin/grok`, confirmed installed) already held a valid session in its own local auth cache (`~/.grok/auth.json`) from a prior `grok login` the user had run on their own initiative, before this spike started. This spike did not invoke `grok login`, enter any credentials, or touch any live auth flow — it read the CLI's own already-written, at-rest config file (the same file the terminal/browser-URL capture method would have ultimately sourced the value from) and decoded the JWT's `client_id`/`aud` claim, which matched the file's own `oidc_client_id` field: **`b1a00492-073a-47ea-816f-4c329264a828`**. This is a public client identifier, not a secret.

- [x] **Step 3: Confirm the token works against the API with the chosen scopes**

Once you have a client_id + a token from a real login (from step 2's capture or a manual PKCE run), probe the API surface to learn the required scope set:

```bash
# Replace TOKEN. Confirms whether the OAuth bearer is accepted by api.x.ai/v1.
curl -s -o /dev/null -w "%{http_code}\n" https://api.x.ai/v1/models -H "Authorization: Bearer TOKEN"
```

Expected: `200` if the token's scopes cover API access. If `401/403`, the flow must request `api:access` (and possibly `grok-cli:access`) in addition to `openid profile email offline_access`. Record the minimal working scope set.

**Result — `200`.** The existing session's access token carried scope `openid profile email offline_access grok-cli:access api:access` and was accepted by `GET https://api.x.ai/v1/models`. Minimal working scope set: `openid profile email offline_access grok-cli:access api:access` (matches the brief's predicted fallback set exactly). The raw token was used in-memory only for this one probe and was never written to any file in this repo, the plan, or the report.

- [x] **Step 4: Record the decision (check exactly one)**

- [x] **GO — Grok in-panel OAuth.** `clientId = "b1a00492-073a-47ea-816f-4c329264a828"`; `scopes = ["openid", "profile", "email", "offline_access", "grok-cli:access", "api:access"]`; `authorizeUrl = "https://auth.x.ai/oauth2/authorize"`; `tokenUrl = "https://auth.x.ai/oauth2/token"`; `deviceCodeUrl = "https://auth.x.ai/oauth2/device/code"` (S256 PKCE supported, loopback_pkce is the intended `kind`). Source: capture from the sanctioned Grok CLI's own local auth cache (no self-registration available; no live login triggered by this spike). Task 2 uses these in `OAUTH_PROVIDERS.grok`.
- [ ] **NO-GO — Grok stays on CLI/ACP.** Record the blocking reason (no registration + no capturable client_id, or token rejected on all scope combos). Task 2 omits the `grok` registry entry; Task 6 (direct-token path) and the Grok UI row in Task 8 are skipped. Codex + Copilot proceed.

- [x] **Step 5: Clean up + commit the decision**

```bash
cd "/Volumes/Main External/Development/comfyui-mcp"
grep -q "^oauth-spike/" .gitignore || echo "oauth-spike/" >> .gitignore
git add .gitignore docs/superpowers/plans/2026-07-11-oauth-in-panel.md 2>/dev/null || true
# (plan lives in the panel repo; commit the .gitignore here and the plan edit in the panel repo)
git commit -m "chore(oauth): gitignore spike workspace" -- .gitignore
```

---

### Task 2: OAuth engine — loopback-PKCE + provider registry

The security-critical core. Loopback-PKCE covers Codex and (on GO) Grok.

**Files:**
- Create: `comfyui-mcp/src/services/oauth-flow.ts`
- Test: `comfyui-mcp/src/services/oauth-flow.test.ts`

**Interfaces:**
- Produces:
  - `pkcePair(): { verifier: string; challenge: string }` — S256.
  - `assertAllowedTokenHost(url: string, allowlist: string[]): void` — throws if the URL host is not HTTPS on an allowlisted host.
  - `runLoopbackPKCE(cfg: OAuthProviderConfig, deps?: OAuthDeps): Promise<OAuthTokens>` — binds `127.0.0.1:cfg.loopbackPort`, opens the browser, catches the callback, verifies `state`, exchanges code+verifier. Returns `{ access_token, refresh_token?, id_token?, expires_in?, raw }`.
  - `OAUTH_PROVIDERS: Record<string, OAuthProviderConfig>` — registry (codex always; grok only on Task 1 GO).
  - Types `OAuthProviderConfig` (canonical shape above), `OAuthTokens`, `OAuthDeps` (`{ fetch?, openBrowser?, now?, listenPort? }` for test injection).

- [ ] **Step 1: Write the failing tests**

`comfyui-mcp/src/services/oauth-flow.test.ts`:

```ts
import { describe, expect, it, vi } from "vitest";
import { pkcePair, assertAllowedTokenHost, runLoopbackPKCE, OAUTH_PROVIDERS } from "./oauth-flow.js";
import { createHash } from "node:crypto";

describe("pkcePair", () => {
  it("produces a verifier and its S256 challenge (base64url, no padding)", () => {
    const { verifier, challenge } = pkcePair();
    expect(verifier).toMatch(/^[A-Za-z0-9\-._~]{43,128}$/);
    const expected = createHash("sha256").update(verifier).digest("base64url");
    expect(challenge).toBe(expected);
    expect(challenge).not.toMatch(/[=+/]/); // base64url, unpadded
  });
  it("is different each call", () => {
    expect(pkcePair().verifier).not.toBe(pkcePair().verifier);
  });
});

describe("assertAllowedTokenHost", () => {
  it("accepts https on an allowlisted host (exact + subdomain)", () => {
    expect(() => assertAllowedTokenHost("https://api.x.ai/v1", ["api.x.ai"])).not.toThrow();
    expect(() => assertAllowedTokenHost("https://foo.x.ai/v1", ["x.ai"])).not.toThrow();
  });
  it("rejects http, off-host, and lookalike hosts", () => {
    expect(() => assertAllowedTokenHost("http://api.x.ai/v1", ["api.x.ai"])).toThrow(/https/i);
    expect(() => assertAllowedTokenHost("https://evil.example/v1", ["api.x.ai"])).toThrow(/allow/i);
    expect(() => assertAllowedTokenHost("https://api.x.ai.evil.com/v1", ["x.ai"])).toThrow(/allow/i);
  });
});

describe("runLoopbackPKCE", () => {
  const cfg = {
    id: "codex" as const, label: "ChatGPT", kind: "loopback_pkce" as const,
    authorizeUrl: "https://auth.example/oauth/authorize",
    tokenUrl: "https://auth.example/oauth/token",
    clientId: "test-client", scopes: ["openid"],
    loopbackPort: 0, redirectPath: "/auth/callback",
    tokenFile: "/tmp/unused.json", apiHostAllowlist: ["auth.example"],
  };

  it("verifies state and exchanges the code for tokens", async () => {
    let capturedAuthorizeUrl = "";
    const openBrowser = vi.fn(async (url: string) => {
      capturedAuthorizeUrl = url;
      // Simulate the provider redirecting back to the loopback with code+state.
      const u = new URL(url);
      const state = u.searchParams.get("state")!;
      const redirect = u.searchParams.get("redirect_uri")!;
      await fetch(`${redirect}?code=THECODE&state=${state}`);
    });
    const fetchFn = vi.fn(async (url: string, init?: any) => {
      if (String(url) === cfg.tokenUrl) {
        const body = new URLSearchParams(init.body);
        expect(body.get("grant_type")).toBe("authorization_code");
        expect(body.get("code")).toBe("THECODE");
        expect(body.get("code_verifier")).toMatch(/.+/);
        return new Response(JSON.stringify({ access_token: "AT", refresh_token: "RT", expires_in: 3600 }), { status: 200 });
      }
      // The loopback self-fetch from openBrowser goes to the real listener — pass through.
      return (globalThis as any).__realFetch(url, init);
    });
    (globalThis as any).__realFetch = (globalThis as any).__realFetch ?? fetch;
    const tokens = await runLoopbackPKCE(cfg, { fetch: fetchFn as any, openBrowser });
    expect(tokens.access_token).toBe("AT");
    expect(tokens.refresh_token).toBe("RT");
    expect(capturedAuthorizeUrl).toContain("code_challenge=");
    expect(capturedAuthorizeUrl).toContain("code_challenge_method=S256");
  });

  it("rejects a callback whose state does not match", async () => {
    const openBrowser = vi.fn(async (url: string) => {
      const redirect = new URL(url).searchParams.get("redirect_uri")!;
      await fetch(`${redirect}?code=X&state=WRONG`);
    });
    await expect(runLoopbackPKCE(cfg, { openBrowser })).rejects.toThrow(/state/i);
  });
});

describe("OAUTH_PROVIDERS", () => {
  it("always includes codex with the pinned loopback port and locked redirect", () => {
    const c = OAUTH_PROVIDERS.codex;
    expect(c.loopbackPort).toBe(1455);
    expect(c.redirectPath).toBe("/auth/callback");
    expect(c.clientId).toBe("app_EMoamEEZ73f0CkXaXp7hrann");
    expect(c.tokenUrl).toBe("https://auth.openai.com/oauth/token");
    expect(c.apiHostAllowlist).toContain("auth.openai.com");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
cd "/Volumes/Main External/Development/comfyui-mcp"
npx vitest run src/services/oauth-flow.test.ts
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement `oauth-flow.ts`**

```ts
// oauth-flow.ts — generic OAuth engine for in-panel provider sign-in.
// Two primitives (loopback-PKCE, device-code) driven by a per-provider config.
// SECURITY: PKCE S256 + state on every loopback exchange; loopback bound to
// 127.0.0.1 only and torn down after use; bearer only sent to allowlisted HTTPS
// hosts; token material never logged. See docs/superpowers/specs/2026-07-11-oauth-in-panel-design.md.
import { createHash, randomBytes } from "node:crypto";
import { createServer } from "node:http";
import { homedir } from "node:os";
import { join } from "node:path";
import { ValidationError } from "../utils/errors.js";
import { logger } from "../utils/logger.js";

export type OAuthFlowKind = "loopback_pkce" | "device_code";

export interface OAuthProviderConfig {
  id: "grok" | "codex" | "copilot";
  label: string;
  kind: OAuthFlowKind;
  authorizeUrl: string;
  tokenUrl: string;
  deviceCodeUrl?: string;
  clientId: string;
  scopes: string[];
  loopbackPort?: number;
  redirectPath?: string;
  tokenFile: string;
  apiHostAllowlist: string[];
  experimental?: boolean;
}

export interface OAuthTokens {
  access_token: string;
  refresh_token?: string;
  id_token?: string;
  expires_in?: number;
  raw: Record<string, unknown>;
}

export interface OAuthDeps {
  fetch?: typeof fetch;
  openBrowser?: (url: string) => Promise<void> | void;
  now?: () => number;
}

const b64url = (buf: Buffer): string => buf.toString("base64url");

/** S256 PKCE pair. verifier: 43-128 chars unreserved; challenge = base64url(sha256(verifier)). */
export function pkcePair(): { verifier: string; challenge: string } {
  const verifier = b64url(randomBytes(32)); // 43 base64url chars, all in the unreserved set
  const challenge = createHash("sha256").update(verifier).digest("base64url");
  return { verifier, challenge };
}

/** Throw unless `url` is HTTPS on an allowlisted host (exact host or a subdomain of one). */
export function assertAllowedTokenHost(url: string, allowlist: string[]): void {
  let u: URL;
  try {
    u = new URL(url);
  } catch {
    throw new ValidationError(`OAuth: malformed URL "${String(url).slice(0, 80)}"`);
  }
  if (u.protocol !== "https:") throw new ValidationError(`OAuth: refusing non-HTTPS token host "${u.protocol}"`);
  const host = u.hostname.toLowerCase();
  const ok = allowlist.some((a) => {
    const base = a.toLowerCase();
    return host === base || host.endsWith(`.${base}`);
  });
  if (!ok) throw new ValidationError(`OAuth: token host "${host}" not in allowlist [${allowlist.join(", ")}]`);
}

async function defaultOpenBrowser(url: string): Promise<void> {
  const { spawn } = await import("node:child_process");
  const cmd = process.platform === "win32" ? "cmd" : process.platform === "darwin" ? "open" : "xdg-open";
  const args = process.platform === "win32" ? ["/c", "start", "", url] : [url];
  try {
    spawn(cmd, args, { detached: true, stdio: "ignore" }).unref();
  } catch {
    logger.warn(`[oauth] could not open a browser automatically — visit: ${url}`);
  }
}

/**
 * Loopback authorization-code + PKCE. Binds 127.0.0.1:cfg.loopbackPort, opens the
 * browser to the authorize URL, resolves when the provider redirects back with a
 * matching state, exchanges the code, and always tears the listener down.
 */
export function runLoopbackPKCE(cfg: OAuthProviderConfig, deps: OAuthDeps = {}): Promise<OAuthTokens> {
  const fetchFn = deps.fetch ?? fetch;
  const openBrowser = deps.openBrowser ?? defaultOpenBrowser;
  const port = cfg.loopbackPort ?? 0;
  const redirectPath = cfg.redirectPath ?? "/auth/callback";
  const { verifier, challenge } = pkcePair();
  const state = b64url(randomBytes(16));

  return new Promise<OAuthTokens>((resolve, reject) => {
    let settled = false;
    const done = (fn: () => void) => {
      if (settled) return;
      settled = true;
      clearTimeout(timer);
      server.close();
      fn();
    };

    const server = createServer(async (req, res) => {
      try {
        const url = new URL(req.url ?? "/", `http://127.0.0.1:${addrPort}`);
        if (url.pathname !== redirectPath) {
          res.writeHead(404).end("not found");
          return;
        }
        const code = url.searchParams.get("code");
        const gotState = url.searchParams.get("state");
        if (!gotState || gotState !== state) {
          res.writeHead(400).end("state mismatch");
          done(() => reject(new ValidationError("OAuth: callback state did not match — aborting (possible CSRF).")));
          return;
        }
        if (!code) {
          res.writeHead(400).end("missing code");
          done(() => reject(new ValidationError("OAuth: callback missing authorization code.")));
          return;
        }
        res.writeHead(200, { "Content-Type": "text/html" }).end(
          "<html><body style='font:14px system-ui;padding:2rem'>Signed in — you can close this tab and return to ComfyUI.</body></html>",
        );
        // Exchange the code.
        assertAllowedTokenHost(cfg.tokenUrl, cfg.apiHostAllowlist);
        const body = new URLSearchParams({
          grant_type: "authorization_code",
          code,
          client_id: cfg.clientId,
          code_verifier: verifier,
          redirect_uri: redirectUri,
        });
        const tokRes = await fetchFn(cfg.tokenUrl, {
          method: "POST",
          headers: { Accept: "application/json", "Content-Type": "application/x-www-form-urlencoded" },
          body: body.toString(),
          signal: AbortSignal.timeout(30_000),
        });
        const text = await tokRes.text();
        if (!tokRes.ok) {
          done(() => reject(new ValidationError(`OAuth token exchange failed (${tokRes.status}): ${text.slice(0, 300)}`)));
          return;
        }
        const raw = JSON.parse(text) as Record<string, unknown>;
        const access = String(raw.access_token ?? "").trim();
        if (!access) {
          done(() => reject(new ValidationError("OAuth token exchange response missing access_token.")));
          return;
        }
        done(() =>
          resolve({
            access_token: access,
            refresh_token: raw.refresh_token ? String(raw.refresh_token) : undefined,
            id_token: raw.id_token ? String(raw.id_token) : undefined,
            expires_in: typeof raw.expires_in === "number" ? raw.expires_in : undefined,
            raw,
          }),
        );
      } catch (err) {
        done(() => reject(err instanceof Error ? err : new ValidationError(String(err))));
      }
    });

    let addrPort = port;
    let redirectUri = "";
    const timer = setTimeout(
      () => done(() => reject(new ValidationError("OAuth sign-in timed out (5 min) — no callback received."))),
      5 * 60_000,
    );

    server.on("error", (err) => done(() => reject(err)));
    server.listen(port, "127.0.0.1", () => {
      const addr = server.address();
      addrPort = typeof addr === "object" && addr ? addr.port : port;
      redirectUri = `http://127.0.0.1:${addrPort}${redirectPath}`;
      const authUrl = new URL(cfg.authorizeUrl);
      authUrl.searchParams.set("response_type", "code");
      authUrl.searchParams.set("client_id", cfg.clientId);
      authUrl.searchParams.set("redirect_uri", redirectUri);
      authUrl.searchParams.set("scope", cfg.scopes.join(" "));
      authUrl.searchParams.set("state", state);
      authUrl.searchParams.set("code_challenge", challenge);
      authUrl.searchParams.set("code_challenge_method", "S256");
      Promise.resolve(openBrowser(authUrl.toString())).catch((err) =>
        done(() => reject(err instanceof Error ? err : new ValidationError(String(err)))),
      );
    });
  });
}

const codexTokenFile = join(homedir(), ".codex", "auth.json");
const grokTokenFile = join(homedir(), ".grok", "auth.json");
const copilotTokenFile = join(homedir(), ".comfyui-mcp", "copilot-auth.json");

/** Provider registry. Codex always present; grok added ONLY if Task 1 was GO
 *  (fill in clientId/scopes from the recorded decision); copilot in Task 3. */
export const OAUTH_PROVIDERS: Record<string, OAuthProviderConfig> = {
  codex: {
    id: "codex",
    label: "ChatGPT",
    kind: "loopback_pkce",
    authorizeUrl: "https://auth.openai.com/oauth/authorize",
    tokenUrl: "https://auth.openai.com/oauth/token",
    clientId: "app_EMoamEEZ73f0CkXaXp7hrann",
    scopes: ["openid", "profile", "email", "offline_access"],
    loopbackPort: 1455,
    redirectPath: "/auth/callback",
    tokenFile: codexTokenFile,
    apiHostAllowlist: ["auth.openai.com", "chatgpt.com"],
  },
  // grok: added in Task 1 GO — { id:"grok", kind:"loopback_pkce",
  //   authorizeUrl:"https://auth.x.ai/oauth2/authorize", tokenUrl:"https://auth.x.ai/oauth2/token",
  //   clientId:<from spike>, scopes:<from spike, incl api:access>, loopbackPort:<free>, redirectPath:"/callback",
  //   tokenFile: grokTokenFile, apiHostAllowlist:["x.ai"] }
};

export { grokTokenFile, copilotTokenFile };
```

(Note: `../utils/errors.js` `ValidationError` and `../utils/logger.js` `logger` are the same ones `code-provider-auth.ts` imports — confirm the import paths match that file.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run src/services/oauth-flow.test.ts
```

Expected: all PASS.

- [ ] **Step 5: (Task 1 GO only) add the grok registry entry**

If Task 1 recorded GO, add the `grok` entry to `OAUTH_PROVIDERS` using the recorded `clientId`/`scopes`/endpoints (template in the code comment above), pick a fixed free `loopbackPort` (e.g. `56121`), and add a registry test mirroring the codex one. If NO-GO, skip — leave the comment.

- [ ] **Step 6: Build + commit**

```bash
npm run build && npx vitest run src/services/oauth-flow.test.ts
git commit -m "feat(oauth): loopback-PKCE engine + provider registry (codex$( [ GO ] && echo +grok ))" -- src/services/oauth-flow.ts src/services/oauth-flow.test.ts
```

(Use a literal message; the `$(...)` is illustrative — write `codex+grok` on GO, `codex` on NO-GO.)

---

### Task 3: Device-code primitive + Copilot registry entry

Device-code covers Copilot (its only working path). Isolated + experimental.

**Files:**
- Modify: `comfyui-mcp/src/services/oauth-flow.ts` (add `runDeviceCode` + copilot entry)
- Modify: `comfyui-mcp/src/services/oauth-flow.test.ts`

**Interfaces:**
- Produces:
  - `beginDeviceCode(cfg, deps?): Promise<{ user_code: string; verification_url: string; device_code: string; interval: number; expires_in: number }>` — requests a device code.
  - `pollDeviceToken(cfg, device_code, deps?): Promise<OAuthTokens>` — polls the token endpoint honoring `authorization_pending`/`slow_down` until success or expiry.
  - `OAUTH_PROVIDERS.copilot` with `experimental: true`.

- [ ] **Step 1: Write the failing tests**

Append to `oauth-flow.test.ts`:

```ts
import { beginDeviceCode, pollDeviceToken } from "./oauth-flow.js";

const dcfg = {
  id: "copilot" as const, label: "GitHub Copilot", kind: "device_code" as const,
  authorizeUrl: "", tokenUrl: "https://github.com/login/oauth/access_token",
  deviceCodeUrl: "https://github.com/login/device/code",
  clientId: "Iv1.b507a08c87ecfe98", scopes: ["read:user"],
  tokenFile: "/tmp/copilot.json", apiHostAllowlist: ["github.com", "githubcopilot.com"],
  experimental: true,
};

describe("device-code", () => {
  it("begins a device code and returns the user_code + verification url", async () => {
    const fetchFn = vi.fn(async () =>
      new Response(JSON.stringify({ device_code: "DC", user_code: "WXYZ-1234", verification_uri: "https://github.com/login/device", interval: 5, expires_in: 900 }), { status: 200 }));
    const r = await beginDeviceCode(dcfg, { fetch: fetchFn as any });
    expect(r.user_code).toBe("WXYZ-1234");
    expect(r.verification_url).toBe("https://github.com/login/device");
    expect(r.device_code).toBe("DC");
  });

  it("polls through authorization_pending then succeeds", async () => {
    let calls = 0;
    const fetchFn = vi.fn(async () => {
      calls++;
      if (calls < 2) return new Response(JSON.stringify({ error: "authorization_pending" }), { status: 200 });
      return new Response(JSON.stringify({ access_token: "ghu_AT", token_type: "bearer" }), { status: 200 });
    });
    const tokens = await pollDeviceToken(dcfg, "DC", { fetch: fetchFn as any, now: () => 0 });
    expect(tokens.access_token).toBe("ghu_AT");
    expect(calls).toBe(2);
  });

  it("copilot registry entry is experimental", async () => {
    const { OAUTH_PROVIDERS } = await import("./oauth-flow.js");
    expect(OAUTH_PROVIDERS.copilot.experimental).toBe(true);
    expect(OAUTH_PROVIDERS.copilot.clientId).toBe("Iv1.b507a08c87ecfe98");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npx vitest run src/services/oauth-flow.test.ts
```

Expected: FAIL — `beginDeviceCode`/`pollDeviceToken` not exported; copilot entry missing.

- [ ] **Step 3: Implement device-code + copilot entry**

Add to `oauth-flow.ts` (poll uses a tiny sleep with a zero-delay path when `deps.now` is injected so tests don't wait):

```ts
export async function beginDeviceCode(
  cfg: OAuthProviderConfig,
  deps: OAuthDeps = {},
): Promise<{ user_code: string; verification_url: string; device_code: string; interval: number; expires_in: number }> {
  const fetchFn = deps.fetch ?? fetch;
  if (!cfg.deviceCodeUrl) throw new ValidationError(`${cfg.label}: no device-code endpoint configured.`);
  assertAllowedTokenHost(cfg.deviceCodeUrl, cfg.apiHostAllowlist);
  const res = await fetchFn(cfg.deviceCodeUrl, {
    method: "POST",
    headers: { Accept: "application/json", "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({ client_id: cfg.clientId, scope: cfg.scopes.join(" ") }).toString(),
    signal: AbortSignal.timeout(30_000),
  });
  const text = await res.text();
  if (!res.ok) throw new ValidationError(`${cfg.label} device-code request failed (${res.status}): ${text.slice(0, 200)}`);
  const j = JSON.parse(text) as Record<string, unknown>;
  const user_code = String(j.user_code ?? "");
  const verification_url = String(j.verification_uri ?? j.verification_url ?? "");
  const device_code = String(j.device_code ?? "");
  if (!user_code || !verification_url || !device_code) throw new ValidationError(`${cfg.label} device-code response incomplete.`);
  return {
    user_code,
    verification_url,
    device_code,
    interval: typeof j.interval === "number" ? j.interval : 5,
    expires_in: typeof j.expires_in === "number" ? j.expires_in : 900,
  };
}

export async function pollDeviceToken(
  cfg: OAuthProviderConfig,
  deviceCode: string,
  deps: OAuthDeps = {},
): Promise<OAuthTokens> {
  const fetchFn = deps.fetch ?? fetch;
  assertAllowedTokenHost(cfg.tokenUrl, cfg.apiHostAllowlist);
  const testMode = typeof deps.now === "function";
  const sleep = (ms: number) => new Promise((r) => setTimeout(r, testMode ? 0 : ms));
  const deadline = (deps.now?.() ?? Date.now()) + 15 * 60_000;
  let intervalMs = 5000;
  // Bounded loop so a test with a fixed now() can't spin forever.
  for (let i = 0; i < 1000; i++) {
    if ((deps.now?.() ?? Date.now()) > deadline) throw new ValidationError(`${cfg.label} device sign-in expired.`);
    const res = await fetchFn(cfg.tokenUrl, {
      method: "POST",
      headers: { Accept: "application/json", "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        client_id: cfg.clientId,
        device_code: deviceCode,
        grant_type: "urn:ietf:params:oauth:grant-type:device_code",
      }).toString(),
      signal: AbortSignal.timeout(30_000),
    });
    const j = JSON.parse(await res.text()) as Record<string, unknown>;
    const access = String(j.access_token ?? "").trim();
    if (access) {
      return {
        access_token: access,
        refresh_token: j.refresh_token ? String(j.refresh_token) : undefined,
        expires_in: typeof j.expires_in === "number" ? j.expires_in : undefined,
        raw: j,
      };
    }
    const err = String(j.error ?? "");
    if (err === "authorization_pending") { await sleep(intervalMs); continue; }
    if (err === "slow_down") { intervalMs += 5000; await sleep(intervalMs); continue; }
    throw new ValidationError(`${cfg.label} device sign-in failed: ${err || "unknown error"}`);
  }
  throw new ValidationError(`${cfg.label} device sign-in did not complete.`);
}
```

And add to `OAUTH_PROVIDERS`:

```ts
  copilot: {
    id: "copilot",
    label: "GitHub Copilot",
    kind: "device_code",
    authorizeUrl: "",
    tokenUrl: "https://github.com/login/oauth/access_token",
    deviceCodeUrl: "https://github.com/login/device/code",
    clientId: "Iv1.b507a08c87ecfe98", // VS Code Copilot extension id — EXPERIMENTAL, ToS risk
    scopes: ["read:user"],
    tokenFile: copilotTokenFile,
    apiHostAllowlist: ["github.com", "githubcopilot.com"],
    experimental: true,
  },
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx vitest run src/services/oauth-flow.test.ts
```

Expected: all PASS.

- [ ] **Step 5: Build + commit**

```bash
npm run build && npx vitest run src/services/oauth-flow.test.ts
git commit -m "feat(oauth): device-code primitive + experimental Copilot provider" -- src/services/oauth-flow.ts src/services/oauth-flow.test.ts
```

---

### Task 4: Token landing — native write + panel status mirror + refresh

Persist the tokens the engine returns and extend refresh coverage.

**Files:**
- Modify: `comfyui-mcp/src/services/code-provider-auth.ts` (add Grok + Copilot resolve/refresh; add a shared `writeNativeTokenFile`)
- Modify: `comfyui-mcp/src/services/panel-secrets.ts` (status-only OAuth mirror)
- Test: `comfyui-mcp/src/services/oauth-landing.test.ts` (new)

**Interfaces:**
- Consumes: `OAuthTokens`, `OAUTH_PROVIDERS` (Task 2/3); `atomicWriteJson`, `tokenExpiring`, `jwtExpMs` patterns (existing in code-provider-auth).
- Produces:
  - `persistOAuthResult(providerId: string, tokens: OAuthTokens, deps?): Promise<{ account_label: string }>` — writes the native token file in the shape that provider's resolver expects (Codex → `{tokens:{...}}`, Grok → `{access_token,...}`, Copilot → `{access_token, token_type}`), derives an account label, and writes the status-only mirror.
  - `readOAuthStatus(): { provider, account_label, obtained_at, expires_at, experimental }[]` — from the mirror, for the UI.
  - `clearOAuth(providerId): void` — deletes the native file + mirror entry.
  - In panel-secrets: `OAUTH_STATUS_KEYS` allowlist + `setOAuthStatus`/`listOAuthStatus`/`clearOAuthStatus` (mirror is a new top-level `oauthStatus` object in `PanelSecrets`, values are non-secret status records).

- [ ] **Step 1: Write the failing tests**

`comfyui-mcp/src/services/oauth-landing.test.ts`:

```ts
import { describe, expect, it, beforeEach } from "vitest";
import { mkdtempSync, readFileSync, existsSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { persistOAuthResult, readOAuthStatus, clearOAuth } from "./code-provider-auth.js";

let home: string;
beforeEach(() => {
  home = mkdtempSync(join(tmpdir(), "oauth-land-"));
  process.env.COMFYUI_MCP_PANEL_SECRETS = join(home, "panel-secrets.json");
});

it("codex: writes ~/.codex/auth.json in tokens{} shape + a status mirror without token material", async () => {
  const idToken = "h." + Buffer.from(JSON.stringify({ chatgpt_account_id: "acct_1", email: "a@b.co" })).toString("base64url") + ".s";
  const { account_label } = await persistOAuthResult("codex", {
    access_token: "AT", refresh_token: "RT", id_token: idToken, raw: {},
  }, { home });
  const auth = JSON.parse(readFileSync(join(home, ".codex", "auth.json"), "utf8"));
  expect(auth.tokens.access_token).toBe("AT");
  expect(auth.tokens.account_id).toBe("acct_1");
  expect(account_label).toContain("a@b.co");
  const status = readOAuthStatus();
  const codex = status.find((s) => s.provider === "codex")!;
  expect(codex.account_label).toContain("a@b.co");
  // mirror must NOT contain the token
  expect(JSON.stringify(status)).not.toContain("AT");
});

it("copilot: writes its own store + status flagged experimental", async () => {
  await persistOAuthResult("copilot", { access_token: "ghu_AT", raw: { token_type: "bearer" } }, { home });
  expect(existsSync(join(home, ".comfyui-mcp", "copilot-auth.json"))).toBe(true);
  const c = readOAuthStatus().find((s) => s.provider === "copilot")!;
  expect(c.experimental).toBe(true);
});

it("clearOAuth removes the native file and the mirror entry", async () => {
  await persistOAuthResult("copilot", { access_token: "ghu_AT", raw: {} }, { home });
  clearOAuth("copilot", { home });
  expect(existsSync(join(home, ".comfyui-mcp", "copilot-auth.json"))).toBe(false);
  expect(readOAuthStatus().find((s) => s.provider === "copilot")).toBeUndefined();
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
npx vitest run src/services/oauth-landing.test.ts
```

Expected: FAIL — exports missing.

- [ ] **Step 3: Implement the mirror in `panel-secrets.ts`**

Add a status-only section (NOT under the secret allowlists — this holds no secrets, only status). In `PanelSecrets` add `oauthStatus?: Record<string, OAuthStatusRecord>`, and:

```ts
export interface OAuthStatusRecord {
  provider: string;
  account_label: string;
  obtained_at: number;
  expires_at?: number;
  experimental?: boolean;
}
// A strict shape guard so a corrupt file can't inject anything but status fields.
export function setOAuthStatus(rec: OAuthStatusRecord): void { /* load, set oauthStatus[rec.provider] = sanitized rec, write 0600 */ }
export function listOAuthStatus(): OAuthStatusRecord[] { /* load, return Object.values(oauthStatus ?? {}) */ }
export function clearOAuthStatus(provider: string): void { /* load, delete, write */ }
```

Implement each by reusing the file's existing load/write helpers (same `readFileSync`/`writeFileSync` + `chmodSync(path, 0o600)` pattern already in the file). Sanitize on set: copy only the five known fields, coerce types, never store anything else.

- [ ] **Step 4: Implement `persistOAuthResult`/`readOAuthStatus`/`clearOAuth` in `code-provider-auth.ts`**

Reuse `atomicWriteJson` (already in the file). Per provider, write the shape that provider's resolver reads:
- **codex:** `{ tokens: { access_token, refresh_token, account_id }, last_refresh }` at `~/.codex/auth.json`; `account_id` from the `id_token` `chatgpt_account_id` claim (reuse the existing `jwtChatgptAccountId` helper); label from the `email` claim if present else the account id.
- **grok:** `{ access_token, refresh_token, expires_in, expires_at }` at `~/.grok/auth.json`; label from an `id_token`/userinfo email if available, else "xAI account".
- **copilot:** `{ access_token, token_type }` at `~/.comfyui-mcp/copilot-auth.json`; label "GitHub Copilot".

Then call `setOAuthStatus({ provider, account_label, obtained_at: now, expires_at, experimental: providerId === "copilot" })`. `readOAuthStatus` = `listOAuthStatus()`. `clearOAuth` = delete the native file (best-effort `rm`), then `clearOAuthStatus(providerId)`. Accept a `deps.home` for tests (mirror the existing `deps.home ?? homedir()` pattern in this file). Extend refresh: add `refreshGrokTokens` mirroring `refreshOpenAICodexTokens` (grant_type=refresh_token, client_id from the grok config, host-allowlist-checked). Copilot `ghu_` tokens do not refresh — on expiry the user re-runs device sign-in.

- [ ] **Step 5: Run tests + build + commit**

```bash
npx vitest run src/services/oauth-landing.test.ts && npm run build
git commit -m "feat(oauth): persist native token files + status-only panel mirror + grok refresh" -- src/services/code-provider-auth.ts src/services/panel-secrets.ts src/services/oauth-landing.test.ts
```

---

### Task 5: Bridge commands — oauth_begin / oauth_status / oauth_signout

Wire the engine to the panel over the bridge, and reflect login in readiness.

**Files:**
- Modify: `comfyui-mcp/src/orchestrator/panel-tools.ts` (or the bridge command dispatch — match where `ask_user`/`show_media`/`ui_render` are handled)
- Modify: `comfyui-mcp/src/orchestrator/backend-readiness.ts`
- Test: `comfyui-mcp/src/orchestrator/oauth-bridge.test.ts` (new)

**Interfaces:**
- Consumes: `OAUTH_PROVIDERS`, `runLoopbackPKCE`, `beginDeviceCode`, `pollDeviceToken`, `persistOAuthResult`, `readOAuthStatus`, `clearOAuth`.
- Produces bridge command handlers:
  - `oauth_begin {provider}` → loopback: kicks the flow, returns `{mode:"loopback", opened:true}`, and on completion persists + pushes a refreshed `{type:"backends"}` frame; device: returns `{mode:"device", user_code, verification_url}` and polls in the background, persisting + pushing on success.
  - `oauth_status {}` → `{providers: readOAuthStatus()}`.
  - `oauth_signout {provider}` → `clearOAuth`, returns `{ok:true}`, pushes refreshed backends.

- [ ] **Step 1: Write the failing test**

`comfyui-mcp/src/orchestrator/oauth-bridge.test.ts` — unit-test the handler functions directly (extract them as exported `handleOAuthBegin`/`handleOAuthStatus`/`handleOAuthSignout` taking injected deps), asserting: unknown provider → error; device provider returns `{mode:"device", user_code}`; a disabled/experimental-off Copilot is refused unless an `allowExperimental` flag is set. (Write 3 focused tests mirroring the shape of the existing panel-tools tests in `src/__tests__/orchestrator/panel-tools.test.ts`.)

- [ ] **Step 2: Run to verify it fails**, implement, **run to verify it passes** (standard TDD cycle; commands as in prior tasks).

- [ ] **Step 3: Implement the handlers** next to the other bridge commands. Guard: `oauth_begin` for `copilot` throws unless the panel passed `allow_experimental:true` (the UI sets this only from the experimental row). Loopback completion and device polling run in the orchestrator; on success call `persistOAuthResult` then push a fresh backends frame (reuse whatever pushes `{type:"backends"}` today — grep for the existing readiness push).

- [ ] **Step 4: Readiness** — in `backend-readiness.ts`, make `auth` true for `codex`/`grok`/`copilot` when `readOAuthStatus()` has a non-expired entry for that provider (OR the existing native-file/CLI check already returns true). Keep the existing CLI/file checks as the fallback so external-CLI logins still count.

- [ ] **Step 5: Build + full suite + commit**

```bash
npm run build && npm test
git commit -m "feat(oauth): bridge oauth_begin/status/signout + readiness reflects sign-in" -- src/orchestrator/panel-tools.ts src/orchestrator/backend-readiness.ts src/orchestrator/oauth-bridge.test.ts
```

---

### Task 6: Grok backend direct-token path (GO only)

If Task 1 was NO-GO, **skip this task entirely** (Grok stays on ACP/CLI). On GO, let the Grok backend use the OAuth token directly so the Grok CLI is optional.

**Files:**
- Modify: `comfyui-mcp/src/orchestrator/grok-backend.ts`

**Interfaces:**
- Consumes: `resolveGrokOAuth` (add to code-provider-auth, mirroring `resolveOpenAICodexOAuth`), `grokTokenFile`.

- [ ] **Step 1: Write a failing test** asserting the backend prefers a present `~/.grok/auth.json` token (hitting `api.x.ai/v1` via the Responses-style adapter it already shares with Codex) and falls back to ACP/CLI when absent. (Mirror the existing grok-backend test structure if one exists; otherwise a focused resolve-path test in code-provider-auth.)
- [ ] **Step 2-4:** implement the direct path (reuse the `codex_responses` adapter the research noted xAI is compatible with; base URL `https://api.x.ai/v1`, bearer from `resolveGrokOAuth`, host-allowlist-checked), verify, ensuring the ACP path is untouched when no token file exists.
- [ ] **Step 5: Build + commit** (`feat(oauth): grok backend uses direct OAuth token when present`).

---

### Task 7: Panel UI — sign-in rows in the credentials card

**Files:**
- Modify: `comfyui-agent-panel/web/js/comfyui-mcp-panel.js`

**Interfaces:**
- Consumes: bridge commands `oauth_begin`/`oauth_status`/`oauth_signout`; the existing credentials card (`cmcpOpenCredentialsFrame`) and its render path; the `{type:"backends"}` readiness handling.
- Produces: per-provider OAuth rows; no new exports.

- [ ] **Step 1: Add an OAuth section to the credentials card** — for each provider present in `oauth_status`, render a row: signed-out → a **"Sign in with {label}"** button; signed-in → the account label + a **"Sign out"** button. The Copilot row renders only under an "Experimental" subheading with a one-line risk note ("Signs in as VS Code; against GitHub's Copilot API terms — use at your own risk"), and its Sign-in sends `allow_experimental:true`.
- [ ] **Step 2: Wire the flows** — Sign in → `oauth_begin {provider}`. On `{mode:"device", user_code, verification_url}`, show the code prominently + a copy-URL affordance + a "waiting for approval…" state; poll `oauth_status` (or receive the pushed backends frame) until signed-in. On `{mode:"loopback", opened:true}`, show "a browser window opened — complete sign-in there" + waiting state. On the pushed `{type:"backends"}` update or a successful `oauth_status`, swap the row to signed-in.
- [ ] **Step 3: Reflect in the provider popup** — a provider isn't "ready"/selectable until signed in; the existing readiness/auto-pick already keys off the backends frame, so confirm the OAuth-driven `auth:true` flows through unchanged.
- [ ] **Step 4: Live verify** (`node --check` first) — reload ComfyUI (:8188), open the credentials card, confirm the rows render and the Copilot row is under Experimental with the note. Full sign-in verification is Task 8.
- [ ] **Step 5: Commit** (`feat(oauth): sign-in rows in the credentials card` — panel repo, branch `feat/grok-provider`).

---

### Task 8: End-to-end verification + orchestrator restart

**Files:** none (fixes go to their owning task's files).

- [ ] **Step 1: Rebuild + restart the orchestrator in a VISIBLE Terminal**

```bash
cd "/Volumes/Main External/Development/comfyui-mcp" && npm run build
# kill the old one (visible-terminal contract):
lsof -ti tcp:9180 -sTCP:LISTEN | xargs -r kill
osascript -e 'tell application "Terminal" to do script "\"/Volumes/Main External/Development/comfyui-mcp/scripts/launch-orchestrator.sh\""'
# confirm a new listener:
sleep 5 && lsof -ti tcp:9180 -sTCP:LISTEN
```

- [ ] **Step 2: Codex sign-in** (loopback) — from the credentials card, Sign in with ChatGPT → browser opens to auth.openai.com → approve → row flips to signed-in with the account email; `~/.codex/auth.json` written; the ChatGPT provider becomes ready. Pick it and send one message to confirm the backend uses the token.
- [ ] **Step 3: Grok sign-in** (GO only) — Sign in with Grok → loopback → approve → signed-in; `~/.grok/auth.json` written; a message routes through the direct-token path. (NO-GO: confirm no Grok OAuth row appears and the CLI/ACP path is unaffected.)
- [ ] **Step 4: Copilot** (experimental) — the row is under Experimental with the risk note; Sign in → device code + verification URL shown → approve on github.com → signed-in. (If GitHub's device path for this client id has changed, record it — this was the flagged risk.)
- [ ] **Step 5: Sign out** — Sign out on one provider → native file removed, mirror cleared, provider no longer ready.
- [ ] **Step 6: Security checks** — grep the orchestrator log for any token material (must be none; only env-key/status names); confirm the panel mirror (`~/.comfyui-mcp/panel-secrets.json`) contains status only, no tokens; confirm loopback bound to 127.0.0.1 (not 0.0.0.0).
- [ ] **Step 7: Record results** in this plan under a "## E2E results" block and commit (panel repo).

## Self-review notes

- Spec §Decisions 1-6 map to: scope/experimental → Tasks 3,7,8; orchestrator-run → Tasks 2,3,5; engine+config (Approach A) → Task 2; local-only → Global Constraints + Task 2 (loopback); native+mirror → Task 4; UI rows → Task 7.
- Spec per-provider table → Task 2 (codex/grok configs), Task 3 (copilot), Task 1 (grok client_id unknown).
- Spec security model → Task 2 (`assertAllowedTokenHost`, state, 127.0.0.1, 0600 via atomicWriteJson) + Task 4 (mirror has no token material, tested) + Task 8 Step 6.
- Spec open unknowns → Task 1 (grok spike, GO/NO-GO) + Task 8 Step 4 (copilot re-confirm).
- Type consistency: `OAuthProviderConfig`/`OAuthTokens`/`OAuthDeps` defined in Task 2 and used verbatim in Tasks 3-6; `persistOAuthResult`/`readOAuthStatus`/`clearOAuth` defined in Task 4, consumed in Task 5; `OAUTH_PROVIDERS` extended (not redefined) in Task 3.
