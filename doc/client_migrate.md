# Per-client Stratum migrate

## Scope

This feature adds admin-API support for:

1. Listing connected Stratum clients as JSON.
2. Asking one client to reconnect to a different `host:port`, then disconnecting that session.

It does **not** change DATUM protocol message `0xA4` (whole-gateway uplink migration between the Gateway and a DATUM pool). That path remains as upstream defines it.

The Gateway does not interpret *why* a move was requested. Any front-end (load balancer, socat, direct bind) or higher-level policy that chooses the destination is outside this feature.

## Why

Operators sometimes need to move a single miner session without restarting the Gateway or migrating the entire DATUM uplink:

- Soft-balancing hashrate across gateways or pool doors
- Draining a gateway for maintenance while other clients stay put
- Lab and failover drills

`kill_client` alone drops the TCP session; many firmwares simply reconnect to the same configured URL. `migrate_client` sends Stratum `client.reconnect` first so cooperative firmware can land on the new endpoint, then closes the old session so stubborn firmware still leaves.

## How

### List sessions

`GET /clients.json` (same digest admin auth as the HTML client dashboard)

Example shape:

```json
{
  "ok": true,
  "count": 1,
  "clients": [
    {
      "tid": 0,
      "cid": 0,
      "unique_id": 123,
      "connect_tsms": 0,
      "username": "bc1qexample.worker",
      "rem_host": "127.0.0.1",
      "subscribed": true,
      "authorized": true
    }
  ]
}
```

`tid` / `cid` identify the session for commands. `username` is the authorized Stratum user (typically payout address plus optional `.worker`). `rem_host` is the peer address as seen by the Gateway process and may be a proxy hop; do not treat it as a stable miner identity.

### Move a session

`POST /cmd` with JSON body (admin password required, same as other `/cmd` actions):

```json
{
  "cmd": "migrate_client",
  "tid": 0,
  "cid": 0,
  "host": "stratum.example",
  "port": 3333,
  "password": "…"
}
```

On the client thread the Gateway:

1. Sends `client.show_message` with a short notice including the destination.
2. Sends `client.reconnect` with `[host, port, 0]`.
3. Flushes the pending write buffer immediately (the normal send loop would otherwise run after a kill closes the fd).
4. Briefly waits, then sets the usual kill path so the socket is closed.

`host` / `port` must be reachable **from the miner**, not necessarily from the Gateway host.

## Code changes

| File | Change |
|------|--------|
| `src/datum_api.h` / `src/datum_api.c` | `datum_api_clients_json`, `datum_api_cmd_migrate_client`, `/clients.json` route, `/cmd` `migrate_client` |
| `src/datum_sockets.h` / `src/datum_sockets.c` | Per-client `migrate_request` / `migrate_host` / `migrate_port`; reconnect + flush before kill |

## Notes

- Prefer matching sessions by Stratum username (payout identity), then passing the current `tid`/`cid` into `migrate_client`. Slot ids are ephemeral.
- Firmwares that ignore `client.reconnect` will still be disconnected; the miner’s configured URL then decides where it returns.
- This feature is intended to work on stock Gateway deployments generally; it does not require a particular pool topology.
