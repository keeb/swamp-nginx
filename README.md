# @keeb/nginx

[Swamp](https://github.com/systeminit/swamp) extension for nginx stream proxy configuration over SSH.

## Models

### `nginx/stream`

Configure nginx as a TCP/UDP stream proxy on a remote host.

| Method | Description |
|--------|-------------|
| `init` | Bootstrap the nginx stream proxy directory structure and start the container |
| `configure` | Write a stream proxy config for a backend service and reload nginx |

Global arguments: `sshHost`, `sshUser` (default `keeb`), `streamDir` (default `~/stream`).

## Workflows

| Workflow | Description |
|----------|-------------|
| `configure-proxy` | Configure nginx stream proxy for a backend service |

## Dependencies

- [@keeb/ssh](https://github.com/keeb/swamp-extensions) — SSH helpers (`lib/ssh.ts`)

## Install

```bash
swamp extension pull @keeb/nginx
```

## Usage

Bootstrap the proxy host once, then add a backend. The `portMap` format is `listen:backend[/proto]` with entries comma-separated — `udp` proto is explicit, everything else defaults to `tcp`.

```bash
# Bootstrap the proxy host (creates ~/stream and starts the container)
swamp model method run nginx-proxy init

# Forward Minecraft (tcp 25565) and Terraria (udp 7777) to a backend
swamp model method run nginx-proxy configure \
  --args vmName=games,targetIp=100.64.0.5,portMap='25565:25565,7777:7777/udp'
```

## License

MIT — see [LICENSE](./LICENSE).
