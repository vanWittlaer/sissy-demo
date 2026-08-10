# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **demo/reference setup for running Shopware 6 on a single cloud server with nothing but Docker Compose** — no Coolify, no Terraform, no PaaS. Two halves that are deliberately independent:

| Path        | Role |
|-------------|------|
| `shopware/` | The Shopware 6.7 application (stock Flex skeleton + `swag/demo-data`). This is what gets baked into the container image. |
| `sissy/`   | **S**imple **Shopware** **S**erver — the server-side topology: compose stacks, bootstrap/deploy scripts, cloud-init. This is the actual subject of the project. |
| `.ddev/`    | Local development environment (DDEV, project name `sissy`, `https://sissy.ddev.site`). |

`sissy/` was named `shop-server/` until 2026-08-08, and the deploy target on the host moved from `/opt/shop-server` to `/opt/sissy` at the same time. An existing server provisioned before that still has the old path — move the directory there (or re-run cloud-init) before deploying, and update the CI `ssh` line with it.

Not a git repository at the time of writing.

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

## Production topology (`sissy/`)

Three compose stacks on one VPS, joined by an external Docker network named `edge`:

```
edge/      Traefik v3.3 — owns :80/:443, ACME TLS, docker provider (exposedbydefault=false)
prod/      web + worker×2 + scheduler + mariadb
stage/     web + worker + scheduler + mariadb   (Traefik basic-auth middleware)
```

The defining architectural choice: **MariaDB is the message queue and the lock store**. `MESSENGER_TRANSPORT_DSN` is left unset so Symfony falls back to `doctrine://`, `LOCK_DSN` points at the DB, and cache/sessions are on the local filesystem. No Redis, RabbitMQ, OpenSearch, or S3 anywhere in the server stacks — even though `shopware/composer.json` still requires `shopware/elasticsearch` (`SHOPWARE_ES_ENABLED=0` disables it).

Consequences that constrain any change you make here:
- **`web` must stay at 1 replica.** Filesystem cache + native PHP sessions are per-container. Workers scale freely. A second web node requires introducing Redis first.
- **Workers must consume both `async` and `low_priority`** — separate Doctrine queues; dropping `low_priority` silently strands those messages.
- **Theme compilation happens inside the live `web` container** (via `shopware-deployment-helper`), because there is no S3 to hold the compiled theme. This is why deploys are not zero-downtime.
- `scheduler` runs `scheduled-task:run` as a blocking long-running process — no `--no-wait`.

### The two-env-file rule (the #1 compose gotcha here)

Every stack dir has **both**, and they are not interchangeable:

- `.env` — sits next to `compose.yaml` and is **interpolated into it** (`TAG`, `DOMAIN`, `DB_PASSWORD`, `BASIC_AUTH`).
- `.env.local` — named by `env_file:`, so its lines are injected as **environment variables** in the app containers. Values are **literal**; `${...}` is *not* expanded, so `DATABASE_URL`/`LOCK_DSN` must spell the password out in full.

Neither file is mounted — both are read host-side by the Compose CLI. Since `env_file` is applied at container creation, a changed `.env.local` needs `docker compose up -d` to take effect; `restart` reuses the old values.

Change a DB password and you must edit it in both files, in every DSN.

Stage's `BASIC_AUTH` htpasswd hash goes through compose interpolation, so **every `$` must be doubled to `$$`** (`htpasswd -nb user pass`, then double). Forgetting this silently breaks auth.

`.gitignore` in `sissy/` blocks `**/.env`, `**/.env.local`, and `**/data/` — only the `*.example` files are ever committed.

### Server operations

```bash
./bootstrap.sh                   # edge + prod + stage; idempotent; default stacks: prod stage
./bootstrap.sh stage             # one stack only
./deploy.sh prod 1a2b3c4         # pull tag, recreate services, migrate + theme-compile
./deploy.sh stage latest
./refresh-stage.sh               # prod DB -> stage DB (routed via host), then deployment helper
```

All three `cd` to their own directory first, so they work from anywhere.

Install and migration both run in a dedicated **`setup` container** (`entrypoint: /setup`, the deployment helper), which the app services gate on via `depends_on: {setup: {condition: service_completed_successfully}}`. So `up -d` *is* the deploy — `bootstrap.sh` and `deploy.sh` no longer exec the helper themselves. The ordering matters: migrations land while the old containers still serve, then new ones start, rather than new code meeting an old schema. `setup` overrides the `x-app` anchor's `depends_on` so it doesn't depend on itself, and `restart: "no"` so a completed run isn't restarted.

`shopware/deployment-helper` is a direct requirement in `composer.json` on purpose — it reached the project only via `shopware/docker`, and the whole deploy path depends on it.

`refresh-stage.sh` pipes `mariadb-dump` through the *host* because prod's and stage's `internal` networks are isolated from each other. It does not copy media — `rsync prod/data/media stage/data/media` separately if needed.

### Reverse proxy / TLS

Traefik terminates TLS and forwards **plain HTTP** to the container on `:8000`. Shopware's public filesystem has no `url` set, so asset URLs are built from the incoming request — which reads as `http://` unless the proxy headers are trusted, producing blocked mixed-content on every `/bundles/...` asset.

