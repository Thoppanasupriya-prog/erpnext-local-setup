# Custom apps

This folder holds this project's own Frappe apps. Everything else in the
repository is the upstream [frappe_docker](https://github.com/frappe/frappe_docker)
setup.

| App | What it does |
|---|---|
| `my_erp` | Navy/indigo Desk theme (light and dark), loaded through `app_include_css`. Home dashboard and form layouts are planned |

## Development

The dev bench uses this folder directly. `development/frappe-bench/apps/my_erp`
is a relative symlink to `../../../custom_apps/my_erp`. It resolves both in WSL
and inside the dev container, where the repository is mounted at `/workspace`.

Edit files in `custom_apps/my_erp`. For CSS changes, reload the browser. For
Python or `hooks.py` changes, restart `bench start`.

On a fresh clone, recreate the link after the bench exists:

```bash
cd development/frappe-bench/apps
ln -s ../../../custom_apps/my_erp my_erp
../env/bin/pip install -e my_erp
cd .. && grep -qx my_erp sites/apps.txt || echo my_erp >> sites/apps.txt
bench --site development.localhost install-app my_erp
```

## Production image

The upstream `apps.json` method can only install apps that `git clone` can fetch
from their own repository. `my_erp` lives in a subfolder of this repository, so it
is added by a small image layered on the official ERPNext image instead:
[`images/my_erp/Containerfile`](../images/my_erp/Containerfile).

Build it from the repository root. The build context is `custom_apps`:

```bash
docker build -f images/my_erp/Containerfile \
  --build-arg ERPNEXT_VERSION=v16.54.0 \
  -t my-erpnext:v16.54.0 custom_apps
```

Use the same ERPNext version as the dev bench, because the theme targets that
version's CSS. Check it with `bench version` in the dev container.

## Deploying with compose.yaml

1. Build the image on the server (above).
2. In `.env`, next to the values from `example.env`, set:

   ```env
   ERPNEXT_VERSION=v16.54.0
   CUSTOM_IMAGE=my-erpnext
   CUSTOM_TAG=v16.54.0
   PULL_POLICY=never
   ```

   `PULL_POLICY=never` stops Compose from looking for `my-erpnext` on Docker Hub.
   If you push the image to a registry instead, use the full image name and leave
   `PULL_POLICY` unset.
3. Start the stack: `docker compose -f compose.yaml <overrides...> up -d`.
4. **For an existing `sites` volume only:** the volume keeps its old `apps.txt`, so
   add `my_erp` to it once:

   ```bash
   docker compose exec backend bash -c \
     "sed -i -e '\$a\\' sites/apps.txt; grep -qx my_erp sites/apps.txt || echo my_erp >> sites/apps.txt"
   ```

5. Install the app on the site and migrate:

   ```bash
   docker compose exec backend bench --site <your-site> install-app my_erp
   docker compose exec backend bench --site <your-site> migrate
   ```

Take a backup before step 5 when moving a site to a newer ERPNext version
(`bench --site <your-site> backup --with-files`). Migrating changes the database
and cannot be undone without that backup.
