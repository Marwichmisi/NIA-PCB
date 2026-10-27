# Routing for NIA-PCB: from `ato` to routed, DRC-clean Gerbers

**Date:** 2026-10-27. **Scope:** one board class only — 2 layers, low-speed, 5 V / 3.3 V, < 50 components, no RF/BGA. **Question:** how to get a routed, DRC-clean, Gerber-exported board from an `ato` design — and whether to route at all or start from pre-routed templates.

**Method:** primary sources only (vendor docs, pricing pages, GitHub repos, KiCad CLI docs, local CLI inspection). Every factual claim cites a URL. Anything not found in a primary source is marked **non vérifié**. No price or feature is invented.

**Established context (not re-proven here):** atopile 0.15.9 is pinned; the layout editor only does move/rotate/flip; `autolayout/` in 0.15.9 calls `api.deeppcb.ai` (proprietary, gateway token, credits); DRC runs via `kicad-cli pcb drc --format json`; Gerbers run via `kicad-cli` — see [atopile-engine.md](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md). The landscape recommendation is pre-routed templates, not a home-grown router — see [landscape.md](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/landscape/docs/research/landscape.md).

---

## 1. Options

| # | Option | What it is | Verdict for 73 h |
|---|--------|------------|------------------|
| A | **Pre-routed templates** | One hand-routed `.kicad_pcb` per board class; re-bind nets via KiCad forward annotation; rip up + retouch only what changed | **Default. Ship this.** |
| B | **DeepPCB (cloud)** | Proprietary learned router, web app + API + KiCad plugin; atopile 0.15.9 reaches it through the atopile gateway | **Opt-in stretch, not the critical path.** Vendor dependency, account + credits, network, review needed |
| C | **freerouting (OSS)** | GPL-3.0 Specctra DSN→SES autorouter with headless CLI, Docker/REST self-host, KiCad DSN/SES round-trip | **Fallback / offline batch router.** Honest quality ceiling, DRC gate mandatory |
| D | **KiCad external (`kicad-cli`)** | DRC + Gerbers + drill + BOM/CPL + 3D exports in batch | **Yes for everything except routing.** There is no batch route command |
| E | **Hybrid (recommended)** | Template-first, freerouting and/or DeepPCB as assisted retouch, honest `unrouted` fallback | **The actual pipeline** |

No OSS batch router inside KiCad was verified. `kicad-cli pcb --help` on KiCad CLI 10.0.7 lists `drc, export, import, render, upgrade` — no routing subcommand (local inspection; command list matches [KiCad CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html)).

---

## 2. Evidence

### 2.1 Pre-routed templates: reuse, netlist change, KiCad re-binding

