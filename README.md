# TruMo

TruMo is a web application for recording, viewing, editing, and comparing movement using camera-based pose detection and animated 3D models. The frontend combines React, Three.js, and TensorFlow.js; the backend uses Elysia on Bun with MongoDB for persistence.

This parent repository manages the application and shared packages as Git submodules and Bun workspaces.

## Repository structure

| Path | Purpose |
| --- | --- |
| [`apps/frontend`](https://github.com/Skizy/instructions-frontend) | React application, pose detection, recording, and Three.js tools. |
| [`apps/backend`](https://github.com/Skizy/instructions-backend) | Authentication, models, recordings, instructions, drafts, camera connections, and time synchronization. |
| [`apps/html_server`](https://github.com/Skizy/instructions-html-server) | Separate Bun server for frontend files and API/WebSocket forwarding. |
| [`packages/custom-types`](https://github.com/Skizy/instructions-custom-types) | Shared camera and socket message types and runtime schemas. |
| [`packages/instruction-viewer`](https://github.com/Skizy/instruction-viewer) | Three.js instruction viewer. |

The frontend development server forwards `/api` requests to the backend and removes the `/api` prefix. MongoDB stores application records; the backend also stores uploaded files under `apps/data` by default.

## Requirements

- Bun and Git.
- A running MongoDB instance, available at `mongodb://localhost:27017` by default.
- A browser with WebGL and camera access for pose detection.
- Access to the submodule repositories.

Camera access requires a secure browser context, such as localhost or HTTPS. Devices connecting over a network need an HTTPS setup trusted by their browsers.

## Clone and install

Clone the parent repository using its GitHub URL:

```sh
git clone --recurse-submodules <parent-repository-url> instructions_monorepo
cd instructions_monorepo
bun install
```

For an existing clone, initialize the pinned submodules before installing:

```sh
git submodule update --init --recursive
bun install
```

Run dependency installation from the parent directory so Bun can link the workspace packages.

## Local configuration

Create the local configuration and data directories:

```sh
mkdir -p apps/config apps/data
```

Optional `apps/config/server.json` configures the backend:

```json
{
  "port": 3005,
  "mongourl": "mongodb://localhost:27017"
}
```

The backend defaults to those values and uses `apps/data` for files. Set `datapath` to an absolute directory in `server.json` to use another location.

Optional `apps/config/client.json` configures the frontend development server:

```json
{
  "port": 3000,
  "backendUrl": "http://localhost:3005"
}
```

Both applications fall back to defaults if their configuration file cannot be read. `CONFIG_PATH` selects an alternate configuration file; relative paths are resolved from the process working directory.

| Variable | Used by | Purpose |
| --- | --- | --- |
| `CONFIG_PATH` | Backend, frontend dev server, HTML server | Path to the application's JSON configuration. |
| `SETUP3D_TLS_CERT` | Frontend dev server, HTML server | TLS certificate file. |
| `SETUP3D_TLS_KEY` | Frontend dev server, HTML server | TLS private key file. |
| `STATIC_DIR` | HTML server | Frontend build directory. |
| `ASSETS_DIR` | HTML server | Static asset directory. |
| `HOTRESTART` | HTML server | Enables its browser reload support when set to a nonempty value. |

The frontend's `dev` script sets the TLS paths to `apps/config/fullchain.pem` and `apps/config/privkey.pem`. The current TLS configuration also hardcodes a deployment hostname in `apps/frontend/rsbuild.config.ts` and `apps/html_server/src/index.ts`; adapt it when configuring HTTPS for another host.

`apps/config` and `apps/data` are ignored by the parent repository. Keep deployment credentials, certificates, and application data in those local directories rather than source files.

## Run development servers

Start MongoDB using your local service manager or installation. Then run the backend in one terminal:

```sh
cd apps/backend
bun run dev
```

Run the frontend in another terminal:

```sh
cd apps/frontend
bun run dev
```

The default backend port is `3005`; the frontend port is `3000`. Open the URL printed by the frontend server. The HTML server is not required for this development workflow.

The backend's `start.sh` contains machine-specific paths. Use `bun run dev` directly for a portable setup.

## Models and assets

Frontend `public` assets and backend application data are not included in the tracked source. A fresh clone therefore needs additional assets for the full application to work. Paths used by the code include:

- `apps/frontend/public/favicon.svg`, referenced by the frontend build configuration.
- `apps/frontend/public/mixamo2.glb`, used as a default character model.
- `apps/frontend/public/ml_models/detector/model.json` and `ml_models/heavy/model.json`, plus their referenced weight files, used by pose detection.
- `apps/data/rigModels/default.glb`, used by the backend when a user has not selected a character model.
- `apps/data/rooms/room.json`, used for room data.

Other views may require additional textures or assets from `public`. Obtain the matching assets for your deployment; installing JavaScript dependencies does not populate them.

## Build and check

Build the frontend from its directory:

```sh
cd apps/frontend
bun run build
```

This runs TypeScript checking followed by Rsbuild. The backend runs TypeScript directly through Bun:

```sh
cd apps/backend
bun run server
```

To check an individual workspace without emitting files, run this from its directory after installing dependencies:

```sh
bun x --no-install tsc --noEmit
```

There is no parent-level build or test script. The backend's `test` script is a placeholder. The latest local check found an existing backend type error in `src/handlers/instructions.ts`; the frontend, HTML server, custom-types, and instruction-viewer checks passed.

### Standalone HTML server

`apps/html_server` provides static file serving and forwards API and camera WebSocket traffic to the backend. Its `dev` script supplies `STATIC_DIR=../frontend/out`, `ASSETS_DIR=../frontend/public`, and `CONFIG_PATH=../config/client.json`.

Its routes currently expect an older per-page build layout, including files such as `home/index.html` and `viewer/main.js`. The frontend's current Rsbuild configuration does not define that layout. Align the server routes with the generated frontend output before using it to serve a build.

## Working with submodules

Each submodule has its own Git history. Commit changes inside the affected repository, then update its pointer in the parent:

```sh
git -C apps/frontend add <changed-files>
git -C apps/frontend commit -m "Describe the frontend change"
git add apps/frontend
git commit -m "Update frontend submodule"
```

Push the submodule commit to its GitHub repository before pushing the parent commit that references it. The configured GitHub remote in the existing local repositories is named `github`; fresh submodule clones normally use `origin`. Check `git remote -v` inside the submodule to choose the correct remote.

After pulling updates to the parent, check out its pinned submodule commits:

```sh
git submodule update --init --recursive
git submodule status --recursive
```

`packages/libs` is no longer tracked by the parent and is not required by the current frontend/backend imports. Local directories under `packages/*` still match the Bun workspace pattern, so keep optional local packages out of a clean clone when checking reproducibility.
