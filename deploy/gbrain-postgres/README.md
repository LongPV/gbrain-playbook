# GBrain PostgreSQL Deployments

Deployment configurations for running PostgreSQL 16 with the `pgvector` extension for GBrain.

## Deployment Options

| Method | Directory | Use Case |
|---|---|---|
| **Podman (Quadlet)** | [`podman/`](podman/) | Production / Linux servers (systemd integrated, rootless container management) |
| **Docker Compose** | [`docker-compose/`](docker-compose/) | Local development on macOS / Docker Desktop |

See each directory's `README.md` for specific instructions and credentials configuration.
