# Real-browser verification

For the release containing the CSRF cross-subdomain fix and the request-body
fixes.

**Why a human has to do this.** Every automated test for the CSRF change runs
in-process against an injected Fastify instance. That cannot reproduce the thing
that actually broke: what a *browser* does with a **host-only cookie across two
subdomains**. The tests prove the mechanism is right. Only this proves the fix
works.

The failure being verified, from the production journal, 2026-09-23 04:24:55
UTC:

```
POST /v1/auth/otp/request -> 403
csrf_missing; missing x-csrf-token header
```

Note the shape: the cookie **was** present and reached the API. The header was
not, because the web origin could not read a cookie belonging to the API origin.

---

## Setup

Use Chrome or Edge with DevTools open on **Network**, and tick
**Preserve log** — the recovery involves a request that is *supposed* to fail,
and without preserve-log a redirect or reload will hide it.

Have two browser states ready, because the bug only appeared in one of them:

| State | Why it matters |
|---|---|
| **A — returning visitor** | Already holds a refresh cookie, so the API enforces CSRF. **This is the broken case.** |
| **B — fresh incognito** | No refresh cookie, so CSRF is never enforced. This is why the bug looked intermittent, and it must not regress. |

To get state A without waiting: sign in once (or simply visit the site after a
previous session), then confirm in **DevTools → Application → Cookies →
`https://api.dashworldng.com`** that `dash_rt` is present.

---

## 1. The cookie is still host-only

DevTools → Application → Cookies.

- [ ] Under `https://api.dashworldng.com`: `dash_csrf` is present, **Domain is
      `api.dashworldng.com`** (no leading dot), `HttpOnly` **unticked**.
- [ ] Under `https://api.dashworldng.com`: `dash_rt` is present, `HttpOnly`
      **ticked**, `Secure` ticked, `Path` is `/v1/auth`.
- [ ] Under `https://declutters.dashworldng.com`: **no `dash_csrf`**.

That last one is the point of the whole design. A `dash_csrf` appearing on the
web origin means someone added a `Domain` attribute, which would fix the symptom
by broadcasting the token to every subdomain — including hosting we do not
control. If you see it, stop and escalate.

---

## 2. State A — the returning visitor recovers

On `https://declutters.dashworldng.com/declutter/signin`, enter a real address
and request a code. Watch Network.

- [ ] `OPTIONS /v1/auth/otp/request` → **204**
- [ ] `POST /v1/auth/otp/request` → **403**, body code `csrf_missing`
- [ ] `GET /v1/auth/csrf` → **200**, and `Cache-Control: no-store` in the
      response headers
- [ ] `POST /v1/auth/otp/request` again → **not 403**
- [ ] The page shows the normal "we sent you a code" state, not an error

**One 403 is expected and correct.** The client does not ask for a token
speculatively — it asks only once the server has said one is needed. A rejected
request never reached a handler, so nothing happened twice.

On the retry, click the second `POST` → Headers → Request Headers:

- [ ] `X-CSRF-Token` is present
- [ ] Its value matches the `csrfToken` in the `GET /v1/auth/csrf` response body
- [ ] It matches the `dash_csrf` cookie value in Application → Cookies

---

## 3. The token is reused, not refetched

Still in state A, perform a second write — request another code after the
cooldown, or use any other action.

- [ ] **No second `GET /v1/auth/csrf`**
- [ ] **No second 403** — the write succeeds first time

If you see the bootstrap repeat on every write, the in-memory cache is not
holding and it should be raised. It is not a correctness failure, but it is a
round trip per write.

---

## 4. State B — a fresh visitor pays nothing

New incognito window, `https://declutters.dashworldng.com/declutter/signin`.

- [ ] `POST /v1/auth/otp/request` → **not 403**, first time
- [ ] **No `GET /v1/auth/csrf` at all**

This is the path that already worked, and the fix must not have made it slower.

---

## 5. Reload and stale-session recovery

- [ ] With a session established, press **F5**. The page restores its signed-in
      state; no CSRF error appears.
- [ ] In Application → Cookies, delete **only** `dash_csrf` (leave `dash_rt`),
      then perform a write. Expect one 403, one `GET /v1/auth/csrf`, then
      success — the same recovery, which is what a genuinely expired token will
      do.
- [ ] Delete **both** cookies and reload. The app should present a signed-out
      state cleanly, with no console errors and no spinner that never resolves.

---

## 6. CSRF is still doing its job

This is the half that matters most. A fix that makes sign-in work by weakening
the protection is worse than the bug.

In the DevTools console **on `https://declutters.dashworldng.com`**:

```js
await fetch('https://api.dashworldng.com/v1/auth/otp/request', {
  method: 'POST',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: 'someone@example.test', purpose: 'login' }),
}).then(r => r.status)
```

- [ ] Returns **403**. The browser attaches the cookie; without the header the
      API must still refuse.

Repeat with `'X-CSRF-Token': 'not-the-real-token'` added to the headers:

- [ ] Also **403**, code `csrf_invalid`.

If either returns 200, **stop and roll back.** CSRF protection is not working.

---

## 7. The request-body fixes

These shipped in the same release and are separately verifiable. Each was
sending a JSON string where the API expected an object.

- [ ] **Become a seller** (`/declutter/sell`) — submit the form. Expect success,
      not a validation error about a field you filled in correctly.
- [ ] **Contact request** — open a published listing and send one. In Network,
      click the request → Payload: it must show a **JSON object**, not a quoted
      string beginning `"{\"`.
- [ ] **Admin** (`/ops/declutter`, signed in as staff) — approve or reject a
      seller, and move a submission through a transition. Both must succeed.

The Payload check is the quick tell for all of them: a double-encoded body shows
as one long quoted string with escaped inner quotes.

---

## 8. Mobile, on mobile data

- [ ] Open a listing on a phone, on **mobile data rather than office wifi**, and
      confirm the photographs load.

This is the check that catches a storage or `same-site` misconfiguration, and it
cannot be done from a desktop on the same network as everything else.

---

## If anything in section 6 fails

Roll back per `DEPLOYMENT.md` — API first — and report which assertion failed
and the exact request and response headers. Do not attempt a fix-forward on a
CSRF regression.

## What this does not cover

- **Email delivery.** Nothing here proves SES delivered anything. A sign-in code
  that never arrives is a separate, open investigation; correlate SES event
  types and message IDs in CloudWatch rather than inferring from an HTTP 200.
- **Public signup at large.** While SES is in the sandbox, signup is restricted
  to the allowlist. Verify with an allowlisted address only.
