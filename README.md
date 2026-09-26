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

## The `-kapso` variant

Every tag above is also built in a second flavor with the [Kapso WhatsApp Hermes plugin](https://github.com/gokapso/hermes-agent-plugin) baked in, tagged `<tag>-kapso` / `latest-kapso`. [`kapso.patch`](kapso.patch) applies on top of `lean.patch` and:

- Drops the plugin's `plugin.yaml` / `__init__.py` / `adapter.py` into `plugins/platforms/kapso/`, the same in-tree shape as the bundled `telegram`/`whatsapp`/etc. adapters. Hermes discovers it as a **bundled** platform, so it lazy-loads with no `hermes plugins install` step and no `config.yaml` edit — it activates purely from the `KAPSO_*` env vars below.
- Bakes `aiohttp` (the plugin's only dependency) into the image at the same pinned version already used elsewhere in upstream's `pyproject.toml`, since the lean image's `.venv` is read-only at runtime and can't take a lazy `pip install`.

Use the `-kapso` tag in `.env`'s `HERMES_IMAGE` and add these to `.env` (see [Kapso's docs](https://docs.kapso.ai) for values):

```bash
KAPSO_API_KEY=kapso_...
KAPSO_WEBHOOK_SECRET=shared_webhook_secret
KAPSO_PHONE_NUMBER_ID=1041695002363992
KAPSO_HOME_CHANNEL=15551234567
KAPSO_ALLOWED_USERS=15551234567
```

Kapso needs a public HTTPS webhook at `https://<your-hostname>/kapso/webhook`, so in Cloudflare Zero Trust add a second public hostname (or a path rule on the existing one) that also routes to `http://hermes:8648` — the plugin's webhook port, separate from the dashboard's `9119`. Create the phone-number webhook in the Kapso dashboard pointing at that URL (kind `kapso`, event `whatsapp.message.received`, payload version `v2`, secret matching `KAPSO_WEBHOOK_SECRET`).

`hermes kapso setup --install-cli` (the guided setup command) needs Node/npm to install the Kapso CLI, which this image removes at build time — run it on your host with `npx @kapso/cli` instead, or configure everything by hand with the env vars and manual webhook above.

## Tags

| Tag | Meaning |
|---|---|
| `vYYYY.M.D` | Built from that upstream release tag |
| `latest` | Most recent release built |
| `vYYYY.M.D-kapso` | Same release, with the Kapso plugin baked in |
| `latest-kapso` | Most recent release built, with the Kapso plugin baked in |

Only `linux/amd64` is built.

## Usage

[`docker-compose.yml`](docker-compose.yml) runs the Hermes gateway with the web dashboard behind a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/). It publishes no host ports.

1. In Cloudflare Zero Trust, create a tunnel and copy its token. Add a public hostname (e.g. `hermes.example.com`) that routes to `http://hermes:9119`.
2. Copy [`.env.example`](.env.example) to `.env` and fill it in.
3. Start it:

   ```bash
   docker compose up -d
   ```

Agent state lives in `./data`. The dashboard won't start on a non-loopback address without an auth provider; the basic-auth variables in `.env` provide one.

To update, bump the tag and run:

```bash
docker compose pull && docker compose up -d
```

## How the build works

[`.github/workflows/build.yml`](.github/workflows/build.yml) runs daily at 06:00 UTC. It can also be started by hand from the Actions tab, with an optional tag.

1. Find the latest upstream release (or use the tag given).
2. Skip if that tag is already on Docker Hub.
3. Check out upstream at that tag, apply `lean.patch`, then build and push `:<tag>` and `:latest`.
4. In parallel, repeat the same build for the `-kapso` variant: also fetch the [Kapso plugin](https://github.com/gokapso/hermes-agent-plugin) (latest tag, or the ref given), copy it into `plugins/platforms/kapso/`, apply `kapso.patch` on top of `lean.patch`, then build and push `:<tag>-kapso` and `:latest-kapso`.

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

`kapso.patch` applies on top of the result, so regenerate it the same way once `lean.patch` applies again:

```bash
git apply ../hermes-agent-lean/kapso.patch --3way   # resolve conflicts
git diff Dockerfile > ../hermes-agent-lean/kapso.patch
```
