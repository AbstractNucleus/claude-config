---
name: noelkleen.com domain layout
description: The user's personal root domain is noelkleen.com. Not all subdomains are behind auth — protected internal apps live under the admin.noelkleen.com subzone.
type: project
originSessionId: bf5382a2-2ca4-4512-9de9-911bc0254bfb
---
The user's personal root domain is **`noelkleen.com`**. It hosts a mix of protected and unprotected content.

**Protected internal apps** (all behind Authelia via forward-auth) live under the **`admin.noelkleen.com` subzone**:

- `auth.admin.noelkleen.com` — Authelia portal
- `postgres.admin.noelkleen.com` — Postgres management panel (highest sensitivity; controls the Postgres instance used by all projects)
- `mcontrol.admin.noelkleen.com` — Minecraft server management
- `analytics.admin.noelkleen.com`
- `finance.admin.noelkleen.com`

**Why:** scoping the Authelia session cookie to `.admin.noelkleen.com` keeps the auth trust boundary off the broader `noelkleen.com` zone, which has uncontrolled or public subdomains that would otherwise inherit the cookie.

**How to apply:** when working on any of the admin apps (postgres, mcontrol, analytics, finance, auth), assume hostnames follow `<app>.admin.noelkleen.com`. When the user mentions exposing a new app "behind auth," the default is to put it under this subzone too. Public/marketing content belongs on bare `noelkleen.com` or other subdomains, never inside the admin zone.
