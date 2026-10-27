# atopile as a PCB design engine for a web product

**Research note — 27 October 2026**

**Scope.** Ten questions about using atopile as the PCB engine behind a web product: install/runtime, `ato build` inputs and outputs, 3D rendering, layout API, chat agent, reusable UI, parts registry, licensing, reuse-vs-build, and latency.

**Sources.** Primary sources only:

- Public repository `atopile/atopile`, commit [`619eda7f777558a3e500dbad9cc2941712881495`](https://github.com/atopile/atopile/tree/619eda7f777558a3e500dbad9cc2941712881495) (2026-03-11). All GitHub file links below are pinned to this commit.
- Published package `atopile==0.15.9` on PyPI (uploaded 2026-09-12), both the manylinux wheel and the sdist. Files that exist only in the package are marked **(wheel)** or **(sdist)**.
- Live endpoints: `docs.atopile.io`, `app.atopile.io`, `pypi.org`, `api.github.com`.
- Runtime measurements on this machine: Ubuntu, 4 vCPU, 7.6 GB RAM, Python 3.14.7, KiCad CLI 10.0.7, `uv` 0.12.15, atopile 0.15.9 installed in `/tmp/opencode/ato-venv`.

**Reading convention.** “Verified” means read in the code and exercised at runtime. “Code-level” means read in the source but not exercised end-to-end. “Unverified” means neither confirmed in primary sources nor reproducible here. I never fill a gap with a plausible guess.

---

## 0. Verdicts at a glance

| Question | Short answer |
|---|---|
| **1. Install / runtime** | Python `>=3.14,<3.15` only; `cp314` wheels only; no Linux ARM64 wheel. ~336 MB fresh venv (~380 MB after first imports; 517 MB with the layout editor’s npm deps), 123 installed distributions, ~7 s CLI startup. KiCad not needed to compile/update the PCB; `kicad-cli` needed for 3D and manufacturing exports. |
| **2. `ato build` I/O** | Input: `ato.yaml` + `.ato` sources + existing `.kicad_pcb` + parts/network. Output: the `.kicad_pcb` is rewritten in place (nets/designators/footprints), artifacts under `build/builds/<target>/`. **No netlist file**, no JSON stdout, no incremental build cache. |
| **3. 3D** | `kicad-cli pcb render` → PNG; `kicad-cli pcb export glb` → glTF 2.0; `kicad-cli pcb export step` → STEP. No Blender, no custom renderer. Fixed camera; 12–20 s for a PNG on this host. |
| **4. Layout API** | Placement only: move / rotate / flip / undo / redo. No routing, no track/via editing, no DRC in the editor. The shipped 0.15.9 server is WebSocket-RPC only; the public repo HEAD has a richer REST + standalone editor. |
| **5. Agent** | Real, MIT-licensed agent code with 44 tools in the public repo (31 in 0.15.9), but the shipped version is tied to `gateway.atopile.io` and an atopile account. Reusable as a fork, not as a drop-in service. |
| **6. Reusable UI** | The 2D/3D viewers and the WebGL2 layout renderer are reusable with extraction work. They are not published as an npm package and are coupled to a VS Code webview bridge. |
| **7. Parts registry** | Proprietary/auth-gated. 0.15.9 requires sign-in for part picking; the old `components.atopileapi.com` / `packages.atopileapi.com` hosts no longer resolve; `legacy.atopileapi.com` returns 401. **Biggest blocker** to a self-contained product. |
| **8. Licensing** | atopile code is MIT, but the runtime depends on **AGPL-3.0** `atopile-easyeda2kicad`, KiCad is **GPLv3**, KiCad libraries are CC-BY-SA 4.0, and `occt-import-js` is LGPL-2.1. The “MIT” label is not the whole story. |
| **9. Reuse vs build** | Reuse: `ato` CLI as a build worker, the GLB/STEP/PNG exporters, the viewers, the placement editor. Build: job orchestration, artifact store, auth/multi-tenancy, parts pipeline, routing, agent product shell. |
| **10. Latency** | ~7 s fixed Python startup; 2–11 s stage time for 0–100 modules without parts; 34.7 s stage / 46 s wall for 400 modules, 766 MB RSS; PNG 12–20 s; GLB ~1 s. Real boards with parts not measured (auth blocked). |

---

## 1. The most important finding: there is not one atopile

Every other answer depends on *which* atopile you mean. The public repository, the published package, and the current web app are three different products.

| Artifact | Date | What it is | What it contains |
|---|---|---|---|
| **Public repo `main`** | last commit 2026-03-11; last GitHub release `v0.14.1004` (2026-02-12) | The open-source repo | `src/atopile`, `src/faebryk`, `src/ui-server`, `src/vscode-atopile`, `src/atopile/layout_server`. No `autolayout/`; no `agent/gateway.py`. |
| **PyPI `atopile==0.15.9`** | 2026-09-12 | What `pip install atopile` gets today | 49 declared dependencies, 123 installed distributions; wheel contains `atopile/autolayout/` and `atopile/agent/gateway.py`, which do **not** exist in the public repo; also ships `ui/webview` source (11 MB, 111 `.tsx` files). |
| **`app.atopile.io` (0.16+)** | current | The maintained product | Sign-in required. Docs live inside it. No public API documentation found. The docs stub shows a different language syntax (`@STD::Import`, `@type`) from 0.15’s `import` / `module`. |

Evidence:

- The public repo’s last commit is `619eda7…` on 2026-03-11 ([GitHub API](https://api.github.com/repos/atopile/atopile/commits?per_page=1)); its latest release is `v0.14.1004` ([GitHub releases](https://api.github.com/repos/atopile/atopile/releases?per_page=5)). There is no `v0.15.x` tag.
- PyPI has `0.15.9` uploaded 2026-09-12 ([PyPI JSON](https://pypi.org/pypi/atopile/json)).
- `atopile/autolayout/README.md` exists in the 0.15.9 sdist but has no public-repo counterpart ([sdist files](https://pypi.org/project/atopile/0.15.9/#files)). It describes an AI placement/routing backend that talks to DeepPCB.
- `atopile/agent/gateway.py` **(wheel)** is the 0.15.9 auth gateway; it imports `GATEWAY_BASE_URL = "https://gateway.atopile.io"` from [`atopile/auth/runtime.py:7`](https://pypi.org/project/atopile/0.15.9/#files) and builds OpenAI/Anthropic clients against gateway URLs.
- The 0.15.9 wheel hardcodes `atopile 0.15.8` in user-facing strings: the picker’s sign-in error ([`faebryk/libs/picker/api/api.py:62`](https://pypi.org/project/atopile/0.15.9/#files)) and the MOTD request’s `User-Agent` ([`atopile/cli/motd.py:31`](https://pypi.org/project/atopile/0.15.9/#files)). At runtime the MOTD printed “atopile 0.15.8 is the last CLI release and is in maintenance mode only… replaced by the app at app.atopile.io (atopile 0.16+)”. This is a patched maintenance line, not a clean new release.
- `docs.atopile.io` is now a stub: “The docs live in the editor now” ([docs.atopile.io](https://docs.atopile.io/)); the old `/atopile/guides/install` path returns 404. `app.atopile.io` and `packages.atopile.io` both redirect to a sign-in page.

**Implication.** Pin the *published artifact* (`atopile==0.15.9`) and vendor it. Do not treat the public repo as the source of truth for the shipped product, and do not assume the 0.16 app has a public API. The exact source commit used to build 0.15.9 is **not public** and is unverified.

---

## 2. Q1 — How is atopile installed and run?

### 2.1 Runtime contract

`pyproject.toml` declares Python `>=3.14,<3.15` ([`pyproject.toml:16`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/pyproject.toml#L16)). The 0.15.9 release ships only `cp314` wheels: macOS x86_64/arm64, manylinux x86_64, musllinux x86_64, Windows amd64, plus free-threaded variants. There is **no Linux ARM64 wheel** ([PyPI files](https://pypi.org/project/atopile/0.15.9/#files)). On Graviton/ARM cloud instances you would have to build from the sdist, which needs Zig 0.15.1 and a C++ toolchain; that path was not tested here.

| Item | Measured / verified |
|---|---|
| manylinux wheel size | 12.8 MB |
| Unpacked wheel | 47 MB |
| Installed venv | 336 MB fresh; ~380 MB after first imports (`.pyc`); 517 MB after `bun install` in `ui/webview` (+136 MB `node_modules`) |
| Installed distributions | 123 (49 declared dependencies) |
| Install time (`uv pip install atopile==0.15.9`) | 2 min 12 s cold (network + empty uv cache); 2.5 s with a warm uv cache |
| `ato --version` | 7.06 s wall |
| `python -c "import atopile.cli.cli"` | 3.89 s |

The startup cost is not the compiler: it is Python import plus CLI bootstrap. On this host, importing `atopile.cli.cli` alone takes 3.6–4.0 s, and the import-time profile is dominated by atopile/faebryk modules (`atopile.buildutil`, `atopile.version`, `faebryk.library._F`), not by numeric or plotting libraries. `ato --version` costs ~7 s wall; the remainder is interpreter/CLI bootstrap and per-command work (the compatibility MOTD fetch alone measured ~1 s). A build therefore pays ~4 s of imports plus several seconds of CLI overhead before design work starts. For a web build service this is a fixed per-job tax; a warm worker or a persistent server matters.

### 2.2 Is KiCad required?

No for the core compile-and-update loop. The README states: “KiCad is optional to get started. Without it, you won’t open the PCB, but builds still run and update the `.kicad_pcb`” ([`README.md`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/README.md)). The build steps that need KiCad are tagged `REQUIRES_KICAD` and are skipped or fail with a clear message when `kicad-cli` is absent ([`buildutil.py:158`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/buildutil.py#L158)). Those steps are `step`, `glb`, `3d-image`, `2d-image`, and `mfg-data`.

Yes for 3D and manufacturing. KiCad’s CLI is invoked as an external process, not linked as a Python library. The code pins KiCad library paths to `KICAD_VERSION = "9.0"` ([`paths.py:15`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/libs/kicad/paths.py#L15)), but on this host `kicad-cli` 10.0.7 produced valid PNG/GLB/STEP/Gerber output for an empty board. Whether a KiCad-10-authored board with footprints and 3D models round-trips cleanly is **unverified**.

### 2.3 Deployment recipe

The repo ships a CI Dockerfile based on `ghcr.io/kicad/kicad:9.0.6`, installs Python 3.14 with `uv`, then installs the manylinux wheel ([`dockerfiles/Dockerfile.kicad`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/dockerfiles/Dockerfile.kicad)). That is the sane shape for a web worker: a pinned image with Python 3.14, the 0.15.9 wheel, KiCad CLI + libraries, and no user account.

Practical minimum per worker: 2 vCPU, 2 GB RAM, 2 GB disk for the image plus workspace. A 400-module no-parts design already used 766 MB RSS on this 4-vCPU/7.6 GB host, so 2 GB is a floor, not a comfortable target.

Operational note: every CLI invocation tries to install the KiCad plugin and logs a warning when it cannot find the plugin path. It is noisy but not fatal.

---

## 3. Q2 — What does `ato build` take in and produce?

### 3.1 Inputs

A project is a directory with `ato.yaml`, `.ato` sources, an optional `layout/` directory, and optional `packages/` / `parts/`. A minimal config:

```yaml
requires-atopile: ^0.15.0
paths:
  src: ./
  layout: ./layout
builds:
  default:
    entry: main.ato:App
```

The build also consumes:

- the existing `layout/<target>/<target>.kicad_pcb` (created if missing, then rewritten in place);
- installed part data and, for new parts, the parts registry (network + auth in 0.15.9);
- `ato.yaml` build options (`frozen`, `keep_picked_parts`, `keep_net_names`, `keep_designators`, `standardize_designators`);
- CLI options: `--build/-b`, `--target/-t`, `--exclude-target/-x`, `--frozen`, `--keep-picked-parts`, `--keep-net-names`, `--keep-designators`, `--standardize-designators`, `--standalone`, `--open`, `--all`, `--jobs/-j` (default `os.cpu_count()`), `--verbose` ([`ato build --help`](https://pypi.org/project/atopile/0.15.9/#files); CLI source in [`src/atopile/cli/build.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/cli/build.py)).

### 3.2 The build DAG

`build_steps.py` registers 31 steps in 0.15.9. The important chain is:

1. resolve dependencies → init build context → modify type graph → instantiate app;
2. graph/design checks;
3. load PCB → **pick parts** → prepare nets;
4. post-solve checks → **update PCB** → post-PCB checks;
5. artifact steps: BOM, STEP, GLB, 3D image, 2D image, manufacturing data, manifest, variable report, pinout, power tree, stackup, datasheets, data-interface layout;
6. virtual targets `default` and `all`.

`default` includes BOM, manifest, variables, pinout, stackup, power tree, datasheets and bus layouts. `all` adds GLB and manufacturing data. **`all` does not include `3d-image` or `2d-image`**; those must be requested explicitly. In 0.15.9, `all` depends on `generate_default`, `generate_glb`, and `generate_manufacturing_data` ([`build_steps.py:1258`](https://pypi.org/project/atopile/0.15.9/#files)); `3d-image` is a separate target at line 1001, `2d-image` at line 1021.

### 3.3 Outputs

All paths below are relative to the project root. `<target>` is the build name, usually `default`.

| Artifact | Path | Produced by | Notes |
|---|---|---|---|
| Updated KiCad PCB | `layout/<target>/<target>.kicad_pcb` | `update-pcb` | Rewritten **in place**. Nets, footprints and designators are written here. A timestamped backup is kept under `build/builds/<target>/backups/`. |
| PCB copy | `build/builds/<target>/<target>.kicad_pcb` | `mfg-data` | Only when manufacturing data is built. |
| BOM | `build/builds/<target>/<target>.bom.csv`, `.bom.json` | `bom` | JSON carries `build_id`. |
| Variables | `build/builds/<target>/<target>.variables.ato.json` | `variable-report` | Solver values per node. |
| Power tree | `build/builds/<target>/<target>.power_tree.ato.json` + `build/builds/<target>/power_tree.md` | `power-tree` | |
| Data-interface tree | `build/builds/<target>/<target>.data_interface_tree.ato.json` | `data-interface-layout` | |
| Pinout | `build/builds/<target>/pinout/` | `pinout` | Directory of per-component JSON. Skipped when the design has no components. |
| Stackup | `build/builds/<target>/<target>.stackup.json` | `stackup` | Emitted only when the design defines both a board and a stackup. |
| Testpoints | `build/builds/<target>/<target>.testpoints.json` | `mfg-data` | |
| 3D geometry | `build/builds/<target>/<target>.pcba.glb`, `.pcba.step` | `glb`, `step` | Requires `kicad-cli`. |
| 3D PNG | `build/builds/<target>/<target>.pcba.png` | `3d-image` | **Only if explicitly requested.** |
| 2D SVG | `build/builds/<target>/<target>.pcba.svg` | `2d-image` | **Only if explicitly requested.** |
| Gerber | `build/builds/<target>/<target>.gerber.zip` | `mfg-data` | |
| DXF | `build/builds/<target>/<target>.pcba.dxf/` | `mfg-data` | Directory, not a single file. |
| Pick-and-place | `build/builds/<target>/<target>.pick_and_place.csv`, `.jlcpcb_pick_and_place.csv` | `mfg-data` | JLC conversion is built in. |
| PCB summary | `build/builds/<target>/<target>.pcb_summary.json` | `mfg-data` | Stackup/dimensions; `version` field is `20241229`. |
| Project manifest | `build/manifest.json` | `manifest` | Version `2.0`; maps layouts to `<target>.layouts.json`. |
| Manufacturing bundle | `manufacturing/<target>/` | `collect-manufacturing` | Optional target; copies a fixed suffix list. |

**There is no netlist artifact.** `config.py` defines a `netlist` path ([`config.py:287`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/config.py#L287)), but the only use is setting KiCad’s last-used netlist path ([`pcb.py:62`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/libs/app/pcb.py#L62)). Nets are written directly into the `.kicad_pcb`. If your web product needs a SPICE/HDL netlist, you must generate it separately.

### 3.4 Semantics that matter for a web runner

- **The build mutates the source tree.** `update-pcb` writes the layout file at its source path. The only lock in the CLI guards dependency installation (`.ato/modules/.lock`, [`buildutil.py:51`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/buildutil.py#L51)); nothing serializes two `update-pcb` runs of the same project. Serialize builds per project.
- **`--frozen` is a guard, not a cache.** It fails if the rebuilt PCB differs from the existing one, and writes `<target>.original.kicad_pcb` / `<target>.updated.kicad_pcb` under `build/builds/<target>/` for inspection ([`build_steps.py:718`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/build_steps.py#L718)). Useful in CI; not an incremental-build mechanism.
- **No incremental artifact cache.** Every invocation re-solves the design and re-runs the requested artifact steps. Repeated runs benefit only from OS/KiCad caches. At 400 modules a full build was 46 s wall; an edit-and-rebuild loop pays that again.
- **No JSON output.** `ato build --help` has no `--json`. Human output is Rich/ANSI; the server persists logs in SQLite (`~/.local/state/atopile/log/build_logs.db` on Linux) and streams them over WebSocket ([`sqlite.py:20`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/model/sqlite.py#L20), [`build_queue.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/model/build_queue.py)). For a web runner, use the exit code and the artifacts; capture stdout/stderr for UX only.
- **Exit codes are a stable contract**: `0` success, `1` failure, `130` SIGINT ([`cli.py:275`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/cli/cli.py#L275)).
- **Offline/pinned builds are possible in principle.** `get_pick_tree` returns early for modules that already carry `has_part_picked` ([`picker.py:133`](https://pypi.org/project/atopile/0.15.9/#files)), so a fully pre-picked board with `--keep-picked-parts` should not need registry lookups. This is code-level, not measured end-to-end (we had no account to create a picked board).
- **Concurrency.** `--jobs` defaults to `os.cpu_count()` (4 on this host) and runs builds through a local `BuildQueue`; `--verbose` forces sequential ([`cli/build.py:27`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/cli/build.py#L27), [`cli/build.py:68`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/cli/build.py#L68)). Parallelism is across build targets/projects, not within one design. For a SaaS, cap by memory (766 MB for 400 modules) and by per-project serialization.

---

## 4. Q3 — 3D rendering: what is actually available?

There are two distinct pipelines, both delegated to `kicad-cli`.

### 4.1 Raster PNG (`3d-image`)

`export_3d_board_render()` invokes `kicad-cli pcb render` with:

- `width=1000`, `height=1000`
- `side=top`
- `background=opaque`
- `quality=high`
- `zoom=0.8`
- `rotate=335,0,-45`
- `KIPRJMOD` set to the layout’s parent so 3D model paths resolve

([`artifacts.py:158`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/exporters/pcb/kicad/artifacts.py#L158)).

Observed output on an empty board: `984 × 984` RGBA PNG, 27 KB. The nominal 1000×1000 is the render canvas; the saved image is cropped to the board. The camera is fixed — no turntable, no exploded view, no custom materials. This is a KiCad 3D-viewer render, not a photoreal product shot.

### 4.2 GLB and STEP

`export_glb()` calls `kicad-cli pcb export glb` with tracks, zones, pads, soldermask, silkscreen, substituted models, DNP excluded, vias cut into the body, and `KIPRJMOD` defined ([`artifacts.py:95`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/exporters/pcb/kicad/artifacts.py#L95)). The result is binary glTF 2.0 (`glTF` magic, JSON chunk). `export_step()` produces ISO-10303-21 (`FILE_DESCRIPTION(('KiCad electronic assembly'),'2;1')`).

Observed sizes on an empty board: GLB 5.3 KB, STEP 8.8 KB. Those are not representative; a real board is dominated by per-part 3D models.

### 4.3 Where 3D models come from

Part picking downloads the EasyEDA/LCSC footprint **and STEP model** via `easyeda2kicad` ([`lcsc.py:368`](https://pypi.org/project/atopile/0.15.9/#files)). If a part has no 3D model, the GLB/STEP simply has a hole where the body should be. There is no fallback procedural body. For a web product, 3D quality is therefore a function of the parts catalog, not of atopile.

### 4.4 Viewer side

The public repo’s `src/ui-server` has:

- `ModelViewer.tsx` — Google `<model-viewer>` for GLB, PBR materials, orbit controls ([`ModelViewer.tsx`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/ui-server/src/components/ModelViewer.tsx)).
- `StepViewer.tsx` — three.js + `occt-import-js` (WASM) to tessellate STEP in the browser ([`StepViewer.tsx`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/ui-server/src/components/StepViewer.tsx)).

The 0.15.9 wheel ships a different viewer (`ui/webview/panel-3d`) that uses three.js + `GLTFLoader` directly, plus `occt-import-js` ([`ui/webview/panel-3d/threeDPreviewManager.tsx`](https://pypi.org/project/atopile/0.15.9/#files)). Either approach works; both are MIT-licensed application code with an LGPL-2.1 WASM dependency.

### 4.5 Timings

On the empty board:

| Target | Stage time | Wall | RSS |
|---|---:|---:|---:|
| `3d-image` (first run) | 19.86 s | 34.4 s | 171 MB |
| `3d-image` (warm) | 12.09 s | 20.0 s | 171 MB |
| `glb` | 0.83 s | 8.1 s | 137 MB |
| `step` (inside `mfg-data`) | 0.81 s | — | — |

PNG rendering is the expensive 3D artifact because it ray-renders the board; GLB/STEP export is cheap. Recommendation: generate GLB on every successful build (cheap, interactive), generate PNG on demand or precompute as a thumbnail, and generate STEP only on download.

---

## 5. Q4 — Is there a layout API we can drive?

Yes, but it is a **placement** API, not a routing/editing API. And the API surface differs between the public repo and the shipped 0.15.9.

### 5.1 Two implementations

| | Public repo HEAD (`619eda7`) | PyPI 0.15.9 |
|---|---|---|
| Server | Standalone FastAPI app + REST + WebSocket | Standalone FastAPI app + **WebSocket RPC only** |
| REST | `GET /api/layout/render-model`, `GET /api/layout/footprints`, `POST /api/layout/execute-action` ([`server.py:52`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server/server.py#L52)) | None; only `/` and `/static/*` in OpenAPI |
| WebSocket | `/ws/layout` ([`server.py:116`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server/server.py#L116)) | `/ws` with actions `subscribeLayout`, `unsubscribeLayout`, `getLayoutRenderModel`, `executeLayoutAction` ([`layout_rpc.py:16`](https://pypi.org/project/atopile/0.15.9/#files)) |
| Frontend | Full vanilla TS + WebGL2 editor, ~26.6k LOC, esbuild bundle ([`frontend/src`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server/frontend/src)) | Thin 106-line shell reusing `ui/webview/common/layout` (~27.9k LOC), built with bun at runtime |
| Packaging | `editor.js` built into `static/` by the repo build | `editor.js` **not shipped**; `__main__` runs `bun run build` on startup ([`__main__.py:109`](https://pypi.org/project/atopile/0.15.9/#files)) |

### 5.2 Commands

The action model is exactly five commands ([`models.py:277`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server/models.py#L277); 0.15.9 has the same model in [`layout_models.py:282`](https://pypi.org/project/atopile/0.15.9/#files)):

- `move` — UUIDs, `dx`, `dy`
- `rotate` — UUIDs, `delta_degrees`
- `flip` — UUIDs
- `undo`
- `redo`

There is no command for drawing a track, placing a via, editing a zone, changing a net, editing design rules, or running DRC. The render model includes tracks, vias, zones, pads, drawings and text, so the editor *displays* copper, but it does not edit it.

Each successful action saves the **entire** `.kicad_pcb` with `kicad.dumps` ([`pcb_manager.py:408`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server/pcb_manager.py#L408)), broadcasts a delta to WebSocket clients, and (in 0.15.9) tries to reload the file in a running KiCad. Undo/redo is in-memory only; restarting the process loses the stack.

DRC exists separately: `faebryk.libs.kicad.drc.run_drc` shells out to `kicad-cli pcb drc --format json`, parses KiCad’s JSON report, and is exposed as the agent tool `layout_run_drc` in the public repo ([`drc.py:13`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/libs/kicad/drc.py#L13), [`tools.py:1704`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent/tools.py#L1704)). It is not wired into the layout server UI.

### 5.3 Running it

The documented standalone command is:

```bash
python -m atopile.layout_server path/to/board.kicad_pcb --port 8100
```

([`layout_server/README.md`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server/README.md)).

On a fresh 0.15.9 install this command **fails out of the box**. `__main__.py` rebuilds the editor bundle at startup with `bun run build`; the bundle is not in the wheel, and the shared `ui/webview` tree ships without `node_modules`, so the build cannot resolve `earcut` and aborts. I reproduced the failure, ran `bun install` in `ui/webview` (142 packages), and re-ran the server: it then returned HTTP 200 and served the editor. This is a packaging bug, not a design limitation — but it means the layout editor is not a turnkey component of the 0.15.9 wheel.

### 5.4 Routing / autolayout

There is **no autorouter in the public repo**. The shipped 0.15.9 package adds `atopile/autolayout/`, which:

- uploads a board file to `https://api.deeppcb.ai` (`POST /api/v1/files/uploads/board-file`);
- creates a job, polls it, downloads candidate revisions;
- requires a bearer token obtained through the atopile gateway;
- consumes DeepPCB credits.

See [`atopile/autolayout/README.md`](https://pypi.org/project/atopile/0.15.9/#files) and `atopile/autolayout/deeppcb/client.py` (`base_url="https://api.deeppcb.ai"`, `Authorization: Bearer …`). This is a remote proprietary service. It is not self-hostable from the published code, and its routing quality, price, data retention and SLA are **unverified**.

### 5.5 Implications

- For an MVP, the layout editor is good enough for **placement**: load a board, drag/rotate/flip parts, undo, save. That is a real, useful feature.
- It is not a KiCad replacement. Routing, copper editing, DRC fixing and design-rule editing must come from somewhere else (KiCad desktop, DeepPCB, or your own editor).
- Multi-tenancy is not built in: one `LayoutService` holds one active PCB and an in-memory undo stack. Run one instance per user session, put it behind your auth proxy, and do not expose port 8100 directly.
- The file format is KiCad 9 (`version 20241229`). Opening a board written by KiCad 10 and saving it through atopile may downgrade or drop newer constructs; **unverified**, and a real risk if users bring their own boards.

---

## 6. Q5 — Is there an agent we can reuse?

Yes, there are two agents, and they are not the same.

### 6.1 Public repo HEAD agent

`src/atopile/server/agent/` is a full agent runtime:

- **44 tools** registered in [`tools.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent/tools.py), including `parts_search`, `parts_install`, `build_run`, `design_questions`, `layout_get_component_position`, `layout_set_component_position`, `layout_run_drc`, file read/edit/search, package search, datasheets and web search.
- OpenAI **Responses API** provider, default model `gpt-5.4`, summary model `gpt-4.1-nano` ([`config.py:42`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent/config.py#L42)).
- Configurable base URL and key via `ATOPILE_AGENT_BASE_URL`, `ATOPILE_AGENT_OPENAI_API_KEY`, model/timeout/loop env vars ([`config.py:99`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent/config.py#L99)).
- Limits: 240 tool loops, 7200 s per turn ([`config.py:47`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent/config.py#L47), [`config.py:110`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent/config.py#L110)).
- REST surface: `/api/agent/sessions`, `/messages`, `/runs`, `/steer`, `/interrupt` ([`routes/agent/main.py:84`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/routes/agent/main.py#L84)).
- Prompt/skills are files: `atopile/agent/skills/*/SKILL.md` in the wheel, and the repo’s agent docs.

This is genuinely reusable: MIT code, explicit tool contracts, your own OpenAI key, no atopile account. The catch is coupling: it expects an `AppContext`, a project root, the build queue, SQLite agent logs, and a filesystem sandbox. Extracting it is a fork-and-adapt job, not `npm install`.

### 6.2 Shipped 0.15.9 agent

The wheel contains a newer, differently structured agent:

- 31 tools (files, build, parts, packages, checklist, skills, reporting, stdlib, web search); **no layout tools**. Verified at runtime by building the registry and listing its definitions.
- All model calls go through `gateway.atopile.io`; `openai_client()` raises `RuntimeError("Not signed in. Sign in from the extension sidebar.")` without an atopile token ([`atopile/agent/gateway.py`](https://pypi.org/project/atopile/0.15.9/#files)).
- Model list includes `gpt-5.4`, `gpt-5.4-nano`, `claude-opus-4-7`, `claude-sonnet-4-6` ([`atopile/agent/providers/base.py`](https://pypi.org/project/atopile/0.15.9/#files)).
- Exposed through the UI WebSocket (`agent.createSession`, `agent.createRun`, `agent.steerRun`, `agent.cancelRun`, `agent.setModel`), not a public REST API ([`atopile/server/ui/websocket.py`](https://pypi.org/project/atopile/0.15.9/#files)).

So the shipped agent is **not self-hostable** without an atopile account, and it cannot edit layout. The public repo agent is the better base for a product, even though it is older.

### 6.3 What the agent actually does

It is an implementation agent, not a chat assistant. From the official skill:

- turns a request into `.ato` files, uses `design_questions` to batch unresolved decisions, creates a checklist, edits files, installs parts, runs builds, reads logs, and iterates until the build passes;
- has a “do not end the turn to ask permission” bias;
- can run 240 tool loops and 2 hours per turn.

That is powerful and risky. For a multi-tenant web product you need: a container sandbox per session, path allowlists, network egress policy, build timeouts, token/cost budgets, audit logs, and human approval for destructive operations. The repo has policy/scope/circuit-breaker modules, but they are not a complete multi-tenant security boundary.

### 6.4 Cost and latency

**Unverified.** No token-usage or per-design cost measurement was possible without an account. Expect the agent to be the dominant variable cost: file reads/edits and build logs are large, and 240 tool loops can burn a lot of context. Cap loops and turn time well below the defaults for a consumer product.

---

## 7. Q6 — Which UI pieces can we reuse?

### 7.1 Inventory

| Component | Technology | License | Reuse assessment |
|---|---|---|---|
| `GerberViewer.tsx` | `@tracespace/core`, `react-zoom-pan-pinch` | MIT | **Good.** Renders `gerber.zip` as a realistic fab preview, top/bottom, layer panel. Needs artifact URL plumbing. |
| `ModelViewer.tsx` | Google `<model-viewer>` (GLB) | Apache-2.0 (bundle contains BSD-3-Clause Lit) | **Good.** Drop-in for GLB; PBR materials; orbit controls. |
| `StepViewer.tsx` | three.js + `occt-import-js` (WASM) | MIT + LGPL-2.1 | **Good, with care.** STEP tessellation in a worker; LGPL obligations. |
| `KiCanvasEmbed.tsx` | KiCanvas web component | MIT | **Good.** Interactive `.kicad_pcb` viewer with layers, nets, board stack. |
| `AgentChatPanel.tsx` | React chat UI | MIT | **Medium.** Useful UX reference; tied to the backend’s message/run protocol. |
| Layout editor | Vanilla TS + WebGL2 + earcut | MIT | **Good for placement.** Standalone, no React; iframe-able. |
| 0.15.9 `ui/webview` | React/TS source, 111 `.tsx`, 11 MB | MIT | **Reference only.** Not a packaged library; built by the extension. |

The repo also has `ComponentShowcase.tsx`, but there is no published component library, no Storybook, and no npm package. The components import app-specific stores, API clients and VS Code theme variables.

### 7.2 What “reuse” actually costs

- **Extraction:** copy the viewer components, replace the `api/` postMessage bridge with `fetch`/WebSocket calls, replace VS Code CSS variables with your design tokens, and add error/loading states.
- **Build artifacts:** the viewers expect a build target’s artifact paths (`pcba.glb`, `pcba.step`, `gerber.zip`). Your backend must expose them with stable URLs or signed object-storage URLs.
- **Licensing:** keep the MIT/Apache/BSD notices; do not strip the `occt-import-js` LGPL notice. If you bundle KiCanvas, keep its MIT notice.

### 7.3 Recommendation

Reuse **three viewers + the placement editor**, not the whole `ui-server`. For 2D, use `GerberViewer` for manufacturing preview and `KiCanvasEmbed` for board inspection; they answer different questions. For 3D, serve GLB for interactive preview and STEP for download, not the other way round. For layout, iframe the standalone editor behind your auth proxy for the MVP.

---

## 8. Q7 — What is the parts registry, and can we self-host it?

### 8.1 Architecture in the OSS code

`faebryk.libs.picker.api.api.ApiClient` is the registry client ([`api.py:50`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/libs/picker/api/api.py#L50)). It calls:

- `GET /v0/component/lcsc/{lcsc}`
- `GET /v0/component/mfr/{mfr}/{mfr_pn}`
- `POST /v0/query/{method}`
- `POST /v0/query`

The base URL is `config.project.services.components.url`; the repo default is `https://components.atopileapi.com` ([`config.py:804`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/config.py#L804)). 0.15.9 changes the default to `https://legacy.atopileapi.com` ([`atopile/config.py`](https://pypi.org/project/atopile/0.15.9/#files)).

### 8.2 Current availability

| Endpoint / version | State |
|---|---|
| `components.atopileapi.com` | DNS does **not** resolve. |
| `packages.atopileapi.com` | DNS does **not** resolve. |
| `legacy.atopileapi.com` | Resolves; API paths return **401** without a token (the public `/v0/motd` returns 200). |
| `atopile==0.15.9` part picking | Hard-fails without sign-in: “Part picking on atopile 0.15.8 requires sign-in. Run `ato auth login`, or migrate to app.atopile.io (0.16+)” ([`api.py:62`](https://pypi.org/project/atopile/0.15.9/#files)). |
| `atopile==0.14.1004` | No auth check, but the default host is dead; also breaks on `more-itertools>=11` (`ImportError: cannot import name 'zip_equal'`) and cannot read the 0.15 SQLite schema (`no column named display_name`). Not a viable fallback. |

So there is **no supported self-hosted or offline registry** today.

### 8.3 Search vs lookup

- **Lookup by LCSC ID** goes through the atopile API client (auth required in 0.15.9).
- **Keyword search** in 0.15.9 uses the public JLCPCB API, not the atopile registry: `PartCatalog.search()` delegates to `search_jlc_parts()` ([`atopile/model/parts.py:179`](https://pypi.org/project/atopile/0.15.9/#files)). The module docstring calls itself “JLCPCB public parts search provider (hacky)”, rate-limits to 5 requests/second, and scrapes `jlcpcb.com/api/overseas-pcb-order/...` ([`parts_search_jlc.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/domains/parts_search_jlc.py)). That is fragile and subject to JLCPCB’s terms.
- **Part install** downloads EasyEDA data (footprint, symbol, 3D model) and writes it into the project’s parts/packages tree ([`ProjectParts.install_raw`](https://pypi.org/project/atopile/0.15.9/#files)). The 3D model comes from EasyEDA, not atopile.

### 8.4 Can we make it work?

Options, in increasing order of effort:

1. **Use an atopile account/token** (if the vendor permits API use). This is the only supported path for new part picking in 0.15.9. It introduces a vendor dependency, account provisioning and rate limits.
2. **Pre-pick parts outside the web product.** Build boards with parts already pinned, then run `ato build --keep-picked-parts` in the product. Code-level: already-picked modules are skipped by `get_pick_tree`, so no registry call should happen. This is the cleanest MVP if your users choose from a curated catalog rather than arbitrary LCSC parts.
3. **Self-host a registry shim** implementing the `/v0` shape and point `services.components.url` at it. The 0.15.9 client still requires *a* token to be present (`get_auth_token()`), so you would need to inject one (patched runtime or pre-seeded auth). This is unsupported, brittle across releases, and a maintenance burden.
4. **Replace the picker** with your own parts service/parametric search. This is a large project: you need component attributes (voltage, current, package, stock, price), footprints, symbols, 3D models and a solver-compatible picker. It is the only path to full independence.

**Verdict:** parts are the single biggest blocker to a self-contained atopile-powered product. If arbitrary part selection is a core requirement, budget for option 4 or negotiate access; do not assume the old public registry will come back.

---

## 9. Q8 — Licensing: what is actually MIT?

### 9.1 License map

| Component | License | What it means for a web product |
|---|---|---|
| atopile source (repo) | **MIT** ([`LICENSE`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/LICENSE)) | Commercial use, modification, redistribution allowed with notice. No warranty. |
| PyPI `atopile` metadata | **MIT** ([PyPI JSON](https://pypi.org/pypi/atopile/json)) | Same. |
| VS Code extension `src/vscode-atopile` | **Apache-2.0** per its `LICENSE`; `package.json` says **MIT** ([`LICENSE`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/vscode-atopile/LICENSE), [`package.json:9`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/vscode-atopile/package.json#L9)) | Inconsistent metadata inside the repo. Treat as Apache-2.0 (the stricter, file-level license): patent grant; notice/state-changes obligations. |
| `atopile-easyeda2kicad` (runtime dependency) | **AGPL-3.0** ([PyPI](https://pypi.org/pypi/atopile-easyeda2kicad/json)) | Strong network copyleft. Imported for LCSC→KiCad conversion; every build imports the picker stack. Distribution/network-use obligations need counsel. |
| KiCad (`kicad-cli`, libraries) | **GPLv3** for the program; **CC-BY-SA 4.0** for libraries ([kicad.org/licenses](https://www.kicad.org/about/licenses/)) | Invoking `kicad-cli` as a separate process is the usual boundary; distributing a KiCad-containing image triggers GPL source/notice obligations for KiCad. Libraries have attribution/share-alike terms. |
| `occt-import-js` | **LGPL-2.1** ([`package-lock.json`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/ui-server/package-lock.json)) | WASM STEP importer in the browser; LGPL relinking/source obligations. |
| `<model-viewer>` | **Apache-2.0**; bundled Lit is **BSD-3-Clause** ([bundle header](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/vscode-atopile/resources/model-viewer/model-viewer.min.js)) | Keep notices. |
| KiCanvas | **MIT** | Keep notice. |
| `@tracespace/*`, three.js, React, earcut | **MIT / ISC** | Keep notices. |
| atopile registry data, DeepPCB, gateway | **Proprietary / unknown** | Terms of service, data processing and IP ownership are unverified. |

**The headline:** the repository’s MIT license does not make the whole runtime MIT. `atopile-easyeda2kicad` is AGPL-3.0 and is a declared runtime dependency ([`pyproject.toml:30`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/pyproject.toml#L30)); KiCad is GPLv3; the STEP viewer is LGPL-2.1. This needs legal review before a commercial launch, especially if you distribute a container image or offer the service over the network.

### 9.2 Telemetry

atopile initializes a PostHog client at import, sending to `https://telemetry.atopileapi.com` with a write-only project key ([`telemetry.py:69`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/telemetry.py#L69), host at line 72). It collects a hashed project ID from the git remote, git user email, platform/Python info, build events and exceptions ([`telemetry.py:216`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/telemetry.py#L216)).

The reliable opt-outs are:

- set `CI=1` in the worker environment (the code checks `CI` and returns a mock client), or
- pre-write `~/.config/atopile/telemetry.yaml` with `telemetry: false` ([`telemetry.py:133`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/telemetry.py#L133)).

`ATO_DISABLE_TELEMETRY` appears only in a log message, not in the condition ([`telemetry.py:65`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/telemetry.py#L65)). For a multi-tenant product, disable telemetry at the image level and block egress to the telemetry host.

### 9.3 Other IP points

- The agent sends project files and build logs to an LLM (your key in the repo version; atopile’s gateway in 0.15.9). Your privacy policy must cover that.
- DeepPCB uploads the board file to `api.deeppcb.ai`. Data retention and ownership are unverified.
- LCSC/EasyEDA footprints and 3D models may carry vendor terms; the registry data is not clearly licensed for redistribution.
- “atopile” is a trademark; do not imply endorsement.

*This is not legal advice. Have counsel review the AGPL/GPL/LGPL boundaries and the vendor terms before launch.*

---

## 10. Q9 — What can we reuse, and what must we build?

Effort legend: **S** < 2 weeks, **M** 2–6 weeks, **L** > 6 weeks for one experienced engineer, excluding product design/QA.

| Capability | What atopile gives us | What is missing for a web product | Verdict | Effort |
|---|---|---|---|---|
| **Chat agent** | MIT agent loop, 44 tool contracts, prompts/skills, OpenAI Responses provider, REST routes in repo HEAD ([`server/agent`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent)) | Product transport/auth, per-tenant sandbox, quotas, cost control, persistence, evals, layout tools in 0.15.9. The shipped gateway is not self-hostable. | **Fork the repo HEAD agent; build the product shell.** Do not use 0.15.9’s gateway as the only path. | **L** |
| **Build runner** | `ato` CLI with 31-step DAG, exporters, `--frozen`, exit codes, official KiCad Dockerfile | Job queue, workspace lifecycle, per-project lock, artifact store/CDN, log streaming, retries/timeouts, version pinning, registry/auth handling, cleanup | **Reuse the CLI as a subprocess; build the service around it.** | **M** |
| **2D viewer** | `GerberViewer` (tracespace), `KiCanvasEmbed`, static SVG | Artifact URLs, extraction from webview bridge, layer UX, mobile/responsive, caching | **Reuse the components; adapt the plumbing.** | **S–M** |
| **3D viewer** | `ModelViewer` (GLB), `StepViewer` (three + occt), `3d-image`/`glb`/`step` exporters | On-demand generation, cache by build ID, CDN, missing-model fallbacks, STEP worker, mobile performance | **Reuse the viewers; build the artifact service.** | **S–M** |
| **Layout editing** | Standalone WebGL2 editor + move/rotate/flip/undo/redo + any `.kicad_pcb`; DRC via `kicad-cli`; DeepPCB remote autolayout in 0.15.9 | Routing, track/via editing, zone editing, net highlighting, design rules, transactions, multi-board, KiCad 10 fidelity, auth/multi-tenancy | **Reuse for placement MVP.** Build or buy routing separately; do not promise a full editor. | **S** for placement, **L** for full editor |
| **Parts registry** | LCSC lookup client, EasyEDA conversion, project-local part install, JLC search | A supported, licensed, self-hosted catalog with attributes/pricing/stock/3D; auth; rate limits | **Build/partner.** This is not a reuse item. | **L** |
| **Accounts / billing / projects** | Nothing product-grade | Everything | **Build.** | **M** |
| **Manufacturing handoff** | Gerber, BOM, pick-and-place, DXF, JLC naming | Quote/order integration, DFM checks, vendor APIs | **Build/partner.** | **M** |

**Overall shape.** atopile is a strong **compile-and-export engine** and a decent **placement editor**. It is not a turnkey web product: no multi-tenant build service, no supported parts backend, no routing, no product auth, and a licensing surface that needs review. The shortest credible MVP is “pinned 0.15.9 worker + curated/pre-picked parts + viewers + placement editor”; the longest pole is parts and routing, not the compiler.

---

## 11. Q10 — Latency and capacity

### 11.1 Method

All measurements are on the host described above, with atopile 0.15.9 + KiCad CLI 10.0.7, warm filesystem, no parts (the empty/scale projects contain only `ElectricPower` interfaces, so the picker has nothing to fetch). “Stage” is the sum of Muster stage timings printed by the CLI; “wall” is process start to exit and includes ~7–10 s of process startup (of which ~4 s is Python imports). Runs on a shared 4-vCPU box varied by up to ~2×, so treat these as indicative ranges, not SLA numbers.

### 11.2 Measurements

| Scenario | Stage time | Wall | Max RSS |
|---|---:|---:|---:|
| `ato --version` | — | 7.06 s | — |
| `import atopile.cli.cli` | — | 3.89 s | — |
| Empty board, `default` (cold) | 3.25 s | 11.12 s | 134 MB |
| Empty board, `default` (warm) | 2.32 s | 12.8–13.0 s | 134 MB |
| Empty board, `3d-image` (first) | 19.86 s | 34.4 s | 171 MB |
| Empty board, `3d-image` (warm) | 12.09 s | 20.0 s | 171 MB |
| Empty board, `glb` | 0.83 s | 8.10 s | 137 MB |
| Empty board, `mfg-data` (warm) | 3.19 s | 11.23 s | 220 MB |
| Empty board, `mfg-data` (first) | 12.62 s | — | — |
| 20 modules, `default` | 2.06–3.56 s | 8.8–13.9 s | 155 MB |
| 100 modules, `default` | 6.03–11.34 s | 15.2–22.2 s | 245 MB |
| 400 modules, `default` | 34.74 s | 46.13 s | 766 MB |
| 20 modules, `all` | 28.49 s | — | — |

The 20-module `all` run includes GLB + manufacturing exports, which is why its stage time is much higher than `default`. It is the closest no-parts proxy for a “real” build with 3D and fab outputs.

### 11.3 What the numbers mean

- **Fixed cost dominates small builds.** ~7–10 s of process startup (of which ~4 s is imports) + ~2 s of design work means an empty build is ~11–13 s wall. Caching or a warm process is the only way to make interactive rebuilds feel fast.
- **Design size scales super-linearly near 400 modules.** 100 modules: ~6–11 s stage; 400 modules: ~35 s stage, ~766 MB. Memory grows roughly with the instance graph; plan ~2 MB per module plus a baseline.
- **3D PNG is the expensive artifact.** 12–20 s even on an empty board; a real board with many 3D models will be worse. GLB is ~1 s. Do not render PNG on every keystroke; do it on demand or as a background thumbnail.
- **Manufacturing exports are moderate.** Gerber/STEP/DXF/pick-and-place ~3–13 s on an empty board; real boards with many footprints will be slower, especially STEP with 3D models.
- **No parts measured.** Real part picking adds network latency, solver work, EasyEDA downloads and 3D model conversion. This is the largest unknown for capacity planning (see Unverified).
- **Concurrency.** With 766 MB for 400 modules and 4 vCPU, a 2 vCPU/2 GB worker is the minimum; 1 build per 2 vCPU is a sensible starting cap. Four concurrent 400-module builds on this 7.6 GB host would be at ~3 GB just for atopile processes, plus KiCad and the OS.

### 11.4 Capacity recommendation

Start with a queue and a small pool of warm workers:

- 2 vCPU / 2 GB per worker, 1 build at a time;
- 10–20 GB ephemeral disk per workspace (venv + node_modules + build artifacts);
- hard timeout of 10 minutes per build (400 modules is 46 s; real boards with parts/3D will be minutes, not hours);
- separate “export” queue for 3D/manufacturing so interactive compile builds are not blocked;
- per-project mutex to avoid concurrent writes to the same `.kicad_pcb`;
- cache `~/.cache` / KiCad libraries / pip wheels in the image, not per job.

---

## 12. Recommended integration architecture

```
Web app (React/Next)
   │  auth, projects, editor, viewers
   ▼
API / job service (Postgres + Redis)
   │  enqueue build, stream logs, store artifacts
   ▼
Worker image: Python 3.14 + atopile 0.15.9 + KiCad CLI
   │  one workspace per job, one build per project lock
   ├─ ato build --build <target> [--target 3d-image|mfg-data]
   ├─ write artifacts to object storage (S3/R2)
   └─ emit stage/log events (parse stdout; do not rely on JSON)
   ▼
Viewers: Gerber (tracespace) · GLB (<model-viewer>) · STEP (occt) · Layout (iframe)
```

**MVP choices**

1. **Pin `atopile==0.15.9`** in the image; do not track `main`.
2. **Disable telemetry** (`CI=1` or `telemetry.yaml`) and block `telemetry.atopileapi.com`.
3. **Curate the parts catalog** or require pre-picked projects; do not promise arbitrary LCSC picking in v1.
4. **Build the PCB on the server**, store the `.kicad_pcb` and artifacts; treat the layout file as the source of truth for placement.
5. **Use the placement editor** for move/rotate/flip; route users to KiCad/DeepPCB for copper.
6. **Render GLB on build, PNG on demand**; cache by build ID.
7. **Isolate layout server instances** per session, behind auth; never expose `:8100`.
8. **Review AGPL/GPL/LGPL** before distributing the worker image.

**What not to do**

- Do not build on repo HEAD’s default parts host; it is dead.
- Do not use 0.15.9’s gateway agent as the only agent path; it requires an atopile account and has no layout tools.
- Do not assume `ato build` is incremental; every build is a full solve.
- Do not let two builds write the same project concurrently.
- Do not treat `docs.atopile.io` as the 0.15 docs; it now describes the 0.16 app and its new syntax.

---

## 13. Go / no-go

**Go, with 0.15.9 and a curated parts flow, if the product needs:**

- compile `.ato` to a KiCad board, BOM, Gerber, pick-and-place, GLB/STEP/PNG;
- placement editing (move/rotate/flip) in the browser;
- a design agent that edits files and runs builds;
- a mostly self-contained engine with a known, pinned artifact.

**No-go with the current OSS stack, if the product needs:**

- arbitrary LCSC part search/picking without an atopile account;
- tracks/vias/routing/DRC fixing in the browser;
- KiCad 10 round-trip fidelity for user-supplied boards;
- a vendor-supported SLA, security patches, or a public 0.16 API;
- a clean MIT-only licensing story.

The gap is not the compiler. It is the **parts supply chain**, **routing**, and **productization** around a maintenance-mode CLI.

---

## 14. Unverified / non vérifié

The following could not be verified from primary sources or reproduced here. They are material to a production decision.

1. **Real-board build latency and quality with parts.** No atopile account, so part picking (and therefore a realistic board) could not run. All timings above are no-part designs.
2. **The exact source commit of 0.15.9.** The public repo is older and lacks files present in the package; the internal build commit is not public.
3. **Current 0.16+ app behavior, API, pricing, SLA and data policy.** `app.atopile.io` requires sign-in; no public API docs were found.
4. **DeepPCB routing quality, price, credits, data retention and self-host options.** Only the client code and README were inspectable.
5. **KiCad 10 round-trip fidelity.** 0.15.9 emits `version 20241229` (KiCad 9); opening/saving a KiCad-10-authored board was not tested.
6. **Linux ARM64 support.** No ARM64 wheel; sdist build on ARM was not attempted.
7. **Clean-host install size/time across OSes.** One Linux x86_64 host was measured.
8. **Agent token usage, latency and cost per design.** No LLM key/account was used.
9. **Parts registry terms of service, data licensing and long-term availability.** The endpoints are gated/dead; terms were not reviewed.
10. **Legal interpretation of AGPL/GPL/LGPL boundaries** for a hosted service and a distributed container. Not legal advice.
11. **Whether the 0.15.9 layout editor starts on a clean host without a manual `bun install`.** It failed here until 142 frontend packages were installed; a packaged fix may exist in a future release.
12. **Whether `--keep-picked-parts` fully avoids all registry/network calls.** Code inspection says yes for picked modules; not measured end-to-end.

---

## 15. Source index

**Public repo (pinned to `619eda7f777558a3e500dbad9cc2941712881495`)**

- Build DAG: [`src/atopile/build_steps.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/build_steps.py)
- Exporters: [`src/faebryk/exporters/pcb/kicad/artifacts.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/exporters/pcb/kicad/artifacts.py)
- Config: [`src/atopile/config.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/config.py)
- CLI: [`src/atopile/cli/build.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/cli/build.py), [`src/atopile/cli/cli.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/cli/cli.py)
- Layout server: [`src/atopile/layout_server/`](https://github.com/atopile/atopile/tree/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/layout_server)
- Agent: [`src/atopile/server/agent/`](https://github.com/atopile/atopile/tree/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/server/agent)
- UI: [`src/ui-server/src/components/`](https://github.com/atopile/atopile/tree/619eda7f777558a3e500dbad9cc2941712881495/src/ui-server/src/components)
- Build queue: [`src/atopile/model/build_queue.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/model/build_queue.py)
- DRC: [`src/faebryk/libs/kicad/drc.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/faebryk/libs/kicad/drc.py)
- Telemetry: [`src/atopile/telemetry.py`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/telemetry.py)
- Docker/KiCad: [`dockerfiles/Dockerfile.kicad`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/dockerfiles/Dockerfile.kicad)
- Licenses: [`LICENSE`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/LICENSE), [`src/vscode-atopile/LICENSE`](https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/vscode-atopile/LICENSE)

**Published package 0.15.9**

- Files and hashes: [https://pypi.org/project/atopile/0.15.9/#files](https://pypi.org/project/atopile/0.15.9/#files)
- manylinux wheel SHA-256: `9b1571061b74a8a098b354408d2e5d15cec962ff9dc75d72c22c00b8322fbffc`
- Package-only paths referenced above: `atopile/autolayout/`, `atopile/agent/gateway.py`, `atopile/model/parts.py`, `atopile/model/parts_search_jlc.py`, `atopile/server/domains/layout_*.py`, `atopile/layout_server/__main__.py`, `ui/webview/`.

**External**

- atopile docs (stub): [https://docs.atopile.io/](https://docs.atopile.io/)
- atopile app (sign-in): [https://app.atopile.io/](https://app.atopile.io/)
- PyPI metadata: [https://pypi.org/pypi/atopile/json](https://pypi.org/pypi/atopile/json)
- GitHub releases/commits: [https://api.github.com/repos/atopile/atopile/releases](https://api.github.com/repos/atopile/atopile/releases)
- `atopile-easyeda2kicad` license: [https://pypi.org/pypi/atopile-easyeda2kicad/json](https://pypi.org/pypi/atopile-easyeda2kicad/json)
- KiCad licenses: [https://www.kicad.org/about/licenses/](https://www.kicad.org/about/licenses/)

---

## 16. Résumé (français)

1. atopile 0.15.9 est un vrai compilateur/solveur PCB, mais le dépôt public (mars 2026) est en retard sur le paquet PyPI (sept. 2026) : `autolayout/` et `agent/gateway.py` ne sont que dans le paquet.
2. Le CLI est en maintenance ; la doc publique est devenue un stub et le produit 0.16+ vit derrière `app.atopile.io` (connexion obligatoire), avec une syntaxe différente.
3. Installation : Python 3.14 uniquement, roues `cp314` (pas d’ARM64 Linux), ~336 Mo de venv (~380 Mo après imports ; 517 Mo avec les paquets npm de l’éditeur), 123 paquets installés, ~7 s de démarrage.
4. `ato build` réécrit le `.kicad_pcb` sur place et produit BOM, Gerber, pick-and-place, GLB/STEP (PNG/SVG sur demande) ; **aucun netlist**, pas de sortie JSON, pas de cache incrémental.
5. La 3D passe uniquement par `kicad-cli` (pas de Blender) : PNG 12–20 s, GLB ~1 s, STEP ~1 s ; la qualité dépend des modèles 3D EasyEDA.
6. L’API layout ne fait que déplacer/tourner/retourner/undo/redo ; pas de routage, pas de DRC dans l’éditeur ; le 0.15.9 est en WebSocket RPC et son éditeur exige `bun` + 142 paquets pour démarrer.
7. L’agent est réutilisable (MIT, 44 outils dans le dépôt), mais la version livrée passe par le gateway atopile et n’a pas d’outils layout ; il faut le forker et bâtir l’enveloppe produit.
8. Les viewers 2D/3D et l’éditeur de placement sont réutilisables ; ils ne sont pas packagés en npm et restent couplés au webview VS Code.
9. Le registre de composants est le blocage majeur : authentification obligatoire en 0.15.9, anciens hôtes morts, recherche JLC « hacky » ; pas de registre auto-hébergé supporté.
10. Licence : code MIT, mais dépendance AGPL-3.0 (`atopile-easyeda2kicad`), KiCad GPLv3, bibliothèques CC-BY-SA, `occt-import-js` LGPL-2.1 ; revue juridique indispensable avant lancement.
