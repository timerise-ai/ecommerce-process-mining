# Connecting the extension **[A]** (handshake specified in the earlier design **[D]**, never built there)

The employee never copies a secret. The extension gets a scoped API key through an OAuth-style
authorization-code flow with PKCE, using Chrome's `chrome.identity.launchWebAuthFlow`.

## Handshake

```
popup            chrome.identity        consent page            exchange route        DB
  | verifier, state
  | challenge = S256
  +- launchWebAuthFlow(/extension-auth?state&code_challenge&extension_id&redirect_uri)
  |                 +----- GET --------->  (login first if needed)
  |                 |                      validate 4 params, check the allowlist
  |                 |                      user clicks Authorize
  |                 |                      re-validate ------------------------> insert grant (60 s)
  |                 <----- 302 redirect_uri?code&state
  <- callback URL --+
  | check state
  +------------ POST {code, code_verifier, extension_id} ------> claim (DELETE ... RETURNING)
  |                                                              verify S256(verifier)
  |                                                              mint key [process_mining:emit]
  <------------ { secret, key_id } -----------------------------+
  | store the secret, show "Connected"
```

`https://<extension-id>.chromiumapp.org/` and its sub-paths are a redirect endpoint Chrome owns and only hands
to the extension with that ID. No web page can receive it.

## The module (shared by extension and server)

```ts
// file: lib/process-mining/pkce.ts
// OAuth-style connect flow for a browser extension (public client, PKCE S256).
// Uses WebCrypto only, so the SAME file runs in the extension service worker,
// in a route handler on Node >= 20, and in tests.

const B64URL = /^[A-Za-z0-9_-]+$/;

function toBase64Url(bytes: Uint8Array): string {
  let bin = '';
  for (const b of bytes) bin += String.fromCharCode(b);
  return btoa(bin).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

export function randomToken(byteLength: number): string {
  return toBase64Url(crypto.getRandomValues(new Uint8Array(byteLength)));
}

export function randomHex(byteLength: number): string {
  return [...crypto.getRandomValues(new Uint8Array(byteLength))]
    .map((b) => b.toString(16).padStart(2, '0'))
    .join('');
}

export async function challengeFor(verifier: string): Promise<string> {
  const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(verifier));
  return toBase64Url(new Uint8Array(digest));
}

function constantTimeEqual(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  return diff === 0;
}

export async function verifyPkce(verifier: string, storedChallenge: string): Promise<boolean> {
  // RFC 7636 section 4.1: 43 to 128 characters from the unreserved set.
  if (!/^[A-Za-z0-9._~-]{43,128}$/.test(verifier)) return false;
  return constantTimeEqual(await challengeFor(verifier), storedChallenge);
}

export interface AuthorizeParams {
  state: string;
  code_challenge: string;
  extension_id: string;
  redirect_uri: string;
}

export type AuthorizeParamError =
  | 'bad_state'
  | 'bad_code_challenge'
  | 'bad_extension_id'
  | 'extension_not_allowed'
  | 'bad_redirect_uri';

export type AuthorizeValidation =
  | { ok: true; params: AuthorizeParams }
  | { ok: false; error: AuthorizeParamError };

/**
 * Run on the consent page AND again inside the authorize action, because hidden form
 * inputs are attacker-controlled by the time they come back.
 *
 * allowedExtensionIds is the check the regexes cannot make: ANY extension can
 * produce a well-formed id and a matching chromiumapp.org redirect. Without an
 * allowlist, a look-alike extension gets a real key from a real consent click.
 */
export function validateAuthorizeParams(
  raw: Record<string, string | undefined>,
  allowedExtensionIds: readonly string[],
): AuthorizeValidation {
  const { state = '', code_challenge = '', extension_id = '', redirect_uri = '' } = raw;
  if (!/^[0-9a-f]{32,128}$/.test(state)) return { ok: false, error: 'bad_state' };
  // Unpadded base64url of 32 bytes is exactly 43 characters.
  if (code_challenge.length !== 43 || !B64URL.test(code_challenge)) {
    return { ok: false, error: 'bad_code_challenge' };
  }
  if (!/^[a-p]{32}$/.test(extension_id)) return { ok: false, error: 'bad_extension_id' };
  if (!allowedExtensionIds.includes(extension_id)) {
    return { ok: false, error: 'extension_not_allowed' };
  }
  let u: URL;
  try {
    u = new URL(redirect_uri);
  } catch {
    return { ok: false, error: 'bad_redirect_uri' };
  }
  // Compare the PARSED host rather than pattern-matching the raw string. A regex
  // is correct only while it keeps its trailing "/" and every escaped dot; drop
  // either, and "https://<id>.chromiumapp.org.evil.test/" or "...org@evil.test/" pass.
  if (
    u.protocol !== 'https:' ||
    u.hostname !== `${extension_id}.chromiumapp.org` ||
    u.username !== '' ||
    u.password !== '' ||
    u.port !== ''
  ) {
    return { ok: false, error: 'bad_redirect_uri' };
  }
  return { ok: true, params: { state, code_challenge, extension_id, redirect_uri } };
}

export function buildCallbackUrl(redirectUri: string, code: string, state: string): string {
  const u = new URL(redirectUri);
  u.searchParams.set('code', code);
  u.searchParams.set('state', state);
  return u.toString();
}

export interface ClaimedGrant {
  user_id: string;
  scope_id: string;
  code_challenge: string;
}

export interface ExchangeDeps {
  /**
   * Atomically delete-and-return the unexpired grant for (code, extension_id).
   * MUST be one statement: DELETE ... RETURNING, or a transaction. SELECT-then-
   * DELETE lets two concurrent requests both redeem the same code.
   */
  claimGrant(code: string, extensionId: string): Promise<ClaimedGrant | null>;
  mintKey(grant: ClaimedGrant, extensionId: string): Promise<{ secret: string; keyId: string }>;
}

export type ExchangeResult =
  | { ok: true; secret: string; key_id: string }
  // One generic failure. Saying WHICH check failed is a free oracle for a brute-forcer.
  | { ok: false; status: 401; error: 'invalid_grant' };

export async function exchangeCode(
  body: { code?: unknown; code_verifier?: unknown; extension_id?: unknown },
  deps: ExchangeDeps,
): Promise<ExchangeResult> {
  const fail = { ok: false, status: 401, error: 'invalid_grant' } as const;
  const { code, code_verifier, extension_id } = body;
  if (typeof code !== 'string' || typeof code_verifier !== 'string') return fail;
  if (typeof extension_id !== 'string' || !B64URL.test(code) || code.length > 128) return fail;

  // Claim FIRST, verify second: a wrong verifier still burns the code, so a
  // stolen code gets exactly one guess.
  const grant = await deps.claimGrant(code, extension_id);
  if (!grant) return fail;
  if (!(await verifyPkce(code_verifier, grant.code_challenge))) return fail;

  const key = await deps.mintKey(grant, extension_id);
  return { ok: true, secret: key.secret, key_id: key.keyId };
}
```

