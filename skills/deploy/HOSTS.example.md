# HOSTS — your deploy targets

Copy this file to `HOSTS.md` (same folder) and edit it. `HOSTS.md` is gitignored; the example is committed so it survives setup on a new machine.

The `/deploy` skill reads this file verbatim to plan deploys. Be concrete. Use the same hostnames you use in `~/.ssh/config` so SSH "just works".

---

## aserver

- **Role:** nginx gateway. Public-facing.
- **SSH:** `ssh noel@aserver` (alias in `~/.ssh/config`)
- **Reachability:** WAN ports 80 + 443 forwarded from the home router to this host.
- **Lives here:**
  - nginx config: `/etc/nginx/sites-available/`, enabled via symlink in `/etc/nginx/sites-enabled/`.
  - TLS certs: managed by `certbot`, auto-renewed.
- **Routing convention:** one `sites-available/<project>.conf` per app, `proxy_pass http://bserver:<port>;`.
- **Reload command:** `sudo nginx -t && sudo systemctl reload nginx`.

## bserver

- **Role:** Docker host. Runs all application containers.
- **SSH:** `ssh noel@bserver`
- **Lives here:**
  - Repos: `/home/noel/repos/<project>/`
  - Containers: `docker compose`, one stack per project, scoped with `-p <project>`.
- **Convention:** `cd /home/noel/repos/<project> && git pull && docker compose -p <project> up -d --build`.
- **Ports:** Each app picks a unique internal port. When adding a new app, also update the corresponding nginx config on `aserver`.

---

## How to extend this file

- **Add a host:** new `##` heading + the same fields. Be explicit about role and conventions — vague entries produce vague deploys.
- **Use SSH aliases**, not raw IPs. Set them up in `~/.ssh/config`.
- **Never paste passwords or private keys here.** The AI reads this file.
- **Project-specific overrides:** if one project deploys differently from your convention, write it as a sub-bullet under the host where it lives.
