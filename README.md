# homelab-grafana

Grafana instance for a homelab monitoring dashboard. Docker-based, scraping from the same `homelab-prometheus` stack and exposing dashboards via `localhost:3000`.

## Configuration

Copy the example config and edit it to your liking before deploying:

```bash
cp config/grafana.ini.example config/grafana.ini
# Edit config/grafana.ini with your preferred settings
```

The default covers a standalone homelab deploy (SQLite, local login on `0.0.0.0:3000`). For production setups you'll likely want to:

- Switch the database type to PostgreSQL or MySQL
- Set `secret_key` and a strong admin password
- Disable sign-ups and logins (`disable_login_form = true`) behind an SSO / auth proxy

## Quick start

### Deploy with docker-compose (recommended)

```bash
cp config/grafana.ini.example config/grafana.ini   # edit to taste
docker compose up -d
```

This mounts your `config/grafana.ini` into the container and persists dashboard data in a named volume. Browse to `http://localhost:3000` (default credentials: `admin/admin`).

#### Volume mount reference

| Container path | Purpose | Compose key |
|---|---|---|
| `/etc/grafana/grafana.ini` | Grafana configuration file | `./config/grafana.ini:/etc/grafana/grafana.ini:ro` |
| `/var/lib/grafana` | Dashboard & user data persistence | `grafana-data:/var/lib/grafana` |

#### Custom ports and volumes (docker-compose)

Edit `docker-compose.yml` directly — for example, to expose Grafana on port `3001`:

```yaml
ports:
  - "3001:3000"
```

Or switch the data volume to a bind mount (e.g. on an external disk):

```yaml
volumes:
  - /mnt/data/grafana:/var/lib/grafana
```

After editing, re-run `docker compose up -d`.

#### Standalone `docker run` flags

The equivalent CLI flags for the compose volume mappings look like this:

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v "$(pwd)/config/grafana.ini:/etc/grafana/grafana.ini:ro" \
  -v grafana-data:/var/lib/grafana \
  ghcr.io/simplylimitless/homelab-grafana:latest
```

| Flag | Maps to | Example |
|---|---|---|
| `-p` or `--publish` | Port mapping (host:container) | `-p 3001:3000` for a non-default host port |
| `-v` or `--volume` | Bind mount or named volume | See table above |

### Deploy from GitHub Container Registry

```bash
docker login ghcr.io -u YOUR_GITHUB_USERNAME   # password: a GitHub PAT with `packages` scope (create one at https://github.com/settings/tokens, not the fine-grained type)
docker pull ghcr.io/simplylimitless/homelab-grafana:latest
docker run -d --name grafana -p 3000:3000 \
  -v "$(pwd)/config/grafana.ini:/etc/grafana/grafana.ini:ro" \
  -v grafana-data:/var/lib/grafana \
  ghcr.io/simplylimitless/homelab-grafana:latest
```

### Or build locally

```bash
docker build -t homelab-grafana .
docker run -d --name grafana -p 3000:3000 \
  -v "$(pwd)/config/grafana.ini:/etc/grafana/grafana.ini:ro" \
  -v grafana-data:/var/lib/grafana \
  homelab-grafana
```

> **Note:** After setting up the `GHCR_PAT` secret, re-run the workflow from the Actions tab or push a test commit if you don't see the image published yet.

### CI/CD

Pushing to `main` triggers an automatic build (see [`.github/workflows/docker.yml`](.github/workflows/docker.yml)). Each push is tagged with both `latest` and a numeric build ID for rollback:

```bash
docker pull ghcr.io/simplylimitless/homelab-grafana:3
```

**GHCR write permission:** The workflow uses a PAT stored in the repository secret `GHCR_PAT` to push to GitHub Container Registry. Create one at https://github.com/settings/tokens (classic tokens, not fine-grained — that scope isn't available there) with the **`packages`** scope ticked, then add it as a repo secret named `GHCR_PAT`. The automatic `GITHUB_TOKEN` doesn't grant GHCR package write permissions, so this PAT is required.