## Server pieces

**Consent page.** An ordinary signed-in page. Validate with `validateAuthorizeParams(searchParams,
ALLOWED_EXTENSION_IDS)`; on failure render an error and **no form**. Say what the extension will do, and that
nothing is captured until the consent toggle is on. Two buttons: Authorize, Cancel.

**Authorize action.** Re-validate, because hidden inputs are attacker-controlled by the time they return),
then:

```ts
// file: app/extension-auth/actions.ts
// Host seams: currentUser(), currentScope(), insertGrant(), redirect().
const v = validateAuthorizeParams(Object.fromEntries(formData) as Record<string, string>, ALLOWED_EXTENSION_IDS);
if (!v.ok) throw new Error('invalid authorization request');
const code = randomToken(48);
await insertGrant({
  code,
  user_id: currentUser().id,
  scope_id: currentScope(),              // the tenant of THIS session
  code_challenge: v.params.code_challenge,
  extension_id: v.params.extension_id,
});
redirect(buildCallbackUrl(v.params.redirect_uri, code, v.params.state)); // server-side 302
```

The redirect is server-issued so the code never enters client-side router history.

**Exchange route.** `POST`, with no auth header, because the code is the credential. Rate-limit by IP and by
`extension_id` *before* touching the database, then call `exchangeCode` with:

- `claimGrant` calls `select * from pm_claim_extension_grant($1, $2)` through the privileged client.
- `mintKey` calls the host's existing key-creation path. Scope it to exactly `['process_mining:emit']`, owner
  the grant's user, tenant the grant's `scope_id`, and a name such as `Routine Capture (Chrome abcdefgh)`.
  **Do not reimplement secret generation.**

Respond `200 { secret, key_id }` with `cache-control: no-store`, or `401 invalid_grant`. Record a key-created
audit event tagged `via: 'extension'`.

**Revoke route.** `POST`, authenticated by the key itself. Body `{ key_id }`. Proceed only if the bearer key's
id equals `key_id`: a key can delete itself and nothing else. It exists because the popup cannot call a
session-authenticated server action.

## Extension side **[A]**, not compiled, because no Chrome typings were available

