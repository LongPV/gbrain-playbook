# GBrain PostgreSQL on Podman (Quadlet)

Runs PostgreSQL 16 with `pgvector` as a systemd-managed service using Podman Quadlet and Kubernetes YAML definitions (`podman kube play`). The same directory also carries a standalone [Hermes Agent](#6-hermes-agent) unit.

## Prerequisites

- **Podman >= 4.4** (e.g. Debian 13 trixie, Fedora, RHEL 9). Note: Debian 12 ships with Podman 4.3 which does not support Quadlet `.kube` files. The Hermes unit needs **Podman >= 4.7** for `AutoUpdate=` in a `.kube` file.
- `systemd` (either rootful or rootless user session).

## Setup Instructions

### 1. Create Secrets

#### 1A. Database Secret

`podman kube play` expects secrets structured as Kubernetes Secrets or JSON maps rather than plain strings. Create the `gbrain-postgres` secret as the user running the service:

```bash
# 1. Generate or supply your database password
read -rs PGPASS   # or: PGPASS=$(openssl rand -hex 24)

# 2. Create the secret in Podman
printf 'apiVersion: v1\nkind: Secret\nmetadata:\n  name: gbrain-postgres\nstringData:\n  password: "%s"\n' \
  "$PGPASS" | podman secret create gbrain-postgres -

# 3. Configure GBrain connection URL
export GBRAIN_DATABASE_URL="postgresql://gbrain:${PGPASS}@127.0.0.1:5432/gbrain"

# 4. Clear the shell variable
unset PGPASS
```

#### 1B. Caddy TLS and Environment Secrets

The stack includes a Caddy reverse proxy. You must create Podman secrets for your TLS certificates and a `.env` file to configure Caddy.

1. **Create your `.env` file** (keep this out of git):
   ```bash
   cat <<EOF > caddy.env
   EXTERNAL_DOMAIN=yourdomain.com
   UPSTREAM_DESTINATION=127.0.0.1:3131
   EOF
   ```

2. **Securely store and import TLS certificates**:
   Ensure your Cloudflare (or other provider) `.pem` and `.key` files are securely stored, e.g., in a password manager. Then create the secrets in Podman:
   ```bash
   # Ensure strict permissions on the key before importing
   chmod 600 cf.key

   # Create the secrets in Podman
   podman secret create caddy-tls-pem cf.pem
   podman secret create caddy-tls-key cf.key
   podman secret create caddy-env caddy.env
   ```
   *Note: After importing into Podman, you can safely remove `cf.key` and `caddy.env` from the disk if you wish.*

To verify the secrets were created:
```bash
podman secret ls
```

### 2. Install the Quadlet Files

Copy or symlink the `.kube` and `.yaml` files into the Quadlet directory.

#### Option A: Rootless (Recommended)

Install under your user configuration directory:

```bash
mkdir -p ~/.config/containers/systemd/
cp *.kube *.yaml ~/.config/containers/systemd/

# Reload systemd generator and start the services
systemctl --user daemon-reload
systemctl --user start gbrain-postgres gbrain-caddy

# Enable running at boot without logging in
loginctl enable-linger $USER
```

#### Option B: Rootful

Install system-wide:

```bash
sudo cp *.kube *.yaml /etc/containers/systemd/

# Reload systemd generator and start the services
sudo systemctl daemon-reload
sudo systemctl start gbrain-postgres gbrain-caddy
sudo systemctl enable gbrain-postgres gbrain-caddy
```

### 3. Verification & Management

Check systemd service status:
```bash
# Rootless
systemctl --user status gbrain-postgres gbrain-caddy

# Rootful
sudo systemctl status gbrain-postgres gbrain-caddy
```

View container logs:
```bash
podman logs -f gbrain-postgres
```

### 4. Direct Testing with Podman (Without Quadlet)

To quickly test the pod without installing systemd units:

```bash
# Start the pod
podman kube play gbrain-postgres.yaml

# Stop and remove the pod
podman kube down gbrain-postgres.yaml
```

### 5. Running the GBrain Server

To run the GBrain web server as a background service that Caddy will proxy traffic to, configure it as a systemd unit. **This is required because the `gbrain-caddy.kube` service automatically links to it and will try to start it.**

*(Note: Adjust the `ExecStart` path in the `gbrain-serve.service` file if `gbrain` is installed elsewhere, such as `/usr/local/bin/gbrain`)*

#### Option A: Rootless (User service)

1. **Install the service file:**
   Create the directory and copy `gbrain-serve.service` to `~/.config/systemd/user/`:
   ```bash
   mkdir -p ~/.config/systemd/user
   cp gbrain-serve.service ~/.config/systemd/user/
   ```

2. **Reload systemd:**
   The Caddy proxy service (`gbrain-caddy`) is configured to automatically start this service when it runs.
   ```bash
   systemctl --user daemon-reload
   ```

3. **Check the logs:**
   ```bash
   journalctl --user -fu gbrain-serve
   ```

#### Option B: Rootful (System service)

1. **Install the service file:**
   Copy `gbrain-serve.service` to `/etc/systemd/system/`:
   ```bash
   sudo cp gbrain-serve.service /etc/systemd/system/
   ```

2. **Reload systemd:**
   The Caddy proxy service (`gbrain-caddy`) is configured to automatically start this service when it runs.
   ```bash
   sudo systemctl daemon-reload
   ```

3. **Check the logs:**
   ```bash
   sudo journalctl -fu gbrain-serve
   ```

### 6. Hermes Agent

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research) runs as its own pod from `hermes-agent.kube` + `hermes-agent.yaml`. It is standalone: it does not depend on, start, or connect to the gbrain units, and its pod network cannot reach Postgres or `gbrain serve` on host loopback.

