# kodeart license-agent

The license-agent runs on each node (one agent per deployment/node). It activates this node, heartbeats and fetches a signed license from the license-server, verifies it offline, and exposes the current `AgentStatus` to local application replicas via gRPC on `127.0.0.1:6101`. It also provides a local health endpoint on `127.0.0.1:6100/healthz`.

## Container image
- `ghcr.io/kodeart/license-agent:<version>` (public). Image contains only the statically built binary.
- Port 6100 (health HTTP) — bind locally. Port 6101 (gRPC) — node-local only.
- Healthcheck: `GET 127.0.0.1:6100/healthz` → `200 {"status":"ok"}`.

## Configuration

| Variable | Default | Required | Description |
|---|---|---|---|
| `LICENSE_GRPC_ADDRESS` | — | yes | license-server gRPC endpoint (e.g. `license.example.com:50053`) |
| `LICENSE_ACTIVATION_CODE` | — | no* | One-time activation credential. Only needed on first boot; persisted state is reused. |
| `LICENSE_PRODUCT_CODE` | — | no** | Product code for this deployment (required when activating) |
| `LICENSE_PUBLIC_KEYRING_PATH` | — | yes† | Directory containing `*.pub` public keys (preferred; survives key rotation). |
| `LICENSE_PUBLIC_KEY_PATH` | — | yes† | Single public key file (fallback). |
| `LICENSE_AGENT_STATE` | `/var/lib/license-agent/state.json` | no | Path to persisted state (deployment id, api key, cached tokens). |
| `LICENSE_AGENT_LISTEN` | `127.0.0.1:6100` | no | Local HTTP health listen address. |
| `LICENSE_AGENT_GRPC_LISTEN` | `127.0.0.1:6101` | no | Local gRPC listen address for app replicas. |
| `LICENSE_INSECURE` | `false` | no | Dial plaintext (dev/staging only). |
| `LICENSE_SERVER_NAME` | — | no | SNI/authority override when fronting via proxy. |
| `LICENSE_HOSTNAME` | (os.Hostname) | no | Deployment hostname label. |
| `LICENSE_HEARTBEAT_INTERVAL` | `60s` | no | Heartbeat cadence. |
| `LICENSE_CRL_INTERVAL` | `5m` | no | CRL refresh interval. |
| `LICENSE_CONNECT_TIMEOUT` | `3s` | no | gRPC connect timeout. |
| `LICENSE_KEEPALIVE_TIME` | `20s` | no | gRPC keepalive time. |

\* Required on first boot if no persisted state.  
\*\* Required during activation.  
† Either `LICENSE_PUBLIC_KEYRING_PATH` or `LICENSE_PUBLIC_KEY_PATH` must be set.

## Docker example
```bash
docker run --rm \
  --name license-agent \
  -p 127.0.0.1:6100:6100 \
  -p 127.0.0.1:6101:6101 \
  -v /etc/license-agent/keys:/etc/license-agent/keys:ro \
  -v /var/lib/license-agent:/var/lib/license-agent \
  -e LICENSE_GRPC_ADDRESS=license.example.com:50053 \
  -e LICENSE_ACTIVATION_CODE=YOUR-CODE \
  -e LICENSE_PRODUCT_CODE=your-product \
  -e LICENSE_PUBLIC_KEYRING_PATH=/etc/license-agent/keys \
  ghcr.io/kodeart/license-agent:vX.Y.Z
```

## Notes
- Persisted state avoids re-activation. Remove it only to re-activate this node.
- `/healthz` is for supervision. Apps should use the local gRPC API to read `AgentStatus`.
- 6101 is **node-local** — never expose to other machines.