- **Preferred transfer path is forward annotation, not a netlist file.** "Update PCB from Schematic is the preferred way to transfer design information from the schematic to the PCB. In older versions of KiCad, the equivalent process was to export a netlist […] It is no longer necessary to use a netlist file." — [KiCad docs, forward annotation](https://docs.kicad.org/9.0/en/eeschema/eeschema_schematic_to_pcb.html). Same statement in the board editor docs: [KiCad PCB Editor](https://docs.kicad.org/10.0/en/pcbnew/pcbnew.html).
- **What the tool does:** "adds the footprint for each symbol to the board and transfers updated schematic information to the board. In particular, the board's net connections are updated to match the schematic." Options include re-link by reference designator, replace footprints, delete footprints with no symbols, override locks, sync fields — [KiCad docs](https://docs.kicad.org/9.0/en/eeschema/eeschema_schematic_to_pcb.html). New footprints arrive clustered by sheet and must be placed — [KiCad PCB docs](https://docs.kicad.org/master/en/pcbnew/pcbnew.html).
- **What happens when a component is added/removed:** nets update, footprints are added or (optionally) deleted, but **stale copper is the operator's responsibility**. Community report (not official docs): "KiCad never deletes stale tracks in an Update PCB from Schematic… Tracks and vias are your responsibility to fix up. You might find Tools > Cleanup Tracks & Vias useful." — [KiCad forum](https://forum.kicad.info/t/update-pcb-from-schematic-shouldnt-all-the-unused-parts-be-deleted/67277). Treat as community evidence, not a spec guarantee.
- **Consequence for templates:** one template per board class (power input, MCU, sensor/IO section differ enough that copper cannot be shared). A parametric change (same footprint, same pins, new value) re-binds cleanly. A footprint or pinout change orphans the attached tracks: rip up the affected nets and re-route them (by hand, freerouting, or DeepPCB), then re-run DRC. A connector swap (e.g. USB-C → 12 V barrel + buck) is a new template, not a "retouch".
- **Batch headless re-binding:** **non vérifié**. No `kicad-cli` subcommand for "update PCB from schematic" was found in the [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html) or in local `kicad-cli pcb --help` (drc/export/import/render/upgrade only). The GUI path is `Tools → Update PCB from Schematic (F8)` — [KiCad docs](https://docs.kicad.org/9.0/en/eeschema/eeschema_schematic_to_pcb.html). Scripted re-binding would need `pcbnew` Python or GUI automation, neither verified here.
- **Quality/honesty:** a template is hand-routed quality where untouched, unknown quality where re-touched until DRC + review pass. For the demo, say "pre-routed template re-bound to the new netlist" before any judge asks. Do not imply a model routed the board.

### 2.2 DeepPCB: access, price, API, token, conditions, quality, latency, licence

- **What it is:** learned placement + routing engine; upload a placed board, get back a routed board. "Routes Signal nets to your net classes · diff pairs · blind & buried vias · 8 layers. DRC-clean, runs published in the open." — [DeepPCB KiCad plugin page](https://deeppcb.ai/kicad-autorouter-plugin/).
- **Explicit limits (vendor-stated):** "Very dense boards: completion falls past ~1,200 airwires · free tier 30 min · 4 layers · 150 airwires" and "NOT MODELLED: Length matching & skew · crosstalk · DDR · complicated BGA fanout. Route these in your tools." — [same page](https://deeppcb.ai/kicad-autorouter-plugin/). Standard plan caps: up to 8 layers, 1,200 airwires, 1,000 components, 2,200 pins; free trial 30 min, 1 board, up to 4 layers, 150 airwires, 100 components — [pricing](https://deeppcb.ai/pricing/).
- **Price (list, USD, excl. tax):** 1 hour $30, 10 hours $280, 30 hours $800; enterprise custom, beyond 1,200 airwires / 8 layers contact support — [pricing](https://deeppcb.ai/pricing/).
- **API price:** "for both placement and routing, jobs consume 0.50 AI Credit per minute (or equivalently, 30 AI credits per hour)" and "the same price as the corresponding web app" — [DeepPCB API announcement](https://deeppcb.ai/deeppcb-api-your-pcb-design-ai-agent/).
- **API access:** "After signing up on app.deeppcb.ai, ask us for an API key" / "at app.deeppcb.ai/api-key if you have a DeepPCB account"; integrate via [API reference](https://api.deeppcb.ai/v1/scalar/deeppcb) — [same announcement](https://deeppcb.ai/deeppcb-api-your-pcb-design-ai-agent/). Base URL `https://api.deeppcb.ai/api/v1`; auth header `x-deeppcb-api-key`; board-file upload `POST /api/v1/files/uploads/board-file`, board create `POST /api/v1/boards`, free validation `POST /api/v1/boards/check-board`, start `PATCH /api/v1/boards/{boardId}/confirm` with REQUIRED `timeout`, `maxBatchTimeout`, `jobType` Routing/Placement — per the Scalar reference as surfaced in search excerpts ([reference root](https://api.deeppcb.ai/v1/scalar/deeppcb)). Direct fetch of the Scalar page was not retrievable in this session; field names above are **provisional pending a live API read** (see §6).
- **Usability by a third party with its own key vs atopile gateway:** direct third-party use with its own `x-deeppcb-api-key` is the documented path — [announcement](https://deeppcb.ai/deeppcb-api-your-pcb-design-ai-agent/). The atopile 0.15.9 path is different: `DeepPCBClient(base_url="https://api.deeppcb.ai", headers={"Authorization": f"Bearer {auth_token}"})` in `atopile/autolayout/deeppcb/client.py`, while the token comes from `atopile.agent.gateway.get_token()` → atopile auth token, with `DEEPPCB_BASE_URL = f"{GATEWAY_BASE_URL}/deeppcb"` (`https://gateway.atopile.io/deeppcb`) in `atopile/agent/gateway.py` and `atopile/autolayout/client_factory.py` (local inspection of the `atopile==0.15.9` wheel; package files at [PyPI](https://pypi.org/project/atopile/0.15.9/#files)). So: a third party **can** call DeepPCB directly with its own key, but **cannot** reuse atopile's gateway allowance without an atopile account/token and without patching the client. Gateway credit terms are **non vérifié**.
- **Supported inputs:** "Specctra/DSN, JSON, KiCad, Altium, Zuken and EasyEDA files supported" — [routing-time article](https://deeppcb.ai/pcb-routing-time/).
- **Latency (vendor-measured, 55k+ completed routing jobs, ~30k with clean timing, as of end July 2026):** median router time < 100 airwires **2 min** (p90 10 min); 100–299 **7 min** (p90 31 min); 300–599 **9 min** (p90 45 min); 600+ **16 min** (p90 60 min); all-boards median ~2.5 min, mean ~6.6 min. End-to-end turnaround (setup + review + re-runs + human finishing) is hours: "about three hours plus 30–60 min cleanup vs 2–3 days by hand" (EEJournal-cited, not a platform measurement). Named single runs: 98-airwire 4-layer board 2 min 06 s; 210-connection board with one diff pair 18 min 08 s; 5,208-airwire 8-layer ~14 h — [routing-time article](https://deeppcb.ai/pcb-routing-time/). Additional single-board runs (99.5% in 1 h 51 m; 100% in 40 min 30 s) — [KiCad plugin page](https://deeppcb.ai/kicad-autorouter-plugin/).
- **Quality:** "96.1% mean completion, 100% median completion, 87% of boards reach 95%+ completion, DRC clean by default. Completion here means connectivity… not design quality." "DDR routing, length matching, signal integrity analysis, EM and thermal analysis are outside what the engine does." "Component placement stays your job." — [routing-time article](https://deeppcb.ai/pcb-routing-time/).
- **Privacy/data:** "Press Route and the plugin sends your board, your project rules, and, optionally, your schematic to api.deeppcb.ai… Your designs are private: never shared, never anyone else's example… the plugin is Apache-2.0, and the upload code is public" ([plugin repo](https://github.com/instadeepai/deeppcb-kicad-plugin)) — [plugin page](https://deeppcb.ai/kicad-autorouter-plugin/). Data-retention SLA and sub-processor terms: **non vérifié** (the [privacy/terms page](https://deeppcb.ai/privacy/) is website terms, not a DPA).
- **Licence:** proprietary cloud service by InstaDeep Ltd ("© 2026 InstaDeep Ltd… DeepPCB® is a registered trademark"; "AI to generate outputs… should be reviewed by a qualified professional") — [pricing footer](https://deeppcb.ai/pricing/), [plugin footer](https://deeppcb.ai/kicad-autorouter-plugin/). No OSS licence for the engine was found. The KiCad plugin itself is Apache-2.0 per the vendor claim above; the engine is not self-hostable from published material.
- **Fit for NIA-PCB's class:** a < 50-component 2-layer low-speed board is well under the 150-airwire free tier and the < 100-airwire 2-minute median band, *if* airwire count maps to the demo board (verify by counting). But "DRC-clean by default" is a vendor default claim, not a guarantee for any specific board; the NIA pipeline must re-run `kicad-cli pcb drc` itself.

### 2.3 freerouting: version, licence, CLI headless, KiCad I/O, quality, time, limits, alternatives

- **Project:** "Advanced PCB auto-router… imports `.dsn` files… and exports routed `.ses` session files" — [freerouting/freerouting](https://github.com/freerouting/freerouting). Licence **GNU GPL-3.0** (repo licence field) — [same](https://github.com/freerouting/freerouting); GPL text at [freerouting.org](https://freerouting.org/freerouting/gpl-v3).
- **Version:** v2.5.0 latest, published 2026-10-05; prior v2.4.1 2026-09-03 (`gh release list`) — [releases](https://github.com/freerouting/freerouting/releases). This is days before the 2026-11-04 hackathon deadline: pin the binary.
- **Headless CLI:** `java -jar freerouting.jar -de MyBoard.dsn -do MyBoard.ses [-inc GND,VCC] [-mp 10] [-mt 4]`; `-de` takes `.dsn` (+ optional `.ses`/`.rules`), `-do` writes `.dsn`/`.ses`/`.scr`; `--gui.enabled=false` for headless; layer, optimizer, logging flags documented — [CLI docs](https://github.com/freerouting/freerouting/blob/master/docs/command_line_arguments.md). From v2.5.0, "standalone, native command-line executables compiled via GraalVM Native Image with PGO… eliminating the Java JRE requirement… startup from several seconds to sub-second" for Linux x64, macOS arm64/x64, Windows x64 — [releases](https://github.com/freerouting/freerouting/releases). Java path needs JRE 25+ — [self-hosting](https://github.com/freerouting/freerouting/blob/master/docs/self-hosting.md).
- **KiCad round-trip:** export Specctra DSN from KiCad (`File → Export → Specctra DSN`), route, re-import Specctra Session (`File → Import → Specctra Session`); KiCad 6+; v2.5.0 adds "direct Protocol Buffers IPC integration for KiCad 10+" with fallback to DSN/SES — [releases](https://github.com/freerouting/freerouting/releases), [integrations](https://github.com/freerouting/freerouting/blob/master/docs/integrations.md), [FreeRouting KiCad guide](https://freerouting.org/freerouting/using-with-kicad) (note: freerouting.org documents the older C++ `freeRouting` binary; the DSN/SES idea is the same, the binary is not).
- **Quality (PCBench, 1,157 real-world OSS boards, per-board timeout budgets):** v2.5.0 clean (0 DRC) **868/1,157 (75.0%)** vs v2.4.1 358 (30.9%); fully routed **872/1,157 (75.4%)** vs 600 (51.9%); avg score 969.2 vs 943.9; total corpus 2,997.1 s vs 3,251.0 s; timeouts 78 (6.7%) both versions — [releases](https://github.com/freerouting/freerouting/releases). Routine 2–4-layer Tier B: clean **345/560 (61.6%)** vs 19 (3.4%); fully routed 348/560 (62.1%) — [same](https://github.com/freerouting/freerouting/releases). Read: roughly 1 in 4 boards is *not* fully routed and 1 in 4 is *not* DRC-clean even in the best published run. The pipeline must handle partial routing.
- **DRC report:** `-drc [file]` "writes the design rules check report in KiCad JSON DRC schema format" — [CLI docs](https://github.com/freerouting/freerouting/blob/master/docs/command_line_arguments.md). Still gate on `kicad-cli pcb drc`, not on the router's self-report.
- **Self-host / batch:** Docker `ghcr.io/freerouting/freerouting:latest` (linux/amd64 + linux/arm64), port 37864, `--api_server.enabled=true --gui.enabled=false`, auth-disable flag for local use, `--router.max_passes`, threads default CPU-1, systemd/launchd/NSSM units — [self-hosting](https://github.com/freerouting/freerouting/blob/master/docs/self-hosting.md). Note the doc's `Max routing passes` default (`0` = no limit) vs CLI doc's `--router.max_passes=100` default differ; treat the effective default as **non vérifié** and always pass an explicit cap in batch.
- **Limits for NIA:** freerouting follows DSN layers/nets/rules written by KiCad; copper-pour/power-plane awareness and planeless-power handling are new in v2.5.0 — [releases](https://github.com/freerouting/freerouting/releases). Blind/buried via, length-matching, SI/EMI modelling: no verified support (nothing found in CLI/self-host docs; **non vérifié** — assume absent). Irrelevant for the 2-layer low-speed class, blocking for anything faster.
- **Alternatives (OSS):** no other mature OSS batch autorouter was verified in this session. KiCad's own interactive push-and-shove is not batch (no route subcommand in `kicad-cli`). Experimental LLM/CLI routers exist but are research-grade, not pinned alternatives. So: freerouting is the only verified OSS batch option; everything else is **non vérifié** for production use.

### 2.4 KiCad external: DRC, Gerbers, batch routing

- **DRC:** `kicad-cli pcb drc --format json|report [--all-track-errors] [--schematic-parity] [--units mm|in|mils] [--severity-all|error|warning|exclusions] [--exit-code-violations] [--refill-zones] INPUT_FILE` — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html); local `kicad-cli pcb drc --help` (10.0.7) additionally shows `--save-board` (must combine with `--refill-zones`). JSON output is the machine gate (`--exit-code-violations` returns 5 on violations per docs).
- **Gerbers:** `pcb export gerbers` (one file per layer — "typically the correct command for fabrication") vs `pcb export gerber` (many layers into one file; **deprecated in 9.0, removed in 10.0**) — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html). Options: `--layers`, `--board-plot-params`, `--no-x2`, `--precision 5|6`, etc. — [same](https://docs.kicad.org/9.0/en/cli/cli.html).
- **Drill:** `pcb export drill --format excellon|gerber [--drill-origin] [--excellon-zeros-format] …` — [same](https://docs.kicad.org/9.0/en/cli/cli.html) (local `kicad-cli pcb export --help` confirms `drill`, `gerbers`, `glb`, `step`, `pos` subcommands).
- **BOM/CPL:** schematic BOM `sch export bom --fields … --group-by … --exclude-dnp` (defaults `Reference,Value,Footprint,QUANTITY,DNP`) and board position `pcb export pos --side front|back|both --format ascii|csv|gerber --units` — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html) + local `--help` outputs. atopile also emits `.bom.csv/.bom.json` and `.pick_and_place.csv` via `mfg-data`, but the KiCad path is the independent gate.
- **No batch routing:** as §1 states, no route command exists in `kicad-cli`. Routing stays interactive (PNS), plugin-driven (DeepPCB/freerouting), or external`.

### 2.5 Hybrid: template + retouch, honest `unrouted` fallback, partial results

- **Template + retouch** is the only 73-hour-credible combination: hand-routed base copper + forward-annotation re-bind + rip-up of changed nets + (hand | freerouting | DeepPCB) retouch + `kicad-cli drc` gate. Rationale: templates carry proven geometry; routers carry scale; DRC carries truth.
- **Fallback:** if DRC fails or airwires remain, ship the board as `unrouted/partial` with the DRC JSON and the unrouted-net list attached — never a fake "100% routed" badge. Precedent for naming unrouted nets exists in research agents, but the mechanism here is KiCad DRC + ratsnest, not a model claim.
- **Partial-result handling:** freerouting's own benchmark leaves ~25% of boards not fully routed / not clean (§2.3); DeepPCB's mean completion is 96.1%, i.e. some boards are partial by design (§2.2). The pipeline must therefore treat "routed" as a measured property (DRC exit code + unconnected count), not as a router exit code.

### 2.6 Pipeline: placement → routing → DRC → Gerbers → BOM/CPL; breakage, CPU/RAM, parallelism

- **Order:** `ato build` (nets/footprints into `.kicad_pcb`) → place (template positions or editor) → route (template copper + retouch) → `kicad-cli pcb drc --format json` → `export gerbers + drill` → `sch export bom + pcb export pos` → `export glb/step` + `render`. atopile's own `mfg-data` bundle (Gerber zip, DXF, pick-and-place, testpoints, `pcb_summary.json`) exists, but the KiCad CLI path is the verifiable gate — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html), [atopile-engine context](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md).
- **Where it breaks:** (1) parts/picking (auth-gated registry — biggest blocker per atopile-engine.md); (2) netlist re-bind (stale copper, §2.1); (3) routing completion (< 100% or DRC violations, §§2.2–2.3); (4) footprint/3D-model gaps (empty STEP/GLB bodies); (5) KiCad 9-vs-10 file fidelity (atopile writes KiCad 9 `version 20241229`; local toolchain is 10.0.7 — round-trip **non vérifié**).
- **CPU/RAM/parallelism (prior measurements, 4 vCPU / 7.6 GB host, no-parts boards):** empty `default` build ~11–13 s wall (~7 s CLI startup); 400-module `default` 46 s wall / 766 MB RSS; `3d-image` 12–20 s; `glb` ~1 s; `mfg-data` ~3–13 s — [atopile-engine.md](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md). freerouting corpus mean ≈ 2.6 s/board (2,997 s / 1,157) under per-board timeouts with 6.7% timeouts — [releases](https://github.com/freerouting/freerouting/releases). DeepPCB medians §2.2 (minutes, cloud). Parallelism: atopile `--jobs` defaults to CPU count, builds serialised per project (lock the `.kicad_pcb`); freerouting threads default CPU-1, cap with `-mt`/`--router.max_threads` — [CLI docs](https://github.com/freerouting/freerouting/blob/master/docs/command_line_arguments.md), [self-hosting](https://github.com/freerouting/freerouting/blob/master/docs/self-hosting.md). Real-board numbers with parts/3D models: **non vérifié**.

### 2.7 3D: GLB/STEP on the routed board, how long

- **Commands:** `kicad-cli pcb export glb` and `pcb export step` (plus `brep/ply/stl/u3d/vrml`, `pcb render` to PNG/JPEG) with `--include-tracks --include-pads --include-zones …` ("time consuming" per docs) — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html); local `render --help` defaults 1600×900, `--quality basic|high`, `--side top|bottom|…`.
- **Time:** empty-board GLB ~1 s, PNG 12–20 s (prior measurement) — [atopile-engine.md](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md). Real routed board with 3D models: **non vérifié** (STEP with `--include-tracks` is documented "time consuming" but not timed here). Recommendation: GLB on every build (cheap), PNG on demand, STEP on download.
- **Fidelity caveat:** 3D quality follows the parts catalogue (EasyEDA/LCSC STEP models); parts without models leave holes — [atopile-engine.md](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md). A routed board with missing models still exports; it just looks incomplete.

### 2.8 Demo honesty: what "fabricable" may mean, which proofs to show

- **May say:** "compiles (`ato build` green), DRC-clean per attached `kicad-cli pcb drc --format json` report, Gerber/drill/BOM/CPL open in KiCad/Gerber viewer, 3D GLB shown, receipt (hashes + timestamps) bound to the exact exported files." Each is independently openable.
- **Must say:** "layout is a pre-routed template re-bound to the new netlist; retouch by [hand/freerouting/DeepPCB]; not a model-routed board." DeepPCB's own footer requires the same posture: "AI to generate outputs… should be reviewed by a qualified professional before being relied upon or implemented" — [pricing footer](https://deeppcb.ai/pricing/).
- **Must not say:** "production-ready", "auto-routed", "high-speed/RF-capable", or any respin-cost claim. No reliable public respin-cost figure was established (see landscape method note) — [landscape.md](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/landscape/docs/research/landscape.md).
- **Proofs on screen:** exit code + DRC JSON (0 violations), Gerber/drill file list, BOM/CPL row counts, GLB render, and the receipt file. One deliberate failure + repair (constraint or DRC) is stronger than a clean first run.

---

## 3. Recommendation (73-hour hackathon)

1. **Default to templates (Option A).** One hand-routed 2-layer template for the single demo class (e.g. ESP32-C3 + USB-C + I²C sensor + 3.3 V LDO). `ato` owns nets/footprints; the template owns copper. Re-bind via Update-PCB-from-Schematic semantics; rip up only changed nets.
2. **Wire `kicad-cli` as the only gate (Option D).** `drc --format json --exit-code-violations` blocks Gerber export. `export gerbers + drill + pos` and `sch export bom` produce the fab package. No green DRC → no "fabricable" claim.
3. **Add freerouting as the offline retouch (Option C).** DSN→SES round-trip for changed nets only (`-inc` the power nets you hand-route), pinned v2.5.0 binary or Docker, explicit `-mp`/`-mt` caps, always followed by the DRC gate. Zero network, zero credits, reproducible.
4. **Keep DeepPCB as an opt-in stretch (Option B), never the critical path.** If used: third-party key via `app.deeppcb.ai/api-key`, direct `x-deeppcb-api-key` calls (not the atopile gateway token), smallest board class (< 100 airwires ≈ 2 min median), and the same DRC gate afterwards. Budget demo time in minutes, not seconds, and keep the freerouting path as fallback.
5. **Ship the hybrid (Option E) with an honest fallback.** DRC-clean → "routed, DRC-clean". Otherwise → "partial, N unrouted, DRC JSON attached". The fallback is a feature (it proves the gate works).

---

## 4. Latency budget (demo class, < 50 comps, 2 layers)

| Stage | Budget | Source / basis |
|-------|--------|----------------|
| `ato build` (empty–small, warm) | 10–20 s wall | prior 11–13 s empty + ~7 s startup — [atopile-engine](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md) |
| Template re-bind + place | 1–5 min (human) | GUI F8 flow — [KiCad docs](https://docs.kicad.org/9.0/en/eeschema/eeschema_schematic_to_pcb.html); headless **non vérifié** |
| freerouting retouch (small board) | 0.5–5 min | corpus mean ~2.6 s/board but small-board variance + optimisation tail; timeouts 6.7% — [releases](https://github.com/freerouting/freerouting/releases); NIA-class timing **non vérifié** |
| DeepPCB retouch (optional, < 100 airwires) | median 2 min, p90 10 min | vendor platform stats — [routing-time](https://deeppcb.ai/pcb-routing-time/) |
| `kicad-cli pcb drc --format json` | seconds | CLI gate — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html); board-specific time **non vérifié** |
| `export gerbers + drill + bom + pos` | seconds–1 min | same — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html) |
| `export glb` | ~seconds | ~1 s empty — [atopile-engine](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md) |
| `render` PNG / `export step` | PNG 12–20 s empty; STEP variable | PNG measured — [same](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md); routed-board STEP **non vérifié** |
| **Total credible demo path (template + freerouting + gates)** | **~5–12 min machine + human place time** | sum of above; precompute the happy path, cache artifacts, keep a recorded fallback |

Do not promise "under 3 minutes" for the full route+gate loop on stage; promise "one sentence → fab package + receipt" with the happy path precomputed.

---

## 5. Failure modes

| # | Breaks where | Symptom | Mitigation |
|---|--------------|---------|------------|
| F1 | Parts registry / picking | `ato build` fails before layout (auth) | Pre-picked/curated BOM only; `--keep-picked-parts` style freeze |
| F2 | Netlist re-bind | New footprint floats at cursor; stale tracks short/open | Place new parts; rip up + `Cleanup Tracks & Vias`; re-route affected nets only |
| F3 | Router incompleteness | Unrouted airwires remain (freerouting ~25% not fully routed; DeepPCB mean 96.1%) | Publish unrouted list; hand-finish or narrow scope; never hide it — [releases](https://github.com/freerouting/freerouting/releases), [routing-time](https://deeppcb.ai/pcb-routing-time/) |
| F4 | DRC violations | `drc` exit 5 / JSON violations | Fix clearance/track/via per report; re-run; attach report to receipt |
| F5 | DSN/SES fidelity | Import drops rules/zones | Diff layer stack + net classes before/after; keep KiCad 9↔10 round-trip out of the live demo |
| F6 | Cloud dependency (DeepPCB) | No network / no credits / queue tail (p90 10–60 min) | freerouting fallback; pre-routed golden backup; recorded video |
| F7 | 3D gaps | Missing STEP models → hollow GLB | Ship GLB anyway + note missing models; STEP on download only |
| F8 | Version skew | atopile writes KiCad 9 (`20241229`), runner is 10.0.7 | Pin both; test the exact pair before the demo; keep a known-good `.kicad_pcb` |
| F9 | Time overrun | Optimiser tail / cloud queue eats the slot | Explicit `-mp`/`-mt` caps; DeepPCB `timeout`/`maxBatchTimeout`; stop-and-ship-partial with receipt |

---

## 6. Non vérifié (do not use in claims)

- DeepPCB Scalar field names/defaults beyond what the announcement + excerpts state (direct page fetch failed); confirm `timeout`, `maxBatchTimeout`, `smartConstraints`, credit metering against a live key before coding.
- Whether the DeepPCB free trial (30 min / 150 airwires / 4 layers) covers the exact demo board including re-runs; and whether gateway credits (atopile account) are usable/transferable by a third party.
- DeepPCB data-retention/DPA terms beyond "private, never shared" (marketing) — [plugin page](https://deeppcb.ai/kicad-autorouter-plugin/).
- freerouting quality/time on the *specific* NIA template (< 50 comps, 2-layer, low-speed) — only corpus aggregates verified ([releases](https://github.com/freerouting/freerouting/releases)).
- Headless "Update PCB from Schematic" (no `kicad-cli` path found) — [CLI docs](https://docs.kicad.org/9.0/en/cli/cli.html).
- `kicad-cli` DRC/Gerber/STEP wall time on the real demo board (only empty-board timings measured).
- KiCad 9-authored (atopile) ↔ KiCad 10.0.7 round-trip fidelity.
- atopile 0.15.9 exact source commit (public repo HEAD is 2026-03-11, PyPI 0.15.9 is 2026-09-12) — [atopile-engine](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md).
- Any "cost per respin", fab yield, or "first/auto-routed" claim — no primary source established.
- freerouting `--router.max_passes` effective default (CLI doc vs self-host doc disagree) — [CLI](https://github.com/freerouting/freerouting/blob/master/docs/command_line_arguments.md) vs [self-host](https://github.com/freerouting/freerouting/blob/master/docs/self-hosting.md); always pass an explicit cap.

---

## 7. Primary sources

- Pricing/limits: <https://deeppcb.ai/pricing/>
- API (price, key, integration): <https://deeppcb.ai/deeppcb-api-your-pcb-design-ai-agent/>
- API reference root: <https://api.deeppcb.ai/v1/scalar/deeppcb>
- Routing-time dataset (55k jobs): <https://deeppcb.ai/pcb-routing-time/>
- KiCad plugin (scope, ceiling, privacy, benchmarks): <https://deeppcb.ai/kicad-autorouter-plugin/>
- Terms/privacy (website terms): <https://deeppcb.ai/privacy/>
- Help (boards, constraints, DRC-clean claim): <https://deeppcb.ai/help/>
- freerouting repo (GPL-3.0, DSN→SES): <https://github.com/freerouting/freerouting>
- freerouting releases (v2.5.0 notes + PCBench 1,157-board numbers): <https://github.com/freerouting/freerouting/releases>
- freerouting CLI: <https://github.com/freerouting/freerouting/blob/master/docs/command_line_arguments.md>
- freerouting self-host: <https://github.com/freerouting/freerouting/blob/master/docs/self-hosting.md>
- freerouting integrations (KiCad DSN/SES): <https://github.com/freerouting/freerouting/blob/master/docs/integrations.md>
- GPL text: <https://freerouting.org/freerouting/gpl-v3>
- KiCad CLI (DRC/Gerbers/drill/BOM/pos/GLB/STEP/render): <https://docs.kicad.org/9.0/en/cli/cli.html>
- Forward annotation: <https://docs.kicad.org/9.0/en/eeschema/eeschema_schematic_to_pcb.html>
- Board editor (import flow): <https://docs.kicad.org/10.0/en/pcbnew/pcbnew.html>
- Stale-tracks behaviour (community): <https://forum.kicad.info/t/update-pcb-from-schematic-shouldnt-all-the-unused-parts-be-deleted/67277>
- atopile 0.15.9 package (autolayout/gateway inspection basis): <https://pypi.org/project/atopile/0.15.9/#files>
- DeepPCB KiCad plugin code (Apache-2.0 upload claim basis): <https://github.com/instadeepai/deeppcb-kicad-plugin>
- Prior context: [atopile-engine](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/atopile-engine/docs/research/atopile-engine.md) · [landscape](https://raw.githubusercontent.com/Marwichmisi/NIA-PCB/research/landscape/docs/research/landscape.md)