The fix is one variable in each stack's `.env.local`: **`SYMFONY_TRUSTED_PROXIES=0.0.0.0/0`**. Note the name — Symfony's FrameworkBundle defaults `trusted_proxies` to `%env(default::SYMFONY_TRUSTED_PROXIES)%`, so a plain `TRUSTED_PROXIES` is read by nothing and no config file is needed. `SYMFONY_TRUSTED_HEADERS` can stay unset; `Kernel.php` then trusts `FOR|PORT|PROTO`, which covers the scheme. It's an env var, so `up -d` suffices — no rebuild.

`0.0.0.0/0` is safe only because no service publishes a port: Traefik is the sole route in, and it strips inbound `X-Forwarded-*` from untrusted clients. Publish an app port and this becomes a live spoofing vector.

To check it's actually applied (`debug:config` fails on a frozen prod container):

```bash
docker compose -f prod/compose.yaml exec -T web php bin/console debug:container --parameter=kernel.trusted_proxies
curl -s https://<domain>/admin | grep -o 'src="[^"]*"' | head   # server-side truth, no browser cache
```

### Firewall

**There is deliberately no host firewall — don't add one back.** ufw/iptables filter the `INPUT` chain, but dockerd DNATs published ports in `nat/PREROUTING` and routes them through `FORWARD`, so every `ports:` mapping stays reachable regardless of the rules. A host firewall here would protect sshd only while appearing to protect everything, which is worse than none.

Inbound filtering is a **network-level Hetzner Cloud Firewall** (22/80/443 open, everything else denied), documented as `hcloud` commands in `sissy/README.md` so it stays part of the recovery record. Port 22 is open deliberately — the admin IP is dynamic — so key-only auth, no root login, and fail2ban are what actually guard SSH. On the host, the control that actually works is **not publishing ports**: both MariaDBs sit on `internal` with no `ports:`. fail2ban still runs for SSH brute-force; the `[sshd]` jail works out of the box on 24.04, so `/etc/fail2ban/jail.local` only tunes ban timings.

### SSH access

No key is committed. `cloud-init.yaml` copies the keys the provider injected for `root` (Hetzner's console selection) onto `deploy`, stripping any leading `command="..."` options cloud-init's `disable_root` may have prefixed — copying those verbatim would give `deploy` a login that prints a message and exits. It then disables password auth and root login via `/etc/ssh/sshd_config.d/99-hardening.conf`.

**The ordering is the point.** That hardening is a guarded `runcmd`, not cloud-init's `ssh_pwauth: false` / `disable_root: true`, because those fire unconditionally in an earlier stage — on a provider that injects no keys they'd produce an unreachable box. The script checks `deploy`'s `authorized_keys` is non-empty, skips with a warning otherwise, and reverts the drop-in if `sshd -t` rejects it. Preserve that check-before-lockdown shape if you touch it.

Also note `users:` starts with `- default`: declaring the list at all replaces the image default, and the provider's keys are handed to the default entry — drop it and there may be nothing to copy from.

### Volumes

`x-app-volumes` in each `compose.yaml` is shared by **web and worker**: `public/media`, `public/thumbnail`, `public/theme`, `public/sitemap`, `files` (Shopware's private filesystem — invoice PDFs, import/export, `theme-config`), and `var/log`. The sharing matters as much as the persistence — the worker generates thumbnails and sitemaps that only `web` serves, so an unshared path means the work happens and is never visible. `files` and `media` are the ones whose loss is unrecoverable; theme/thumbnail/sitemap regenerate.

Never mount `public/bundles` (baked at build by `assets:install`; a bind mount hides the image's copy, as bind mounts don't seed from the image) or share `var/cache` (per-container by design).

All of these must be **owned by UID 82** or the app fails with Permission denied. `bootstrap.sh:prep_dirs` creates and chowns them before Docker can make them root-owned — its `app_dirs` list must stay in sync with `x-app-volumes`.

### Image

`shopware/docker/Dockerfile` is a two-stage build: `ghcr.io/shopware/shopware-cli` runs `shopware-cli project ci /src`, and the result is copied `--chown=82` into `ghcr.io/shopware/docker-base:8.4`. The compose files reference `ghcr.io/vanwittlaer/sissy:${TAG:-latest}`.

**One image name for every environment** — prod and stage differ by *tag*, never by repository, so promoting a build is just deploying an existing tag. `.github/workflows/` pushes two tags per build: a moving pointer (`latest` on `main`, `stage` on `develop`) and the immutable commit SHA. Deploys always use the SHA, passed from the build job's `tag` output, so what ships is exactly what was built. The environment is also recorded as an OCI label (`io.sissy.environment`).

## Known open items

Carried over from `sissy/README.md` — treat as unfinished, not as settled design:

- **Risk W:** confirm a `worker` container actually runs `messenger:consume` and does not boot nginx — the `shopware/docker-base` entrypoint decides by args.
- CI deploy needs its own key: add the `SSH_PRIVATE_KEY` secret's public half to `/home/deploy/.ssh/authorized_keys` on the host, plus repo variables `DEPLOY_HOST` and `SSH_KNOWN_HOSTS`. cloud-init only imports the provider's console keys.

## CI/CD (`.github/workflows/`)

`ci-cd.yml` is the entry point: `develop` and `feature/**` build for `stage`, `main` builds for `prod`, and only `develop`/`main` go on to deploy. `build-docker.yml` builds `shopware/docker/Dockerfile` (context `./shopware`) and pushes; `deploy-ssh.yml` SSHes to the host and runs `/opt/sissy/deploy.sh <stack> <sha>`.

`stage` names both the GitHub environment and the stack directory, so `inputs.environment` and `inputs.stack` are the same string today — they're separate inputs because the environment carries the secrets while the stack is a path on disk.
