# gen

A port-free intranet penetration (NAT traversal) tool — rathole-like, written in Go with **zero third-party dependencies**. One static binary acts as both server and client.

> English docs. 中文文档见 [README.md](./README.md)

## Why gen

- **rathole**: besides the control channel (e.g. 2333), every service must be registered with `bind_addr` / `local_addr` — each new service means editing config and restarting.
- **gen**: the control port (default 2333) is **only used to establish the tunnel**. Data ports need **no per-service registration** — whatever port you hit on the public server is forwarded **1:1 (port-preserving)** to the same port on the client's localhost. Port numbers never change.

```
curl public-ip:80   ──▶   server listening on 80  ── control channel ──▶   client dials 127.0.0.1:80
curl public-ip:443  ──▶   server listening on 443 ── control channel ──▶   client dials 127.0.0.1:443
```

## Quick start

```bash
# Server (on the machine with a public IP)
./gen server -token YOUR_TOKEN          # control port defaults to 2333, data ports auto

# Client (on the intranet machine — only IP + token, no port needed)
./gen client -server PUBLIC_IP -token YOUR_TOKEN
```

The client logs:

```
隧道已建立 (tunnel established): channel 1.2.3.4:2333 ...
```

Now hitting `public-ip:80` reaches port `80` on the client machine.

**Firewall/security group**: allow the service ports you want (e.g. 80/443/22) plus control port 2333 — as usual.

## Options

### Server

| Flag | Default | Description |
|------|---------|-------------|
| `-token` | (required) | Access token; also `GEN_TOKEN` env var |
| `-port` | `2333` | **Control port**: used only to establish the tunnel (customizable) |
| `-ports` | `auto` | Data port range. `auto` = 1-65535 excluding the OS ephemeral range (Linux reads `/proc/sys/net/ipv4/ip_local_port_range`, usually 32768-60999; Windows/macOS exclude 49152-65535); or explicit, e.g. `-ports 1-65535` |
| `-bind` | `0.0.0.0` | Listen address |
| `-config` | auto-detect | Config file path (INI, see below) |

### Client

| Flag | Default | Description |
|------|---------|-------------|
| `-server` | (required) | Server IP/domain, optionally `:control-port` (defaults to 2333); also `GEN_SERVER` env var |
| `-token` | (required) | Access token; also `GEN_TOKEN` env var |
| `-local-host` | `127.0.0.1` | Local service address (port stays the same, no need to fill) |
| `-channels` | `4` | Number of parallel data channels (`0` = single-channel fallback, `1-64`). Splits the data plane across multiple TCP streams: total throughput ≈ per-stream limit × channels; auto-degrades with an old server |
| `-config` | auto-detect | Config file path (INI, see below) |

### Config file (INI)

Both sides can put options in a config file — the command line can then be zero-arg. Client example:

```ini
[config]
remote_addr = "public-ip"     # no port → defaults to 2333; or "public-ip:port"
default_token = "your-token"
# local_host = "127.0.0.1"    # optional
# channels = 4                # optional: parallel data channels (0-64)
```

Server:

```ini
[config]
default_token = "your-token"
# port = 2333                 # control port, optional
# ports = "auto"              # data port range, optional
# bind = "0.0.0.0"            # listen address, optional
```

- Use `-config path` to specify; otherwise auto-detect `./gen.ini`, then `/etc/gen/gen.ini`;
- `#` / `;` comments supported, values may be quoted;
- **Precedence: flag > config file > env var > default**.

### Environment variables

| Variable | Purpose | Precedence |
|----------|---------|------------|
| `GEN_TOKEN` | token fallback | below `-token` / config file |
| `GEN_SERVER` | client server-address fallback | below `-server` / config file |

## How it works

1. **Establish the channel**: the client connects to the server control port (default 2333), sends
   `TOKEN <token>` for auth, forming one control channel. Non-handshake connections on that port
   are treated as data connections (port-preserving forwarding).
2. **Parallel data channels (multi-channel)**: after auth, the client opens N (default 4, `-channels`
   tunable, 0 = off) additional data channels to the control port (`DATA <token> <index>` registration).
   Each forwarded stream is pinned to one channel; its bidirectional `DATA/CLOSE` frames travel only on
   that channel, while `OPEN/PING/PONG/KICK` stay on the control channel. This removes the per-TCP-stream
   throughput ceiling — **total ≈ per-stream limit × channels** — and a retransmission stall on one
   channel no longer blocks other streams.
3. **Port-preserving 1:1 forwarding**: the server listens on every data port. When a public connection
   arrives on port N, the server sends an `OPEN` frame (port N + chosen channel index) over the control
   channel; the client dials `127.0.0.1:N` and relays both directions. Port numbers never change, so no
   service ever needs registering.
4. Frame protocol `OPEN / DATA / CLOSE / PING / PONG / KICK`; client heartbeats every 30 s (control
   channel + every data channel), server declares the session dead after 90 s of silence; automatic
   exponential-backoff reconnect on disconnect.
