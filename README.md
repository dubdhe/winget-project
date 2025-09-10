# 🧰 WinGet Manifests CI/CD Workflow

This repository implements a GitHub Actions pipeline that:

✅ Validates changed WinGet manifests against official schemas via `yaml-ls-check`.  
📦 Extracts installer metadata (URL, hash, arch) into a matrix.  
📥 Downloads installers to `temp_download/<Package>/<Version>/<Architecture>/...` on a self-hosted runner.  
📤 Uploads files to your Windows-based FTPS server under `New Software/…`.  
🔁 Polls the FTPS server for scan results under:

- `Scanned software passed/...`
- `Scanned software failed/...`

## 🧑‍💻 Usage

### 1. Add manifests

Place your full multi-file manifests under `manifests/`, following official WinGet layout:

- `version.yaml`
- `installer.yaml`
- `locale.yaml` / `defaultLocale.yaml`

### 2. Push changes

Create a **new branch** (e.g., `keepassXC`) for each app or manifest group.  
Push your changes. A PR can be opened once all jobs succeed.

## ✅ Prerequisites

### 🖥 Runner

A **self-hosted GitHub Actions runner**:

- Architecture: `Linux ARM64`
- Runner tag: `winget`

### 🧰 Installed Tools

Make sure the runner has:

- `curl`, `git`, `jq`, `lftp`
- [`yq`](https://github.com/mikefarah/yq) v4.x (installed manually)
- `Node.js` (for `yaml-ls-check` GitHub Action)

### 🔐 FTPS Access

Add the following secrets under **Settings > Secrets and variables > Actions**:

| Secret Name | Description                 |
| ----------- | --------------------------- |
| `FTP_USER`  | FTPS username               |
| `FTP_PASS`  | FTPS password               |
| `FTP_HOST`  | FTPS hostname (no `ftp://`) |
| `FTP_BASE`  | e.g. `New Software`         |

## 📂 Directory Structure

```
├── .github/workflows/
│   └── winet-workflow.yml         # CI workflow
├── .vscode/
│   └── settings.json              # Local schema mapping
├── manifests/                     # Your WinGet manifests
│   └── <Vendor>/<App>/<Version>/...
├── schemas/                       # Local JSON schemas (optional)
│   ├── installer.schema.json
│   ├── defaultLocale.schema.json
│   └── version.schema.json
└── README.md
```

## ⚙️ `.vscode/settings.json`

This maps local schemas to your manifest types:

```json
{
  "yaml.schemas": {
    "./schemas/installer.schema.json": "manifests/**/*.installer.yaml",
    "./schemas/defaultLocale.schema.json": "manifests/**/*.locale*.yaml",
    "./schemas/version.schema.json": [
      "manifests/**/*.yaml",
      "!manifests/**/*.locale*.yaml",
      "!manifests/**/*.installer.yaml"
    ]
  }
}
```

If you prefer to use **online schemas**, remove `.vscode/settings.json`  
and re-add the following comment to the top of each manifest:

```yaml
# yaml-language-server: $schema=https://aka.ms/winget-manifest.installer.1.6.0.schema.json
```

> 🧪 Online schemas may fail on self-hosted runners due to missing certificates.

## 🚀 Workflow Breakdown

### ✅ `validate` job

- Removes `# yaml-language-server: $schema=...` lines from all YAML files.
- Runs [`InoUno/yaml-ls-check`](https://github.com/InoUno/yaml-ls-check) against local schemas in `.vscode/settings.json`.

### 🔍 `extract-installers` job

- Scans newly added `.installer.yaml` files only.
- Builds a matrix of:
  ```json
  { "id": "...", "version": "...", "arch": "...", "url": "...", "hash": "..." }
  ```

### 📥 `process-installers` job (matrix)

For each installer:

1. **Download** the file with `curl`.
2. **Verify** SHA256 hash.
3. **Upload** to:
   ```
   /New Software/<PackageIdentifier>/<Version>/<Architecture>/
   ```
   via `lftp`.
4. **Poll** the server up to 3 times for scan results in:

   - `/Scanned software passed/...`
   - `/Scanned software failed/...`

   Starts polling **after waiting** for `SLEEP_INTERVAL_MINUTES`.

## 🛠️ Known Bugs & Fixes

| Issue / Error                                | Fix / Explanation                                                   |
| -------------------------------------------- | ------------------------------------------------------------------- |
| ❌ `ReleaseDate` fails validation            | ✅ Wrap it in quotes: `"2024-11-30"`                                |
| ❌ `AppsAndFeaturesEntries` / `Dependencies` | ✅ Use official JSON schemas via `yaml-ls-check`                    |
| ❌ Matrix is empty in process-installers     | ✅ Ensure new `.installer.yaml` files exist in the commit           |
| ❌ Yamale schema mismatch or failure         | ✅ Fully replaced by `yaml-ls-check`, no need to maintain `.yamale` |
| ❌ lftp mkdir/cd fails                       | ✅ Use raw folder names (e.g., `New Software`) without quotes       |
| ❌ Polling exits too early                   | ✅ Initial wait before polling + error handling improved            |

## 💡 Tips & Best Practices

- 💡 Push each app's manifest set in a dedicated branch.
- 💡 Always include all 3 files: `version`, `installer`, and at least one `locale`.
- 💡 Avoid manually URL-encoding the FTP paths — just use `New Software` literally.
- 💡 Use `curl -v` or `lftp` locally to debug FTP server issues.
- 💡 Check `matrix.installer` carefully to avoid empty runs.

## 🧪 Validation Options

| Method                    | Behavior                                                            |
| ------------------------- | ------------------------------------------------------------------- |
| `.vscode/settings.json`   | ✅ Uses local JSON schemas (recommended for CI)                     |
| `$schema` comment in YAML | ❌ Tries to fetch online schemas (can fail without certs on runner) |

For schema upgrades, improved polling logic, or deeper CI integration,  
edit the workflow in `.github/workflows/` and `.vscode/settings.json`.
