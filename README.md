# Namecoin Full-Stack: namecoind + ElectrumX + Mempool

A single `compose.yaml` that runs a complete Namecoin infrastructure stack using podman rootless.

## Architecture

```
namecoind (port 8336 RPC, 8334 P2P)
  |-- electrumx        -> indexes blockchain, serves Electrum clients on :50001
  |-- mempool-api       -> block explorer backend, serves frontend
  |     |-- mempool-db  -> MariaDB for mempool stats
  |-- mempool-web       -> block explorer UI on :8080
```

All services communicate over an internal compose network. Only P2P (8334), ElectrumX (50001), and the mempool frontend (8080) are exposed to the host.

## Services

| Service | Description | Port |
|---------|-------------|------|
| `namecoind` | Namecoin Core 30.2, built from source with `txindex=1` | 8334 (P2P) |
| `electrumx` | ElectrumX 1.19.0 for Namecoin with LevelDB | 50001 (TCP), 50002 (SSL) |
| `mempool-web` | Mempool block explorer frontend | 8080 (HTTP) |
| `mempool-api` | Mempool backend connected to namecoind RPC | internal |
| `mempool-db` | MariaDB 10.5 for mempool data | internal |

## Prerequisites (Debian/Ubuntu)

### Install podman and podman-compose

```bash
sudo apt-get update
sudo apt-get install -y podman podman-compose
```

### Configure podman rootless

Podman rootless on Debian requires cgroup and registry configuration:

```bash
# Enable user lingering (allows containers to run without active login)
loginctl enable-linger $(whoami)

# Configure cgroupfs (needed if systemd user session is unavailable, e.g. SSH)
mkdir -p ~/.config/containers/
cat > ~/.config/containers/containers.conf <<EOF
[engine]
cgroup_manager = "cgroupfs"
events_logger = "file"
EOF

# Add Docker Hub as default image registry
cat > ~/.config/containers/registries.conf <<EOF
unqualified-search-registries = ["docker.io"]
EOF
```

### SSL certificates directory

Create the cert mount point (required even without SSL):

```bash
sudo mkdir -p /etc/electrumx-certs
```

