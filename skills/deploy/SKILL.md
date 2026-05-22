---
name: deploy
description: Deploy or redeploy the current project to one of the user's configured hosts. Reads `HOSTS.md` (gitignored, user-supplied) for target servers, infers what the project needs from its files (Dockerfile, compose, package.json, etc.), then SSHes to set up nginx routing, pull the repo, and (re)start containers. Auto-detects first-deploy vs update by inspecting the target host. Use when the user invokes /deploy, says "deploy this", "redeploy", "ship it to prod", or similar.
---

# /deploy

Deploy the current project to one of the user's hosts. Single flow; auto-detects whether this is a first deploy or an update.

## Preflight

1. **Read `HOSTS.md`** in this skill folder (`~/.claude/skills/deploy/HOSTS.md`). If it's missing, tell the user to copy `HOSTS.example.md` to `HOSTS.md` and fill it in, then stop.
2. **Identify the project.** Read in parallel:
   - Git remote (`git remote -v`) → repo name and URL.
   - `Dockerfile`, `docker-compose.yml`, `compose.yaml`, containerised? what ports does it expose?
   - `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod`, build system + name.
   - `README.md`, any stated deploy notes.
3. **Pick the target host(s).** From `HOSTS.md`, match the project's needs to host roles (nginx gateway, docker host, bare-metal, etc.). If `HOSTS.md` is ambiguous about where this project should land, ask the user once.

## Plan first, execute second

Write the plan and show it to the user before SSHing. Pull every host-specific path, command, and routing convention from `HOSTS.md` — don't invent. If `HOSTS.md` describes more than one binding pattern (e.g. public-facing vs tailnet-only / internal-only), explicitly state which you're using and why; cert chain, snippet includes, and listening address often differ between them.

```
Project:    <name>
Containers: <list, or "none">
Ports:      <internal port → public path/host>
Public hostname: <e.g. app.example.com — or "tailnet-only">
Binding:    public | tailnet-only | internal-only  (per HOSTS.md)
Nginx host: <from HOSTS.md>  →  proxy_pass to <docker host>:<port>
Docker host: <from HOSTS.md>

Steps:
1. ssh <docker host>, clone or pull repo
2. ssh <docker host>, build + (re)start containers (exact command from HOSTS.md)
3. ssh <nginx host>, drop/update site config + reload (only if routing changed)
```

Wait for the user to confirm before running anything destructive (anything that touches running services or shared nginx config). Deploys can't be undone from this side.

## Detect first-deploy vs update

After confirmation, SSH the relevant host(s) and check:

- Does the repo directory exist on the docker host?
- Is there an existing nginx site config for this project on the nginx host?
- Are containers for this project currently running?

Branch on what you find:

- **First deploy:** repo absent OR no nginx config.
- **Update:** repo present AND nginx config present.

Do not blow away nginx config that is already correct. Do not stop unrelated containers.

## First-deploy flow

1. SSH docker host. Clone the repo into the path specified by `HOSTS.md`.
2. Build and start using the **exact build/run command from `HOSTS.md`** for this host (typically `docker compose -p <project> up -d --build` or similar — convention varies, don't guess).
3. Wait for healthcheck or `docker compose ps` to show containers up.
4. SSH nginx host. Drop `sites-available/<project>.conf` following the routing convention in `HOSTS.md` (cert chain, snippet includes, and listening address depend on whether this is a public or tailnet/internal-only binding). Symlink to `sites-enabled/`.
5. `sudo nginx -t` → `sudo systemctl reload nginx`.
6. Curl the public URL from the user's machine, verify HTTP 200 (or the expected status).

## Update flow

1. SSH docker host.
2. `cd <project path>`. Capture the current SHA so the user can revert later if needed: `git rev-parse HEAD`. Report it.
3. `git pull`, show the diff to the user.
4. Use the **exact build/run command from `HOSTS.md`** to restart the stack.
5. Wait for containers to be healthy.
6. Touch nginx **only** if ports or hostnames changed in this update.
7. Curl the public URL, verify.

## Safety rules

- Never run `git reset --hard`, `git clean -fd`, or force-push on a server's repo without explicit confirmation.
- Show the user the new nginx config (or the diff against the existing one) before applying.
- When stopping containers, scope with `-p <project>` if the docker host runs multiple stacks. Never `docker stop $(docker ps -q)`.
- If a step fails, stop. Do not auto-retry destructive operations. Report what failed and ask.

## Done

Report:
- Hosts touched.
- Container status (`docker compose ps` output, abridged).
- Public URL and the curl result.
- For updates: the pre-pull SHA so the user can `git reset --hard <sha>` and redeploy to revert.
- The exact `curl` or browser URL the user can run themselves to confirm it's live.