5. **Channel self-healing**: when one data channel drops, both ends only tear down the streams pinned to
   it, and the client rebuilds a replacement channel — the control channel and other channels are
   unaffected. Old servers (no data channels) → the client auto-falls-back to single-channel; old clients
   against a new server simply never open data channels. Fully backward/forward compatible.
6. **One client per token**: a new client replaces the old session; the old client gets a `KICK` and exits.
7. **Port conflicts**: a data port already bound by another local service is silently skipped — not taken
   over and not reported (ownership stays with the original service; subsequent conflicts are Linux's
   own `address already in use`). One summary line is printed after startup. A busy **control port**
   terminates startup with the raw OS error.

> Why the ephemeral range is excluded by default: outbound connections (DNS, apt, outgoing SSH) need
> ephemeral source ports. Binding all of 1-65535 would starve the server's own outbound connections.
> So `-ports auto` excludes the OS ephemeral range, covering all regular service ports
> (1-32767 + 61000-65535) without breaking the server's own networking. Use `-ports 1-65535` only if
> you accept that trade-off.

## Build

Requires Go 1.21+:

```bash
go build -o gen .
```

Cross-compile for all platforms:

```bash
./build.sh
```

Outputs: Linux / Windows / macOS × amd64 / arm64. The server listens on a huge number of ports and
automatically raises its file-descriptor soft limit to the hard limit (Linux); use `-ports` to narrow
the range in restricted environments.

## systemd deployment (Linux server, auto-start after boot + network ready)

The repo ships `gen.service` (server) and `gen-client.service` (client).

```bash
# 1. Install the binary (pick amd64/arm64 by architecture: uname -m; x86_64→amd64, aarch64→arm64)
sudo install -m 755 gen-0.5.0-linux-amd64 /usr/local/bin/gen

# 2. Install the unit (server: gen.service; on the client machine use gen-client.service)
sudo cp gen.service /etc/systemd/system/gen.service

# 3. Create the config (0600, keeps the token out of the process command line)
sudo mkdir -p /etc/gen
sudo tee /etc/gen/gen.ini <<'EOF'
[config]
default_token = "your-token"
EOF
sudo chmod 600 /etc/gen/gen.ini

# 4. Enable at boot and start now
sudo systemctl daemon-reload
sudo systemctl enable --now gen
sudo systemctl status gen
```

Unit highlights:

- `Wants=network-online.target` + `After=network-online.target`: starts **after the network is up**
  (DHCP/static addressing done);
- `Restart=always` + `RestartSec=5`: auto-restart on crash;
- `LimitNOFILE=1048576`: raises the fd hard limit (the server holds tens of thousands of ports);
- config is read via `-config /etc/gen/gen.ini`, never on the command line.

## Testing

```bash
./scripts/e2e.sh         # end-to-end: forwarding/conflict/big-file/concurrency/multi-channel/fallback/self-healing
./scripts/config-test.sh # config: INI parsing/bare-address default port/precedence/autoload/channels
```

Coverage: 1:1 port-preserving forwarding (two services, no registration), port-conflict skip with a
single summary line, 4 MB file integrity, 12-way concurrent multiplexing, correct close when the local
service is absent, wrong-token rejection, same-token replacement, auto-reconnect, correct close with no
client online, multi-channel establishment and transfer, `-channels 0` single-channel fallback, and
single-channel drop self-healing.

## vs rathole

| | rathole | gen |
|---|---|---|
| Service registration | each service: `bind_addr` / `local_addr` | **zero**, 1:1 auto-forward |
| Security group | add each service port manually | open service ports + control port as usual |
| Control port | control channel (e.g. 2333) | 2333 (only for establishing the channel, customizable) |
| Config file | TOML per side | optional INI (`gen.ini`), flags suffice |
| Multi-client | supported | one client per token (last connect wins) |
| Language | Rust | Go (no third-party deps) |

## Security notes

- token and traffic are **plaintext** — use only on trusted networks; for public production wrap with
  TLS or WireGuard, or use rathole's TLS mode.
- whoever has the token owns every port of your intranet: use a strong random token, and restrict the
  control port to trusted source IPs in the firewall/security group.
- gen takes over all *free* ports in the data range at startup (occupied ports are untouched).

## Known limitations

- No TCP half-close: EOF on one side closes the whole connection (fine for HTTP etc.).
- Ports must map 1:1 (public N → client local N); no port remapping.
- Throughput ceiling is "per-TCP-stream limit × channels": the default 4 channels already exceed typical
  links; on high-latency (>100 ms) links, raise `-channels` (e.g. 16). OS-level tuning in
  `scripts/tuning.md` (TCP window + BBR).

## Throughput tuning

A single TCP stream is limited by ≈ window ÷ round-trip time. To saturate a large pipe:

1. enlarge the TCP window and enable BBR on both ends (see `scripts/tuning.md`);
2. raise `-channels` on the client (each channel = an independent window);
3. the real ceiling is always the link itself (cloud bandwidth plan / home broadband).

## License

MIT
