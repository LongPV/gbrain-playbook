# GBrain PostgreSQL on Docker Compose

Runs PostgreSQL 16 with `pgvector` locally via Docker Compose.

## Usage

Start the container in the background:

```bash
docker compose up -d
```

Or from the repository root:

```bash
docker compose -f deploy/gbrain-postgres/docker-compose/docker-compose.yml up -d
```

## Configuration

Defaults configured in `docker-compose.yml`:
- **Image**: `pgvector/pgvector:pg16`
- **Port**: `127.0.0.1:5432:5432`
- **Database**: `gbrain`
- **User**: `gbrain`
- **Password**: `gbrainpgpassword`
- **Volume**: `gbrain_db_data` mounted to `/var/lib/postgresql/data`

## Connecting GBrain

```bash
export GBRAIN_DATABASE_URL=postgresql://gbrain:gbrainpgpassword@localhost:5432/gbrain
gbrain init --prefer-postgres
```

## Management

View container logs:
```bash
docker compose logs -f postgres
```

Stop the container:
```bash
docker compose down
```