- **Image:** `docker.io/nousresearch/hermes-agent:latest`, updated by `podman-auto-update`. Upstream retags `:latest` on every push to `main`, so an update is an unreleased commit, not a release. Auto-update rolls back if the restarted unit fails, but not if it starts and misbehaves.
- **State:** the `hermes-data` volume at `/opt/data` holds `config.yaml`, `.env`, `auth.json`, sessions, memories and skills. Back it up with `podman volume export hermes-data`.
- **API:** the OpenAI-compatible API server listens on `127.0.0.1:8642` and requires `Authorization: Bearer $API_SERVER_KEY`. To reach it remotely, use `ssh -L 8642:127.0.0.1:8642 <server>`; it is not published through Caddy.

#### 6A. Create the Secret

The `hermes-agent` secret carries only `API_SERVER_KEY`, which the container needs before Hermes is configured. It must be at least 16 characters or the API server will not start.

```bash
read -rs API_SERVER_KEY      # or: API_SERVER_KEY=$(openssl rand -hex 32)
printf 'apiVersion: v1\nkind: Secret\nmetadata:\n  name: hermes-agent\nstringData:\n  API_SERVER_KEY: "%s"\n' \
  "$API_SERVER_KEY" | podman secret create hermes-agent -
unset API_SERVER_KEY
```

Provider keys (`ANTHROPIC_API_KEY`, a Telegram or Discord token, and so on) do not go in the secret. Hermes loads `/opt/data/.env` over the container environment, so a key in the secret would be shadowed by `.env` anyway. Set them from inside Hermes once it runs; see [6C](#6c-configure-hermes).

#### 6B. Install and Start

Rootless:

```bash
cp hermes-agent.kube hermes-agent.yaml ~/.config/containers/systemd/
systemctl --user daemon-reload
systemctl --user start hermes-agent
systemctl --user enable --now podman-auto-update.timer
```

Rootful: copy to `/etc/containers/systemd/` and drop `--user`. The first start pulls a large image; the unit allows 15 minutes for it.

#### 6C. Configure Hermes

The service starts without a provider key: the gateway and `/health` come up, and chat turns fail with a credentials error until a key is set. Add or change the Anthropic key whenever you are ready. Choose Anthropic, then "Anthropic API key", and paste it at the masked prompt:

```bash
podman exec -it hermes-agent hermes model
```

The key is saved to `/opt/data/.env` in the `hermes-data` volume. The gateway re-reads `.env` and `config.yaml` on every turn, so the next message uses it without a restart. For everything else (chat platforms, tools), run the full wizard with `podman exec -it hermes-agent hermes setup`. The CLI drops to the container's `hermes` user by itself, so files it writes stay readable to the gateway.

Avoid `hermes config set ANTHROPIC_API_KEY <key>`: it works, but the key ends up in the command line and your shell history.

For an unattended gateway, upstream recommends turning on tool-loop hard stops in `/opt/data/config.yaml` (`tool_loop_guardrails.hard_stop_enabled: true`).

#### 6D. Verify

```bash
curl -s http://127.0.0.1:8642/health
curl -s -H "Authorization: Bearer $API_SERVER_KEY" http://127.0.0.1:8642/v1/models
podman logs -f hermes-agent
```

