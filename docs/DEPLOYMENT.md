# Deploying a release

Production code lives at `/opt/dash-declutters` on the Lightsail instance.
Canonical source is `dashworldng-bit/DASH-APP`.

**The API is deployed before the web app. Always.** The reason is specific and
current: the pending release adds `GET /v1/auth/csrf`, which the new web bundle
calls to recover from a CSRF rejection. Deploy the web first and that call
answers 404, so a returning visitor stays broken — no worse than before, but no
better, and you would have spent a deploy to learn nothing.

The general rule behind it: ship the side that *answers* before the side that
*asks*.

---

## 0. Before you start

- [ ] The change is reviewed and approved for release.
- [ ] You know which commit you are deploying, and it is pushed.
- [ ] Someone is available to look at a browser afterwards — see
      `VERIFICATION-BROWSER.md`. A deploy nobody checks is a deploy you find
      out about from a customer.

Take a database snapshot **only if the release contains a migration**. This one
does not. A snapshot of an unchanged database is not free and is not evidence of
anything.

---

## 1. Fetch the code

```bash
cd /opt/dash-declutters
sudo -u dash git fetch --all
sudo -u dash git checkout <branch-or-tag>
sudo -u dash git log -1 --oneline
```

Read that last line back and confirm it is the commit you meant. This is the
cheapest checkpoint in the whole procedure.

```bash
sudo -u dash npm ci
```

---

## 2. API first

The API runs from source through `tsx`, so there is no build step.

```bash
sudo systemctl restart dash-declutters-api
systemctl status dash-declutters-api --no-pager
journalctl -u dash-declutters-api -n 50 --no-pager
```

The API fails **loudly** at boot on a missing or weak variable — it names the
variable. If it will not start, that log is the answer; do not go looking
elsewhere.

Confirm it is actually serving before touching the web app:

```bash
curl -s https://api.dashworldng.com/health/ready
curl -s -o /dev/null -w '%{http_code}\n' https://api.dashworldng.com/v1/auth/csrf
```

`/health/ready` must report `database`, `queue` and `storage` all ok.
`/v1/auth/csrf` must answer **200** — that is the endpoint the new web bundle
depends on, and this is the gate that makes step 3 safe.

> A green readiness check does **not** mean email works. Nothing probes SES.

---

## 3. Web second

`NEXT_PUBLIC_*` is read at **build time**, not at start. The variables must be
exported into the environment of `npm run build`, not only set in the
supervisor. A rebuild is required to change any of them.

```bash
cd /opt/dash-declutters
sudo -u dash \
  NEXT_PUBLIC_API_URL=https://api.dashworldng.com \
  NEXT_PUBLIC_SITE_ORIGIN=https://declutters.dashworldng.com \
  NEXT_PUBLIC_DECLUTTER_INDEXABLE=false \
  DEPLOYMENT_PROFILE=declutters \
  NODE_ENV=production \
  npm run build
```

Then prove the build did not bake in a development value. The repository ships
an `apps/web/.env.local` pointing at `localhost`, and it is overridden only
because the shell environment wins:

```bash
grep -rl "localhost:4100" apps/web/.next/ | head
```

**Must print nothing.** If it prints anything, the build inherited the wrong
variables — rebuild, do not restart.

```bash
sudo systemctl restart dash-declutters-web
systemctl status dash-declutters-web --no-pager
```

The web process listens on **3000** and `PORT` does not change it —
`apps/web/package.json` pins `next start --port 3000`. Nginx proxies to 3000.

---

## 4. Confirm the deployment boundary still holds

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://declutters.dashworldng.com/declutter
curl -s -o /dev/null -w '%{http_code}\n' https://declutters.dashworldng.com/declutter/terms
curl -s -o /dev/null -w '%{http_code}\n' https://declutters.dashworldng.com/ops/declutter
curl -s -o /dev/null -w '%{http_code}\n' https://declutters.dashworldng.com/vendor
curl -s -o /dev/null -w '%{http_code}\n' https://declutters.dashworldng.com/ops
curl -s https://declutters.dashworldng.com/robots.txt
```

Expected: `200`, `200`, `200`, **`404`**, **`404`**, and `Disallow: /`.

A `200` on `/vendor` or `/ops` means the web process does not have
`DEPLOYMENT_PROFILE=declutters` in its environment — check the systemd unit, not
the build.

`/declutter/terms` is worth watching: it is served from a `LegalDocument` row
through the API, so it 404s whenever the API cannot answer. Dead Terms and
Privacy links on a public marketplace are worth stopping a release over.

---

## 5. Then a human opens a browser

`curl` cannot see what this release changed. The CSRF fix is about what a
*browser* does with a host-only cookie across two subdomains, which no
server-side check reproduces. Go to `VERIFICATION-BROWSER.md` and work through
it before telling anyone the release is out.

---

## Rollback

Nothing in a code-only release writes data, so rollback is a checkout and two
restarts — API first again:

```bash
cd /opt/dash-declutters
sudo -u dash git checkout <previous-commit>
sudo -u dash npm ci
sudo systemctl restart dash-declutters-api
# then rebuild the web app exactly as in step 3, and restart it
```

The web app **must** be rebuilt on rollback. Restarting it alone leaves the new
bundle in `.next/` and serves the version you are trying to withdraw.

If a release ever does contain a migration, rolling the code back does **not**
roll the schema back. Migrations are forward-only; a schema problem is fixed
with a new forward migration, never by editing or reverting an applied one.
