# DNS

The zone for `dashworldng.com` is managed at **Truehost**. AWS Route 53 is not
in use for this domain.

## Records

| Type | Host | Points at | Notes |
|---|---|---|---|
| A | `@` (root) | the cPanel host | **Do not change.** The marketing site |
| A | `www` | the cPanel host | **Do not change** |
| A | `declutters` | Lightsail static IPv4 | the marketplace web app |
| A | `api` | Lightsail static IPv4 | the API |

Truehost's editor usually wants the bare label (`declutters`), not the fully
qualified name.

## Two things that are easy to get wrong

**Use a Lightsail *static* IP.** An instance's default public IP changes when
the instance is stopped and started. A launch that survives exactly one reboot
is not a launch.

**Both hostnames must stay under `dashworldng.com`.** The API sends
`Cross-Origin-Resource-Policy: same-site`, so putting the API on an unrelated
domain blocks every listing photograph. `declutters.dashworldng.com` plus
`api.dashworldng.com` is fine. This was hit in development; it is not
theoretical.

## Checking

```bash
nslookup declutters.dashworldng.com 8.8.8.8
nslookup api.dashworldng.com 8.8.8.8
```

Query a public resolver rather than trusting a browser, which will happily
serve you a cached answer for hours.

## TLS

certbot proves domain control over port 80, so it can only succeed **after**
both records resolve:

```bash
sudo certbot --nginx -d declutters.dashworldng.com -d api.dashworldng.com
```

Ports 80 and 443 must be open in the Lightsail firewall (Networking tab on the
instance). Leave 3000 and 4000 closed — they are reached through Nginx only.
