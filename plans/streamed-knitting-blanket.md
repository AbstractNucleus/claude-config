# DR-3: Decouple docker_service from RCON

## Context

`DockerService.exec_command()` mixes two concerns: resolving container RCON connection info from Docker (env vars, network IP) and sending RCON commands. This makes it impossible to test container ops vs game commands independently, and the 40-line `_get_rcon_ip()` network-joining logic lives inside the container management class where it doesn't belong.

**Goal:** `docker_service` returns connection info; a new `rcon_service` handles command execution. Existing callers get a cleaner async interface.

## Files to modify

| File | Change |
|------|--------|
| `app/services/docker_service.py` | Remove `exec_command()` and `_get_rcon_ip()`. Add `get_rcon_info()` that returns (ip, port, password) tuple |
| `app/services/rcon_client.py` | No changes — already well-isolated |
| **new** `app/services/rcon_service.py` | New module: `execute(container_id, command)` orchestrates docker_service → rcon_client |
| `app/routers/containers/console.py` | Switch from `docker_service.exec_command()` to `rcon_service.execute()` |
| `app/services/mc_players_service.py` | Switch `apply_via_rcon()` from `docker_service.exec_command()` to `rcon_service.execute()` |
| `tests/services/test_docker_service.py` | Update/add test for `get_rcon_info()` |

## Step-by-step

### Step 1 — Add `get_rcon_info()` to docker_service

Extract the env-reading and IP-resolution logic from `exec_command()` into a new public method:

```python
def get_rcon_info(self, container_id: str) -> tuple[str, int, str]:
    """Return (ip, port, password) for RCON connection to a container.
    Raises ValueError if container is not running or RCON is not configured.
    """
```

This keeps network-joining in docker_service (it's Docker-specific logic) but exposes it as data, not behavior.

### Step 2 — Create `app/services/rcon_service.py`

Thin orchestrator:

```python
def execute(container_id: str, command: str) -> tuple[int, str]:
    """Send an RCON command to a container. Returns (exit_code, output)."""
    ip, port, password = docker_service.get_rcon_info(container_id)
    try:
        output = rcon_command(ip, port, password, command)
        return 0, output
    except ConnectionRefusedError:
        return -1, "RCON connection refused — server may still be starting"
    except Exception as e:
        return -1, f"RCON error: {e}"
```

### Step 3 — Remove `exec_command()` and `_get_rcon_ip()` from docker_service

Move `_get_rcon_ip()` to become a private helper within docker_service called only by `get_rcon_info()`. Delete `exec_command()` entirely.

Actually — `_get_rcon_ip()` stays in docker_service since it uses the Docker client. It becomes an internal detail of `get_rcon_info()`.

### Step 4 — Update callers

- `console.py`: `docker_service.exec_command(id, cmd)` → `rcon_service.execute(id, cmd)`
- `mc_players_service.py`: same swap inside `apply_via_rcon()`
- Remove `from app.services.rcon_client import rcon_command` from docker_service

### Step 5 — Update tests

- Existing `test_docker_service.py`: remove/update any `exec_command` tests, add `get_rcon_info` test
- Add `tests/services/test_rcon_service.py` with mocked docker_service and rcon_client

## Verification

1. `python -m pytest tests/ -q` — all tests pass
2. `python -c "from app.services.rcon_service import execute"` — import works
3. `grep -r "exec_command" app/` — no remaining references
4. `grep -r "rcon_command" app/services/docker_service.py` — no RCON import left in docker_service
