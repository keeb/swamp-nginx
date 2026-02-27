# @keeb/nginx

[Swamp](https://github.com/systeminit/swamp) extension for nginx stream proxy configuration over SSH.

## Models

### `nginx/stream`

Configure nginx as a TCP/UDP stream proxy on a remote host.

| Method | Description |
|--------|-------------|
| `init` | Bootstrap the nginx stream proxy directory structure |
| `configure` | Write a stream proxy config for a backend service and reload nginx |

## Workflows

| Workflow | Description |
|----------|-------------|
| `configure-proxy` | Configure nginx stream proxy for a backend service |

## Dependencies

- [@keeb/ssh](https://github.com/keeb/swamp-ssh) — SSH helpers (`lib/ssh.ts`)

## Install

```bash
swamp extension pull @keeb/nginx
```

## License

MIT
