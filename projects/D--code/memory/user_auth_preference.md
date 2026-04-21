---
name: Auth preference for internal projects
description: For private internal apps behind Tailscale, the user runs no app-level auth — trusts the tailnet. If auth is ever needed (public-facing app, shared tailnet), default to email+password+TOTP, never social login.
type: user
originSessionId: bf5382a2-2ca4-4512-9de9-911bc0254bfb
---
**Current stance (decided 2026-04-19):**

For private internal web apps (the ones behind his Tailscale network: postgres, mcontrol, analytics, finance, anything similar), the user runs **no app-level authentication**. The apps trust the tailnet — "if you can reach this URL, you're me." No login screens, no user tables, no sessions, no TOTP in the apps themselves.

Reasoning: single user, single tailnet, disciplined device hygiene. Considered and rejected Authelia as unnecessary layering for this threat model.

**Operational implication:** when extracting apps out of `admin_management` or building new ones for the admin zone, drop (don't port) the existing bcrypt/pyotp/session code. App containers bind to the internal docker network + Tailscale interface only — never `0.0.0.0` host ports.

**If auth is ever needed** (e.g., an app moves to the public internet, or trusted users join the tailnet and need per-app scoping):

- Default to **email + password + TOTP** (authenticator app). Authelia is the reference choice — see `D:\code\auth\research\` for the full analysis.
- Explicitly **not** used: social login / upstream IdP federation (no Google, GitHub, etc.), passkeys (may revisit), SAML.

**Future work flagged (2026-04-19):** the user expects to build public-facing apps eventually that *will* need real auth. The `auth` repo (`D:\code\auth`, https://github.com/AbstractNucleus/auth) is kept specifically as the research archive for that future work — do not delete.

**How to apply:** when helping with any app in `D:\code\admin_management` or its extracted pieces (postgres, mcontrol, analytics, finance) or new sibling apps that sit behind Tailscale, assume no login flow, no user model. When the user begins work on a public-facing app, point to `D:\code\auth\research\04-recommendation.md` as the resumption point and revisit the open questions there (proxy choice, M2M clients, session TTL).
