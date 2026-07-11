# grafana

Grafana instance for monitoring dashboards. Docker-based, scraping from various data sources and exposing dashboards via `localhost:3000`.

Published images are multi-arch (`linux/amd64`, `linux/arm64`, `linux/arm/v7`), so they run unmodified on x86 servers, 64-bit ARM boards (e.g. Raspberry Pi 4/5, 64-bit OS), and older 32-bit ARMv7 boards.

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
  ghcr.io/simplylimitless/grafana:latest
```

| Flag | Maps to | Example |
|---|---|---|
| `-p` or `--publish` | Port mapping (host:container) | `-p 3001:3000` for a non-default host port |
| `-v` or `--volume` | Bind mount or named volume | See table above |

### Deploy from GitHub Container Registry

```bash
docker login ghcr.io -u YOUR_GITHUB_USERNAME   # password: a GitHub PAT with `packages` scope (create one at https://github.com/settings/tokens, not the fine-grained type)
docker pull ghcr.io/simplylimitless/grafana:latest
docker run -d --name grafana -p 3000:3000 \
  -v "$(pwd)/config/grafana.ini:/etc/grafana/grafana.ini:ro" \
  -v grafana-data:/var/lib/grafana \
  ghcr.io/simplylimitless/grafana:latest
```

### Or build locally

```bash
docker build -t grafana .
docker run -d --name grafana -p 3000:3000 \
  -v "$(pwd)/config/grafana.ini:/etc/grafana/grafana.ini:ro" \
  -v grafana-data:/var/lib/grafana \
  grafana
```

This produces a single-arch image for your local machine. To build and load a multi-arch image (e.g. to test an ARM target from an x86 host), use `docker buildx` with QEMU emulation:

```bash
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t grafana .
```

Note that `--load` only accepts one platform at a time; multi-platform builds must be pushed to a registry (`--push`) or built one platform at a time for local use.

> **Note:** After setting up the `GHCR_PAT` secret, re-run the workflow from the Actions tab or push a test commit if you don't see the image published yet.

### CI/CD

Pushing to `main` triggers an automatic multi-arch build for `linux/amd64`, `linux/arm64`, and `linux/arm/v7` (see [`.github/workflows/docker.yml`](.github/workflows/docker.yml)), using QEMU + Buildx to cross-build the non-native architectures. Each push is tagged with both `latest` and a numeric build ID for rollback:

```bash
docker pull ghcr.io/simplylimitless/grafana:3
```

**GHCR write permission:** The workflow uses a PAT stored in the repository secret `GHCR_PAT` to push to GitHub Container Registry. Create one at https://github.com/settings/tokens (classic tokens, not fine-grained — that scope isn't available there) with the **`packages`** scope ticked, then add it as a repo secret named `GHCR_PAT`. The automatic `GITHUB_TOKEN` doesn't grant GHCR package write permissions, so this PAT is required.
