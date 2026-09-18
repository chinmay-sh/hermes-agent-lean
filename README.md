# hermes-agent-lean

A slimmer Docker image of [Hermes Agent](https://github.com/NousResearch/hermes-agent), rebuilt automatically for every upstream release.

## What's different from the official image

[`lean.patch`](lean.patch) is applied to the upstream `Dockerfile`:

- **Smaller apt layer.** The build toolchain (`gcc`, `g++`, `make`, `cmake`, `python3-dev`, `libffi-dev`, `libolm-dev`), `docker-cli` and `iputils-ping` are dropped.
- **No Playwright/Chromium.** Browser tools are unavailable.
- **No Photon iMessage sidecar.**
- **npm and `node_modules` removed** after the web UI and TUI are built, in the same layer, so they never ship. Node itself stays so the TUI can run.
- **Fewer Python extras.** Only `mcp` and `web` are installed, plus `python-telegram-bot[webhooks]` for Telegram.

Everything else (s6 supervision, entrypoint, data volume at `/opt/data`, the dashboard) is unchanged from upstream.

## Tags

| Tag | Meaning |
|---|---|
| `vYYYY.M.D` | Built from that upstream release tag |
| `latest` | Most recent release built |

Only `linux/amd64` is built.

## Usage

Swap the image in your compose file:

```yaml
image: <dockerhub-user>/hermes-agent-lean:v2026.9.14
```

To use the web dashboard, set these environment variables:

```env
HERMES_DASHBOARD=1
HERMES_DASHBOARD_BASIC_AUTH_USERNAME=...
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=...
```

The dashboard listens on port `9119`. It won't start on a non-loopback address without an auth provider.

To update, bump the tag and run:

```bash
docker compose pull && docker compose up -d
```

## How the build works

[`.github/workflows/build.yml`](.github/workflows/build.yml) runs daily at 06:00 UTC. It can also be started by hand from the Actions tab, with an optional tag.

1. Find the latest upstream release (or use the tag given).
2. Skip if that tag is already on Docker Hub.
3. Check out upstream at that tag, apply `lean.patch`, then build and push `:<tag>` and `:latest`.

### Setup

- Repo **secret** `DOCKERHUB_TOKEN`: a Docker Hub access token with Read & Write access.
- Repo **variable** `DOCKERHUB_USERNAME`: your Docker Hub username.

## When the patch stops applying

If upstream changes its `Dockerfile`, the build fails and GitHub emails you. To fix it:

```bash
cd hermes-agent
git fetch --tags && git checkout <new-tag>
git apply ../hermes-agent-lean/lean.patch --3way   # resolve conflicts
git diff Dockerfile > ../hermes-agent-lean/lean.patch
```

Commit the new patch, then re-run the workflow.

On Windows, run these in **Git Bash**, not PowerShell. PowerShell's `>` changes the file's encoding and `git apply` rejects it. The patch must also use LF line endings; `.gitattributes` enforces this.