See [SSL with Caddy](#ssl-with-caddy) for setting up TLS certificates.

## Quick Start

```bash
git clone <repo-url>
cd <repo-name>
podman compose build
podman compose up -d

# Wait ~30s for mempool-api to create tables, then patch for Namecoin AuxPoW headers:
sleep 30 && podman exec $(podman ps -qf "name=mempool-db") \
  mysql -u mempool -pmempool mempool \
  -e "ALTER TABLE blocks MODIFY COLUMN header TEXT NOT NULL;"
```

The namecoind build compiles from source (~30 min). After that, the blockchain sync takes several hours from scratch.

## Using an Existing Namecoin Data Directory

If you already have a synced namecoin node, you can reuse its data to skip the initial sync.

1. Stop your existing namecoind
2. Replace the named volume with a bind mount in `compose.yaml`:

```yaml
namecoind:
  volumes:
    - /path/to/.namecoin:/home/namecoin/.namecoin
```



## Namecoin AuxPoW Compatibility

Mempool was designed for Bitcoin's fixed 80-byte block headers. Namecoin uses AuxPoW (merge-mining with Bitcoin), which produces variable-length headers much larger than 80 bytes.

After first deploy, you must patch the `blocks.header` column from `varchar(160)` to `TEXT`:

```bash
podman exec $(podman ps -qf "name=mempool-db") \
  mysql -u mempool -pmempool mempool \
  -e "ALTER TABLE blocks MODIFY COLUMN header TEXT NOT NULL;"
```

This only needs to run once — the `mariadb-data` volume persists the change across restarts.

## Known display disparities (Mempool ↔ Namecoin)

This stack runs the **upstream `mempool/frontend` and `mempool/backend` images unmodified**. Mempool was built for Bitcoin and has no Namecoin awareness. Several pieces of UI are therefore inaccurate when pointed at namecoind. They are documented here rather than silently misleading users.

### Fixed in this compose

| Symptom | Why it happens | Fix in `compose.yaml` |
|---|---|---|
| Mempool shows a USD price for every fee/amount, e.g. `100 sat/vB ≈ $10.82` | Backend fetches BTC/USD from Kraken/Coinbase and applies it to all sats. BTC is ~$108k, NMC is ~$1.50, so fiat values are off by ~70 000×. | `FIAT_PRICE_ENABLED=false` on `mempool-api`, `HISTORICAL_PRICE=false` on `mempool-web`. |

### Not fixed (require a forked Mempool image)

| Symptom | Why it happens | Where the real fix lives |
|---|---|---|
| Fee rate labelled `sat/vB` instead of `swartz/vB` | The string `sat/vB` and the symbol `₿` are hard-coded in 50+ frontend files. There is no env var for the smallest-unit name. | Fork `mempool/mempool` frontend; replace unit/label strings throughout. |
| Transaction amounts shown as `BTC` / `₿` rather than `NMC` / `ℕ` | Same as above — currency labels are hard-coded. The 8-decimal scaling itself is correct (NMC also has 8 decimals). | Fork frontend; replace currency strings. |
| **Every block shows miner = `Unknown`** | Namecoin is merge-mined with Bitcoin. The real miner identity lives in the **parent Bitcoin coinbase** carried inside the AuxPoW header — not in the Namecoin coinbase that mempool inspects. Mempool's `pools-v2.json` regexes never match the Namecoin coinbase, so every block falls through to the `unknown` pool. | Backend change: extract the parent-chain coinbase from the AuxPoW block header before running pool-tag detection. Filed upstream — see linked tracking issue. |

A NMC-aware Mempool fork (covering the unit labels, currency labels and AuxPoW pool detection) is a multi-day undertaking and tracked separately. Until then the items above will continue to show Bitcoin-style data.

## SSL with Caddy

ElectrumX serves SSL on port 50002 using certificates from `/etc/electrumx-certs/`. If you have Caddy managing TLS for your domain:

```bash
CADDY_CERTS="/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/your.domain.com"

# Make Caddy cert path traversable for podman rootless
sudo chmod o+x /var/lib/caddy /var/lib/caddy/.local /var/lib/caddy/.local/share \
  /var/lib/caddy/.local/share/caddy /var/lib/caddy/.local/share/caddy/certificates \
  /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory
sudo chmod o+rx "$CADDY_CERTS"
sudo chmod o+r "$CADDY_CERTS"/*

# Symlink certs (auto-updates on Caddy renewal)
sudo mkdir -p /etc/electrumx-certs
sudo ln -sf "$CADDY_CERTS/your.domain.com.crt" /etc/electrumx-certs/cert.pem
sudo ln -sf "$CADDY_CERTS/your.domain.com.key" /etc/electrumx-certs/key.pem
```

For the mempool frontend, add a reverse proxy in your Caddyfile:

```
mempool.yourdomain.com {
    reverse_proxy localhost:8080
}
```

ElectrumX doesn't auto-reload certs on renewal. Add a cron job to restart the container after Caddy renews (~every 60 days):

```bash
0 3 1 */2 * podman restart electrumx-docker_electrumx_1
```

If you don't need SSL, remove the `SSL_*` environment variables, the `/etc/electrumx-certs` volume mount, and the port 50002 mapping from `compose.yaml`.

## Monitoring

```bash
# Check all container statuses
podman ps -a --filter "name=electrumx-docker"

# Check namecoind sync progress
podman exec electrumx-docker_namecoind_1 \
  namecoin-cli -rpcuser=username -rpcpassword=password \
  -conf=/etc/namecoin/namecoin.conf getblockchaininfo

# Check peer count
podman exec electrumx-docker_namecoind_1 \
  namecoin-cli -rpcuser=username -rpcpassword=password \
  -conf=/etc/namecoin/namecoin.conf getconnectioncount

# Check electrumx logs
podman logs electrumx-docker_electrumx_1 --tail 10

# Check mempool API
curl http://localhost:8080/api/v1/backend-info
```

## Volumes

| Volume | Contents |
|--------|----------|
| `namecoin-data` | Blockchain, chainstate, txindex (~15 GB) |
| `electrumx-data` | ElectrumX index database |
| `mempool-cache` | Mempool backend cache |
| `mariadb-data` | MariaDB database files |

## Configuration

- **RPC credentials**: Set in `namecoin/namecoin.conf` and referenced in `compose.yaml`. Internal only, not exposed to the network.
- **Seed nodes**: Automatically populated from `namecoin-core/contrib/seeds/nodes_main.txt` during the Docker build.
- **Namecoin config**: Stored at `/etc/namecoin/namecoin.conf` inside the container (outside the data volume so config updates don't require volume deletion).