```ts
// file: extension/src/auth.ts
import { challengeFor, randomHex, randomToken } from '../../lib/process-mining/pkce';

export async function connect(consoleOrigin: string): Promise<void> {
  const verifier = randomToken(64);
  const state = randomHex(32);
  // storage.session: in memory, survives service-worker eviction, wiped when the
  // browser closes, never synced. Module variables do NOT survive eviction, and
  // eviction can happen while the user is reading the consent page.
  await chrome.storage.session.set({ pkce: { verifier, state } });
  const redirectUri = chrome.identity.getRedirectURL('cb');
  const url = new URL('/console/extension-auth', consoleOrigin);
  url.search = new URLSearchParams({
    state,
    code_challenge: await challengeFor(verifier),
    extension_id: chrome.runtime.id,
    redirect_uri: redirectUri,
  }).toString();
  try {
    const cb = await chrome.identity.launchWebAuthFlow({ url: url.toString(), interactive: true });
    const params = new URL(cb ?? '').searchParams;
    const saved = (await chrome.storage.session.get('pkce')).pkce as { verifier: string; state: string };
    if (!saved || params.get('state') !== saved.state) throw new Error('state mismatch');
    const res = await fetch(new URL('/api/v1/process-mining/extension-auth/exchange', consoleOrigin), {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ code: params.get('code'), code_verifier: saved.verifier, extension_id: chrome.runtime.id }),
    });
    if (!res.ok) throw new Error('invalid_grant');
    const { secret, key_id } = (await res.json()) as { secret: string; key_id: string };
    await chrome.storage.local.set({ auth: { secret, key_id, origin: consoleOrigin } });
  } finally {
    await chrome.storage.session.remove('pkce'); // success or failure
  }
}
```

## Corrections and reinforcements to the earlier design: keep these

| The design said | Ship this | Because |
|---|---|---|
| Extension id matches `[a-z]{32}` | `[a-p]{32}` | Chrome ids are hex mapped onto a to p |
| `code_challenge` is "44-char" (one place), 43 (another) | exactly 43 | unpadded base64url of 32 bytes |
| `redirect_uri` matches `^https://<id>\.chromiumapp\.org/` | same rule, implemented by comparing the **parsed** hostname and forbidding userinfo and port | **Not a defect.** The design's regex is correct as written. It stays correct only while it keeps its trailing `/` and both escaped dots; parsing does not depend on anyone preserving punctuation |
| No check on *which* extension is asking | allowlist of ids | any extension can run this flow with a well-formed id of its own; the employee sees your real consent page and clicks Authorize |
| Exchange: `SELECT`, verify, then `DELETE` | one `DELETE ... RETURNING`, verify after | two concurrent redeems both pass the `SELECT` and both mint a key |
| Verifier "in memory only" (one place), `storage.session` (another) | `storage.session` | the worker can be evicted between Authorize and the callback |
| Store the secret in `chrome.storage.sync` | `chrome.storage.local` | `sync` uploads the secret to the user's Google account and onto every signed-in machine, including the personal laptop nobody vetted. The cost: connect once per browser |
| Response carries `user_email` | omit it; show the email from a key-authenticated `whoami` | one less identifier on an unauthenticated response |

## Why PKCE

An extension is a public client: its package is downloadable, so it can hold no secret. PKCE binds the code to
a verifier only the initiating extension has. A code stolen from a log or a proxy is useless without it, and
with claim-before-verify a wrong guess burns the code.

## Pin the extension ID

Chrome derives an unpacked extension's id from its path, so every developer machine gets a different one and
the allowlist never matches. Put the public half of a fixed key pair in `manifest.json` `key`; keep the
private half in a secrets manager. Regenerating it changes the id and disconnects everyone.

The pair belongs to the operator. Code built from this skill ships the step that makes it (a key-generation
script is fine) and leaves `key` and the allowlist empty; it never commits a generated pair, and never writes
a derived id into code as a default. `ALLOWED_EXTENSION_IDS` is read from configuration at request time, and
an empty list refuses every extension, which is the closed state the first hard rule asks for. **[A]**

## Tests

```ts
// file: lib/process-mining/pkce.test.ts
import { describe, expect, it } from 'vitest';
import { buildCallbackUrl, challengeFor, exchangeCode, randomHex, randomToken, validateAuthorizeParams, verifyPkce, type ClaimedGrant } from './pkce';

const ID = 'abcdefghijklmnopabcdefghijklmnop';
const good = async () => ({
  state: randomHex(32), code_challenge: await challengeFor(randomToken(64)),
  extension_id: ID, redirect_uri: `https://${ID}.chromiumapp.org/cb`,
});

describe('PKCE primitives', () => {
  it('matches the RFC 7636 appendix B vector', async () => {
    expect(await challengeFor('dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk')).toBe('E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM');
  });
  it('challenge is 43 chars; 64 random bytes is a legal verifier', async () => {
    const v = randomToken(64);
    expect(v.length).toBe(86);
    expect((await challengeFor(v)).length).toBe(43);
    expect(await verifyPkce(v, await challengeFor(v))).toBe(true);
    expect(await verifyPkce(v + 'x', await challengeFor(v))).toBe(false);
    expect(await verifyPkce('short', await challengeFor('short'))).toBe(false);
  });
});

