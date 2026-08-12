# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

The **demo/reference shop** for [sissy](https://github.com/vanWittlaer/sissy) — Shopware 6 on a single cloud server with nothing but Docker Compose, no Coolify, no Terraform, no PaaS. This repo is the *application*; sissy is the *platform*, and this is its first consumer, so the consumption path gets exercised here before anyone else uses it.

| Path        | Role |
|-------------|------|
| `shopware/` | The Shopware 6.7 application (stock Flex skeleton + `swag/demo-data`). This is what gets baked into the container image. |
| `.ddev/`    | Local development environment (DDEV, project name `sissy`, `https://sissy.ddev.site`). |

Repo renamed `sissy` → `sissy-demo` on 2026-08-12, when the platform took the `sissy` name. See [Deployment](#deployment) for what that means for the image path.

## Local development (DDEV)

```bash
ddev start                       # https://sissy.ddev.site
ddev ssh                         # lands in /var/www/html/shopware (working_dir)
ddev exec bin/console <cmd>      # Shopware/Symfony console
ddev composer <cmd>              # composer_root is shopware/
ddev launch -m                   # Mailpit
ddev claude                      # Claude Code inside the web container
```

`shopware/.env.local` (dev) overrides `shopware/.env` (which still carries the stock `prod`/localhost defaults — ignore it locally). DDEV runs PHP 8.4 / MariaDB 11.8 / Node 24, nginx-fpm, docroot `shopware/public`.

Asset builds use the stock Shopware scripts from `shopware/`: `bin/build-administration.sh`, `bin/build-storefront.sh`, `bin/watch-administration.sh`, `bin/watch-storefront.sh`.

There is **no test, lint, or static-analysis tooling in this project** — no PHPUnit, PHPStan, ECS, or Makefile, and `custom/plugins` is empty. Don't invent commands for them; if a plugin is added, its own tooling comes with it.

Two DDEV config fragments are project-specific and worth knowing before touching `.ddev/`:
- `config.claude.yaml` — symlinks `~/.claude*` into DDEV's global cache volume so Claude auth survives `ddev restart`/`rebuild`/`delete`, keyed per project.
- `config.git-signing.yaml` — SSH commit signing inside the web container. **Inert until `GIT_SIGNING_KEY` is set**; personal values belong in a gitignored `config.git-signing.local.yaml`. The private key is never stored — forward it with `ddev auth ssh -f <key>` (needed again after every `ddev poweroff` or host reboot).

## Deployment

The server side lives in a **separate repo**: [vanWittlaer/sissy](https://github.com/vanWittlaer/sissy) — compose stacks, `bootstrap.sh`/`deploy.sh`/`backup.sh`, cloud-init, and the reusable workflows. It was `sissy/` in this repo until 2026-08-12, split out with `git subtree split`, so the history moved with it. **Don't re-add server topology here** — that's the other repo, and hosts clone it directly to `/opt/sissy`.

What stays this repo's job:

- **`shopware/docker/Dockerfile`** — the image build. Two stages: `ghcr.io/shopware/shopware-cli` runs `shopware-cli project ci /src`, and the result is copied `--chown=82` into `ghcr.io/shopware/docker-base:8.4`.
- **`shopware/deployment-helper`** is a direct requirement in `composer.json` on purpose — it reached the project only via `shopware/docker`, and the whole deploy path depends on it.
- **`.github/workflows/ci-cd.yml`** — policy only: `develop`/`feature/**` build for `stage`, `main` builds for `prod`, and only `develop`/`main` deploy. The build and deploy jobs it calls are `vanWittlaer/sissy/...@v1`, so bumping the platform is a tag bump here.

The image name is derived in CI from `github.repository`. This repo was renamed `sissy` → `sissy-demo` on 2026-08-12, so builds now push to `ghcr.io/vanwittlaer/sissy-demo`; a host still holding `APP_IMAGE=ghcr.io/vanwittlaer/sissy` in its stack `.env` will pull happily and never see a new build.
