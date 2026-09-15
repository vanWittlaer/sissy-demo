# sissy-demo

Demo and reference shop for [sissy](https://github.com/vanWittlaer/sissy): Shopware 6 on a single cloud server with nothing but Docker Compose.

This repo is the *application*. The server side (compose stacks, bootstrap/deploy/backup scripts, reusable workflows) lives in the sissy platform repo.

| Path        | Role |
|-------------|------|
| `shopware/` | Shopware 6.7 (stock Flex skeleton + `swag/demo-data`), baked into the container image |
| `.ddev/`    | Local development environment |

## Local development

Requires [DDEV](https://ddev.com).

```bash
ddev start                       # https://sissy.ddev.site
ddev ssh                         # lands in /var/www/html/shopware
ddev exec bin/console <cmd>      # Shopware console
ddev composer <cmd>              # composer root is shopware/
ddev launch -m                   # Mailpit
```

Asset builds and watchers go through [shopware-cli](https://sw-cli.fos.gg), installed in the web container by the [ddev-shopware-cli](https://github.com/vanwittlaer/ddev-shopware-cli) add-on:

```bash
ddev shopware-cli project admin-build       # build administration
ddev shopware-cli project storefront-build  # build storefront
ddev storefront-watch                       # storefront hot-reload
ddev admin-watch custom/static-plugins/<Plugin>  # admin (Vite) hot-reload for a plugin
```

Optional DDEV extras: `.ddev/config.git-signing.yaml` enables SSH commit signing inside the container once `GIT_SIGNING_KEY` is set in a gitignored `config.git-signing.local.yaml`. Forward the key with `ddev auth ssh -f <key>`.

## Build and deploy

- `shopware/docker/Dockerfile` builds the image: `shopware-cli project ci` on top of `ghcr.io/shopware/docker-base:8.4`.
- `.github/workflows/ci-cd.yml` holds only the policy. The build and deploy jobs come from `vanWittlaer/sissy@v1`.

| Ref                     | Builds | Deploys |
|-------------------------|--------|---------|
| `feature/**`            | stage  | no      |
| `develop`               | stage  | stage   |
| `main`                  | prod   | no      |
| `v*` tag                | prod   | prod    |

Images are pushed to `ghcr.io/vanwittlaer/sissy-demo`.
