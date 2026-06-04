# WSL Setup (Node + pnpm) for this repo

This project uses **pnpm**. Running Windows `npm`/`node` against a WSL path can fail with errors like `UNC paths are not supported`.

## Recommended setup

Run these commands in **WSL**:

```bash
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.nvm/nvm.sh
nvm install --lts
nvm use --lts

corepack enable
corepack prepare pnpm@latest --activate
```

Verify the active binaries are Linux ones:

```bash
type -a node npm pnpm
node -v
npm -v
pnpm -v
```

Expected: primary paths should point to `~/.nvm/...`, not `C:\...`.

## Clean install for this repository

From the repo root:

```bash
rm -rf node_modules package-lock.json
pnpm install
```

If pnpm blocks dependency postinstall scripts with `ERR_PNPM_IGNORED_BUILDS`, approve once:

```bash
pnpm approve-builds --all
```

Then run install/build again:

```bash
pnpm install
pnpm build
# Optional if you also package Firefox:
# pnpm build:firefox
```

## Caveats and gotchas

- Do **not** run `npm install pnpm` inside this repo. It can trigger project install hooks with the wrong toolchain.
- If `pnpm` is not found after install, restart the shell or run `source ~/.nvm/nvm.sh` again.
- If you see `npm warn Unknown project config "public-hoist-pattern"`, you are likely invoking `npm`; prefer `pnpm` for this repo.
- If `pnpm` warns that `"pnpm" field in package.json is no longer read`, move those settings to `pnpm-workspace.yaml` / `.npmrc` when maintaining config.

## Optional hardening (avoid accidental Windows Node in WSL)

In `/etc/wsl.conf`:

```ini
[interop]
appendWindowsPath=false
```

Then restart WSL (`wsl --shutdown` from Windows terminal) and open WSL again.
