---
name: nginx
description: Configure nginx as a TCP/UDP stream proxy on a remote host over SSH using the @keeb/nginx swamp extension. Use when the user wants to set up an nginx reverse proxy, forward TCP or UDP ports to a backend, expose a Tailscale service publicly, bootstrap a stream proxy server, run the nginx/stream model, invoke its `init` or `configure` methods, or run the `configure-proxy` workflow. Triggers on "nginx stream proxy", "stream proxy", "tcp proxy", "udp proxy", "port forward", "expose tailscale service", "@keeb/nginx", "nginx/stream", "configure-proxy workflow".
---

# @keeb/nginx skill

Swamp extension that configures an nginx stream (Layer 4 TCP/UDP) proxy on a
remote host over SSH. The proxy runs as a Docker Compose service and routes
public ports to backend services (typically Tailscale IPs).

## Model: `@user/nginx/stream`

Type identifier in workflows: `@user/nginx/stream`. Define an instance with a
short name (e.g. `streamProxy`) and reference it from workflow steps.

### Global arguments

| Field       | Required | Default    | Notes                             |
| ----------- | -------- | ---------- | --------------------------------- |
| `sshHost`   | yes      | —          | SSH hostname of the proxy server  |
| `sshUser`   | no       | `keeb`     | SSH user                          |
| `streamDir` | no       | `~/stream` | Directory holding nginx + compose |

### Methods

#### `init`

Bootstraps the proxy host. No arguments. Creates `<streamDir>/stream.d/`, writes
a base `nginx.conf` and `docker-compose.yml`, then runs `docker compose up -d`.
Run once per host before using `configure`.

Writes resource `server` (data name `server`) —
`{ success, streamDir, timestamp }`.

#### `configure`

Adds (or updates) a per-service stream config and reloads the proxy container.

Arguments:

| Field      | Type   | Description                                                            |
| ---------- | ------ | ---------------------------------------------------------------------- |
| `vmName`   | string | Service name; used as the config filename `<vmName>-nginx.conf`        |
| `targetIp` | string | Backend IP (typically a Tailscale IP)                                  |
| `portMap`  | string | Comma-separated `listen:backend[/proto]` entries; proto defaults `tcp` |

`portMap` examples:

- `25565:25565` — TCP, listen and backend port both 25565
- `7777:7777/udp` — UDP
- `25565:25565,7777:7777/udp,8080:80` — mixed

The method writes `<streamDir>/stream.d/<vmName>-nginx.conf`, then patches
`docker-compose.yml` to publish any new ports (deduped against existing ones,
handling the `ports: []` empty-array case), then runs `docker compose up -d`.

Writes resource `proxy` (data name `proxy`) —
`{ success, vmName, portsAdded, configWritten, timestamp }`.

Both resources use `lifetime: infinite` and `garbageCollection: 10`.

## Workflow: `configure-proxy`

End-to-end pipeline that resolves a Tailscale service name to an IP and points
the stream proxy at it. Inputs:

| Input     | Description                                                    |
| --------- | -------------------------------------------------------------- |
| `vmName`  | Service name (e.g. `allthemons`); must match a tailnet machine |
| `portMap` | Same format as the `configure` method                          |

Steps:

1. `tailscale-status` — runs `tailscale status --json` via the `tailscaleStatus`
   model (a `command/shell` instance).
2. `sync-machines` — calls `tailnet.sync` with the JSON from step 1.
3. `configure-proxy` — calls `streamProxy.configure` with the resolved Tailscale
   IP from
   `${{ model.tailnet.resource.machine[inputs.vmName].attributes.tailscaleIp }}`.

The workflow expects three model instances to exist in the repo:
`tailscaleStatus`, `tailnet`, and `streamProxy`.

## Defining a model instance

```yaml
# .swamp/models/<id>.yaml (or via `swamp model create`)
type: "@user/nginx/stream"
name: streamProxy
globalArguments:
  sshHost: proxy.example.ts.net
  sshUser: keeb
  streamDir: ~/stream
```

## Running directly from the CLI

```bash
# One-time bootstrap of the proxy host
swamp model run streamProxy init

# Add a service
swamp model run streamProxy configure \
  --vmName allthemons \
  --targetIp 100.64.1.23 \
  --portMap '25565:25565,7777:7777/udp'

# Or run the full workflow (resolves the IP from tailnet data)
swamp workflow run configure-proxy \
  --vmName allthemons \
  --portMap '25565:25565,7777:7777/udp'
```

## CEL access to results

Prefer `data.latest`:

```cel
data.latest("streamProxy", "proxy").attributes.configWritten
data.latest("streamProxy", "server").attributes.streamDir
```

The workflow uses the legacy `model.<name>.resource.<spec>...` form for the
tailnet lookup; new wiring should use `data.latest` per repo CLAUDE.md rule 4.

## Dependencies

- **`@keeb/ssh`** — declared in `manifest.yaml`. The model imports
  `./lib/ssh.ts` which provides `sshExec`, `sshExecRaw`, `waitForSsh`. SSH runs
  with `StrictHostKeyChecking=no` and `UserKnownHostsFile=/dev/null`.
- **Tailscale (workflow only)** — the `configure-proxy` workflow assumes a
  `tailscaleStatus` shell model and a `tailnet` model are present in the repo.
  Not required when calling `configure` directly with a known IP.
- **Docker + Docker Compose** must be installed on the proxy host. The `init`
  method runs `docker compose up -d`; it does not install Docker for you.
- **SSH key access** to `sshUser@sshHost` must already work — there is no
  password prompt support and no vault credential is read by this model.

## Gotchas

- **Run `init` first.** `configure` assumes `streamDir/stream.d/` and
  `docker-compose.yml` exist. There is no auto-bootstrap fallback.
- **`vmName` is the filename.** Reusing a `vmName` overwrites
  `<vmName>-nginx.conf` — that is the intended update path. Picking two
  different `vmName`s with overlapping listen ports yields conflicting
  `server { listen <port>; }` blocks and nginx will fail to reload.
- **Port dedupe is per protocol.** `25565` TCP and `25565` UDP are distinct
  keys; both will be published to the host.
- **Compose port insertion is regex-based.** It looks for `ports: []` first,
  otherwise appends after the last `- 'NNNN:NNNN'` line. If you hand-edit
  `docker-compose.yml` into a non-standard shape the patcher may silently no-op
  (`updatedCompose` stays undefined).
- **`sshExec` throws on non-zero exit** with the last 500 chars of stderr.
  `init` and `configure` make several sequential SSH calls; a mid-sequence
  failure leaves the host partially configured.
- **Container name is fixed** to `stream-proxy`. Only one instance of this model
  per host.
- **Default `sshUser` is `keeb`.** Override `globalArguments.sshUser` for any
  other host.
- **No `AbortSignal` plumbing.** SSH calls run to completion; long hangs are
  bounded only by SSH `ConnectTimeout=10` for the initial handshake.
