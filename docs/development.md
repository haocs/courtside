# Development setup

## Workspace

Use normal development storage on the workstation, not a NAS media folder or an editor-managed temporary runtime. A convenient layout is:

```text
courtside-workspace/
  courtside/                  documentation hub
  tennis-web/                 independent Git checkout
  tennis-processing/          independent Git checkout
  tennis-processing-server/   independent Git checkout
  tennis-algorithm-lab/        independent Git checkout
  private/                    local-only settings, synthetic fixtures and state
```

The parent is a workspace directory, not a new monorepo. Keep `private/` outside every Git checkout. Clone only the components needed for a task; worker/server integration tests may need both sibling checkouts.

```sh
mkdir -p courtside-workspace
cd courtside-workspace
git clone https://github.com/haocs/courtside.git
git clone https://github.com/haocs/tennis-web.git
git clone https://github.com/haocs/tennis-processing.git
git clone https://github.com/haocs/tennis-processing-server.git
git clone https://github.com/haocs/tennis-algorithm-lab.git
mkdir -p private/dev
export COURTSIDE_WORKSPACE="$PWD"
export COURTSIDE_PRIVATE="$PWD/private"
```

The component command blocks use `COURTSIDE_WORKSPACE` from the clone step; set it to your existing workspace when resuming. `COURTSIDE_PRIVATE` is an operator-selected local directory, not a shared or committed default. Do not use production mounts/settings to make a local test pass.

## Tools and dependency ownership

Python 3.12 is the common development baseline. Use a separate virtual environment per component and that component's pinned requirements. Website helper tests use Node; the reviewed workflow uses Node 24. Browser regression tests use the pinned Python Playwright package with Chromium and WebKit. FFmpeg is needed for actual media/thumbnail checks. Docker/Compose is needed for container smoke checks and container deployment, not for a documentation edit.

On Windows, adapt `.venv/bin/python` to `.venv/Scripts/python.exe`. Do not interpret passing Python unit tests there as validated Windows HDR export or service installation. Apple Silicon production workers need the component's native Apple encoder/toolchain prerequisites. Runtime/model dependency notices apply separately from this hub.

## Website

```sh
cd "$COURTSIDE_WORKSPACE/tennis-web"
python3.12 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q
node --test tests/frontend/*.test.cjs
.venv/bin/python -m pip install -r requirements-ui.txt
.venv/bin/python -m playwright install chromium webkit
.venv/bin/python -m pytest tests/ui_player.py -q
```

Install FFmpeg through the workstation's normal package manager for media checks; the Linux CI workflow also installs browser OS dependencies. Investigate failures/skips instead of claiming a platform test ran when it did not.

To run a blank local website, use isolated directories:

```sh
export TENNIS_LIBRARY_ROOT="$COURTSIDE_PRIVATE/dev/library"
export TENNIS_DATA_ROOT="$COURTSIDE_PRIVATE/dev/web-state"
mkdir -p "$TENNIS_LIBRARY_ROOT" "$TENNIS_DATA_ROOT"
.venv/bin/python -m flask --app 'app:create_app()' run --host 127.0.0.1 --port 8765
```

An empty library is not a working recording fixture. The browser suite provisions synthetic media/manifests and temporary state. Use its fixtures for review/player checks; do not copy real recordings, location metadata or a live website database into a public test fixture. The website receives no private-originals mount.

## Processing server

```sh
cd "$COURTSIDE_WORKSPACE/tennis-processing-server"
python3.12 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q
```

For a local portal, copy `server.example.json` into the external private development directory. Complete required private settings, point storage to synthetic development folders, and keep watchers disabled. The component's config parser is authoritative; do not invent option names from a proposed architecture.

```sh
export PROCESSING_CONFIG="$COURTSIDE_PRIVATE/dev/server.json"
.venv/bin/python -m flask --app 'app:create_app()' run --host 127.0.0.1 --port 8766
```

The real worker integration test needs the sibling processing checkout and its AV/imageio/numpy runtime. Read the test and owner README before arranging that optional environment. Do not enable real input scanning as a side effect of development setup.

## Video-processing worker

For isolated code tests:

```sh
cd "$COURTSIDE_WORKSPACE/tennis-processing"
python3.12 -m venv .venv
.venv/bin/python -m pip install -r requirements.lock
.venv/bin/python -m pytest -q
```

The lock already contains the test dependency in the reviewed source. Check current pins before adding/replacing dependencies. Full detector execution additionally needs the reviewed external runtime/model artifacts described in `DEPENDENCIES.txt`; this hub does not distribute them. Tests can use synthetic/deterministic detection where provided.

For a standalone Mac installation, follow the component's `install_worker.py` and `--config` instructions, using an external private config. The installer supports `--no-start` for preparation. A preparation command from the worker checkout is:

```sh
python3.12 install_worker.py --config "$COURTSIDE_PRIVATE/video-worker.json" --no-start
```

The file must already contain the operator's valid private configuration; the command does not create that installation profile. Start the installed worker only within the requested scope. The installer manages its own installed source/environment and OS service, independently of the development checkout and any IDE. Review the active-task check before updating an installation.

`workerctl` is the current API-worker control surface. `tennisctl` controls the legacy watcher. Never run both automatic schedulers against the same recordings. See [operations](operations.md).

## Coaching workers

These are packaged under `tennis-web/worker`, not the video-processing worker:

```sh
cd "$COURTSIDE_WORKSPACE/tennis-web"
python3.12 -m venv "$COURTSIDE_PRIVATE/dev/coach-env"
"$COURTSIDE_PRIVATE/dev/coach-env/bin/python" -m pip install ./worker
```

Read [the worker README](https://github.com/haocs/tennis-web/blob/main/worker/README.md) for its required local dependencies/private setup. Render and AI roles have separate processes and state. Development does not require contacting a provider: use synthetic fixtures and mocks. Do not run an AI-role process against a real queue just to check UI behavior.

The render role also handles sharing downloads through an explicit export profile. Keep its video/audio/quality checks separate from coaching evidence generation. Preserve the original-time selection and frozen crop/speed recipe; changing the current UI selection must not relabel an earlier job's result. Exercise delayed responses and close/reopen behavior with mocks.

Website and worker updates can have different compatibility requirements. A sharing job requires a renderer advertising `share-export-v1`; an older worker must leave it unclaimed. Install companion worker updates from the reviewed component revision into the existing independent runtime. Keep private configuration, local recovery state and service identity; do not turn a development checkout or NAS media folder into the installed worker.

## Algorithm lab and this hub

Both are currently documentation-first repositories; do not invent an npm/Python build for them. Validate equations/examples, status claims, cross-links and privacy. When executable evaluation work is introduced, give it a deliberate component-owned runtime and test contract.

## Existing checkouts

Inspect branch and dirty state before updating. Fetch first; fast-forward only a suitable clean checkout. Preserve active work or use a suitable isolated worktree. Avoid force pushes and hard resets to make a multi-agent checkout appear clean. Agents should read [the onboarding guide](agent-guide.md) before crossing Git roots.