describe('validateAuthorizeParams', () => {
  it('accepts a well-formed request from an allowed extension', async () => {
    expect(validateAuthorizeParams(await good(), [ID]).ok).toBe(true);
  });
  it('refuses an extension that is not on the allowlist', async () => {
    expect(validateAuthorizeParams(await good(), [])).toEqual({ ok: false, error: 'extension_not_allowed' });
  });
  it('refuses look-alike, userinfo, http, ported and foreign redirects', async () => {
    const p = await good();
    for (const redirect_uri of [
      `https://${ID}.chromiumapp.org.evil.test/cb`, `https://${ID}.chromiumapp.org@evil.test/cb`,
      `http://${ID}.chromiumapp.org/cb`, `https://${ID}.chromiumapp.org:8443/cb`, 'https://evil.test/cb', 'not a url',
    ]) expect(validateAuthorizeParams({ ...p, redirect_uri }, [ID])).toEqual({ ok: false, error: 'bad_redirect_uri' });
  });
  it('refuses malformed state, challenge and id (ids use a-p only)', async () => {
    const p = await good();
    expect(validateAuthorizeParams({ ...p, state: 'xyz' }, [ID])).toMatchObject({ error: 'bad_state' });
    expect(validateAuthorizeParams({ ...p, code_challenge: p.code_challenge + '=' }, [ID])).toMatchObject({ error: 'bad_code_challenge' });
    const z = 'z'.repeat(32);
    expect(validateAuthorizeParams({ ...p, extension_id: z }, [z])).toMatchObject({ error: 'bad_extension_id' });
  });
  it('builds the callback without clobbering the path', () => {
    expect(buildCallbackUrl(`https://${ID}.chromiumapp.org/cb`, 'c0de', 'st')).toBe(`https://${ID}.chromiumapp.org/cb?code=c0de&state=st`);
  });
});

describe('exchangeCode', () => {
  async function setup() {
    const verifier = randomToken(64);
    const grant: ClaimedGrant = { user_id: 'u1', scope_id: 'org1', code_challenge: await challengeFor(verifier) };
    const store = new Map([['c0de', grant]]);
    let minted = 0;
    const deps = {
      claimGrant: async (code: string) => { const g = store.get(code) ?? null; store.delete(code); return g; },
      mintKey: async () => (minted++, { secret: 'sk_live', keyId: 'k1' }),
    };
    return { verifier, deps, minted: () => minted };
  }
  it('redeems once; the second attempt gets nothing', async () => {
    const { verifier, deps, minted } = await setup();
    const body = { code: 'c0de', code_verifier: verifier, extension_id: ID };
    expect(await exchangeCode(body, deps)).toEqual({ ok: true, secret: 'sk_live', key_id: 'k1' });
    expect(await exchangeCode(body, deps)).toEqual({ ok: false, status: 401, error: 'invalid_grant' });
    expect(minted()).toBe(1);
  });
  it('concurrent redemptions mint exactly one key', async () => {
    const { verifier, deps, minted } = await setup();
    const body = { code: 'c0de', code_verifier: verifier, extension_id: ID };
    const results = await Promise.all([exchangeCode(body, deps), exchangeCode(body, deps), exchangeCode(body, deps)]);
    expect(results.filter((r) => r.ok)).toHaveLength(1);
    expect(minted()).toBe(1);
  });
  it('a wrong verifier burns the code', async () => {
    const { verifier, deps, minted } = await setup();
    expect((await exchangeCode({ code: 'c0de', code_verifier: randomToken(64), extension_id: ID }, deps)).ok).toBe(false);
    expect((await exchangeCode({ code: 'c0de', code_verifier: verifier, extension_id: ID }, deps)).ok).toBe(false);
    expect(minted()).toBe(0);
  });
  it('every failure looks the same', async () => {
    const { deps } = await setup();
    const fails = await Promise.all([
      exchangeCode({}, deps), exchangeCode({ code: 'nope', code_verifier: 'x', extension_id: ID }, deps),
      exchangeCode({ code: 'has space', code_verifier: 'x', extension_id: ID }, deps),
    ]);
    for (const f of fails) expect(f).toEqual({ ok: false, status: 401, error: 'invalid_grant' });
  });
});
```

## Checklist

- [ ] `ALLOWED_EXTENSION_IDS` from config; production and dev ids both listed
- [ ] Params validated on render **and** in the action
- [ ] Grant claimed in one statement; table invisible to API roles
- [ ] Every exchange failure returns the same `401 invalid_grant`
- [ ] Rate limit ahead of the database
- [ ] Minted key: `emit` scope only, bound to the grant's user and tenant
- [ ] Secret in `storage.local`; verifier in `storage.session`, removed in `finally`
- [ ] Manifest `key` pinned
