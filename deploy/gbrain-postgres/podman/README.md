# GBrain PostgreSQL on Podman (Quadlet)

Runs PostgreSQL 16 with `pgvector` as a systemd-managed service using Podman Quadlet and Kubernetes YAML definitions (`podman kube play`).

## Prerequisites

- **Podman >= 4.4** (e.g. Debian 13 trixie, Fedora, RHEL 9). Note: Debian 12 ships with Podman 4.3 which does not support Quadlet `.kube` files.
- `systemd` (either rootful or rootless user session).

## Setup Instructions

### 1. Create the Database Secret

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

To verify the secret was created:
```bash
podman secret ls
```

### 2. Install the Quadlet Files

Copy or symlink both `gbrain-postgres.kube` and `gbrain-postgres.yaml` into the Quadlet directory.

#### Option A: Rootless (Recommended)

Install under your user configuration directory:

```bash
mkdir -p ~/.config/containers/systemd/
cp gbrain-postgres.kube gbrain-postgres.yaml ~/.config/containers/systemd/

# Reload systemd generator and start the service
systemctl --user daemon-reload
systemctl --user start gbrain-postgres

# Enable running at boot without logging in
loginctl enable-linger $USER
```

#### Option B: Rootful

Install system-wide:

```bash
sudo cp gbrain-postgres.kube gbrain-postgres.yaml /etc/containers/systemd/

# Reload systemd generator and start the service
sudo systemctl daemon-reload
sudo systemctl start gbrain-postgres
sudo systemctl enable gbrain-postgres
```

### 3. Verification & Management

Check systemd service status:
```bash
# Rootless
systemctl --user status gbrain-postgres

# Rootful
sudo systemctl status gbrain-postgres
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

## Connecting GBrain

Once running on `127.0.0.1:5432`:

```bash
gbrain init --prefer-postgres
```
