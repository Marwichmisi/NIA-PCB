# Parts catalog without the atopile registry

**Status:** research deliverable for NIA-PCB (hackathon, 73 h). Product scope: one board class only — 2 layers, low speed, 5 V/3.3 V, < 50 placements, no RF/BGA, everything in stock.
**Pinned artifact:** `atopile==0.15.9` (PyPI, 2026-09-12; `requires-python: <3.15,>=3.14`; cp314 wheels only, no Linux ARM64 wheel — https://pypi.org/pypi/atopile/0.15.9/json).
**Bottom line:** never call the registry at runtime. Pre-pick once with an account, vendor `parts/` (+ `packages/` equivalents) into the repo, and design against vendored **atomic parts** that already carry `has_part_picked`. A no-account, no-network build of such a design was verified working (section 2).

---

## 1. Options

| # | Option | Verdict |
|---|---|---|
| A | Use what ships in `atopile==0.15.9` (examples' `parts/`) | Works offline today, but only ~10 passives/LEDs. Seed, not catalog. |
| B | **Pre-pick once, vendor `parts/`, design against atomic parts** | **Recommended.** Zero registry calls at runtime (proven §2). |
| C | `--keep-picked-parts` from an existing `.kicad_pcb` | Does NOT avoid the registry (proven at code level §3). Reject as the runtime strategy. |
| D | JLCPCB public search + `easyeda2kicad` at runtime | Fragile (spoofed UA, IP rotation, hardcoded `is_basic=False`); fine as a *builder-side helper*, not in the product path. |
| E | Registry shim (`/v0/component/...`, `/v0/query`) + injected token | Feasible in hours, unsupported, brittle. Backup plan only (§6). |
| F | Replace the picker | Out of scope for 73 h (§7). |
| G | Official JLCPCB Components API (`api.jlcpcb.com`) | The compliant long-term source for stock/price/spec data (application + review required). |

---

## 2. Evidence

### 2.1 What ships in `atopile==0.15.9`

- The sdist `examples/` ships ready-to-vendor atomic parts, e.g. `esp32_minimal/parts/` with 8 parts, each a directory `<MFR>_<MPN>/` containing `<part>.ato` + `.kicad_mod` + `.kicad_sym` + `.step` + datasheet PDF (inspected in `/tmp/opencode/atopile-0.15.9/examples/esp32_minimal/parts/`; same content in the wheel, `atopile-0.15.9.tar.gz` on https://pypi.org/project/atopile/0.15.9/#files).
- Example catalog IDs (all real, re-verified §2.5): `C1525` (Samsung CL05B104KO5NNNC 100 nF 0402), `C25744` (Uniroyal 0402WGF1002TCE 10 kΩ), `C25804` (0603WAF1002T5E 10 kΩ), `C52923` (CL05A105KA5NQNC 1 µF 0402), `C59461` (CL10A226MQ8NRNC 22 µF 0603), `C2479705` (KOA RK73H1ETTP1821F 1.82 kΩ), `C21190` (0603WAF1001T5E 1 kΩ), `C2286` (Kento KT-0603R LED).
- Each `.ato` carries `trait has_part_picked::by_supplier<supplier_id="lcsc", supplier_partno="C…", manufacturer="…", partno="…">` plus `is_atomic_part` (footprint/symbol/model filenames) and `is_auto_generated` with a `source="easyeda:C…"` checksum (e.g. `Samsung_Electro_Mechanics_CL05B104KO5NNNC.ato`).
- Counter-examples of what does NOT ship: `esp32_minimal` needs registry packages (`atopile/buttons`, `atopile/ti-tlv75901`, `atopile/usb-connectors`, `atopile/espressif-esp32-c3` in its `ato.yaml` `dependencies:`) resolved via `https://packages.atopileapi.com` — that host no longer resolves (DNS `getent` fails, 2026-10-27). `esp32_minimal` therefore cannot build as-is.

### 2.2 A no-account build works if the design instantiates atomic parts directly — verified end to end

Experiment (2026-10-27, `atopile==0.15.9` venv, Python 3.14.7, no `ato auth login`, no token in `~/.config/atopile/`):

1. Copied two example `parts/` dirs (`C1525`, `C25744`) into a scratch project, wrote `main.ato` with `from "parts/…/….ato" import …_package` and `cap = new Samsung_…_package`, connected via pin syntax `power.hv ~ cap.1` (pin-reference grammar: `AtoParser.g4` `pin_reference_end: DOT number_hint_natural` — https://github.com/atopile/atopile/blob/619eda7f777558a3e500dbad9cc2941712881495/src/atopile/compiler/parser/AtoParser.g4).
2. `ato build` → **"Build successful! 🚀"**, `Picking parts [0.17s]`, BOM emitted with `LCSC Part #` C1525/C25744, no sign-in error, repeat build identical.

Why it works (code, installed wheel `faebryk/libs/picker/picker.py`): `get_pick_tree` (line 113) returns an empty tree for any module carrying `F.Pickable.has_part_picked` (line 133); `pick_topologically` (line 583) then has zero modules, calls no API. Atomic parts carry the pick as source data, and `is_atomic_part` (`src/faebryk/library/is_atomic_part.py`) resolves footprint/symbol/model to **local files** — the EasyEDA download path (`lcsc.py:524 get_raw`, `:573 download_easyeda_info`, `:601 attach`) is never entered.

### 2.3 `--keep-picked-parts` does NOT avoid the registry — code-level

- `atopile/build_steps.py:687 pick_parts`: with `keep_picked_parts` (CLI flag documented in `ato build --help`, defaulting from `frozen` per `atopile/config.py:622`), it calls `load_part_info_from_pcb` (`faebryk/libs/app/keep_picked_parts.py:136`).
- That function restores **`is_pickable_by_supplier_id`** traits (LCSC constraints), *not* `has_part_picked` — so `get_pick_tree` stays non-empty.
- `pick_topologically` then calls `_find_modules` → `client.fetch_parts_multiple(queries)` (`faebryk/libs/picker/api/picker_lib.py:248,268`; module-level `client = get_api_client()` at line 44; `ApiClient._post` → `_headers` → `_cfg`, which raises `UserException("Part picking on atopile 0.15.8 requires sign-in…")` without a token (`faebryk/libs/picker/api/api.py:62`).
- Same wall on the footprint path: `lcsc.py:561-564 get_raw` calls `get_api_client().fetch_part_by_lcsc(...)` **before** touching EasyEDA, so even warm-cache-free fresh picks of LCSC-pinned parts need a token. Measured: `download_easyeda_info('C1525')` without a token raises `UserException: … requires sign-in` — the public EasyEDA API is never reached.
- Consequence: keep-picked is useful to *preserve the choice* of part across rebuilds on a builder machine that has an account, but it is not an offline/runtime strategy. (End-to-end keep-picked build with a pre-picked PCB not measured — creating one requires an account; the call chain above is unambiguous.)

### 2.4 Registry state — measured 2026-10-27

- `components.atopileapi.com`, `packages.atopileapi.com`: DNS does not resolve (`getent hosts` rc=2).
- `legacy.atopileapi.com` (default `services.components.url`, `atopile/config.py:805`): resolves; `GET /v0/component/lcsc/1525` → **401**, `GET /v0/motd` → 200.
- Sign-in is Clerk OAuth in the browser (`atopile/auth/session.py:21 CLERK_ISSUER=https://clerk.atopile.io`, `atopile/auth/runtime.py:7 GATEWAY_BASE_URL=https://gateway.atopile.io`), token in system keyring or `~/.config/atopile/clerk_oauth_session.json` — no environment-variable override exists. atopile confirms the policy: "0.15.8 … requires you to sign in before picking parts. After installing 0.15.8, run `ato auth login`" (https://atopile.io/blog/atopile-v16).

### 2.5 JLCPCB public search is live and usable today (builder-side)

Module `atopile/model/parts_search_jlc.py` (228 lines): POSTs to `https://jlcpcb.com/api/overseas-pcb-order/v1/shoppingCart/smtGood/selectSmtComponentList/v2` with a Chrome UA, self-throttled to 5 rps. Caveats read in the code: `is_basic`/`is_preferred` are **hardcoded `False`** (lines 168–169) — assembly surcharge planning cannot rely on this source; prices are a bare `componentPrices` passthrough. Live queries on 2026-10-27 returned real rows (stock + `unit_cost`), e.g. `C1525` stock 34,450,748 @ 0.0035; `C25744` stock 18,213,013 @ 0.0024; `C2838502` (ESP32-C3-MINI-1-N4) stock 13,192 @ 3.0511.

### 2.6 EasyEDA footprints + 3D models exist for every catalog line

Via `EasyedaApi().get_cad_data_of_component` (public EasyEDA API, no token) on 2026-10-27: `C2838502` → `WIFIM-SMD_ESP32-C3-MINI-1` + 3D same name; `C393939` → `USB-C-SMD_TYPE-C16PIN` + 3D; `C77361` → `DFN-8_L2.5-W2.5-P0.50-BL-EP` + 3D `DFN-8_L2.5-W2.5-H1.0-P0.50`; `C6186` → `SOT-223-3_…` + 3D. Passives/LED STEP files ship in the examples (`C0402_L1.0-W0.5-H0.5.step`, `R0402_L1.0-W0.5-H0.4.step`, `KT-0603R-YELLOW.STEP` — note: yellow variant; red C2286 3D model name unverified, same footprint family).

---

## 3. Recommended catalog (demo board: ESP32-C3 + USB-C + I²C sensor + LDO)

Sensor board, 2 layers, 5 V (USB-C) → 3.3 V (LDO) → ESP32-C3-MINI + SHT30 + 2 LEDs. **30 placements, 12 unique refs**, all LCSC IDs + stock + `unit_cost` measured via the public JLC search on 2026-10-27 (overseas storefront; re-check at order time; prices indicative).

| Qty | Designator | Value | LCSC | Mfr / MPN | Package | Stock | Unit |
|---:|---|---|---|---|---|---|---|
| 1 | U1 | ESP32-C3-MINI-1-N4 | C2838502 | Espressif ESP32-C3-MINI-1-N4 | SMD 16.6×13.2 | 13,192 | 3.0511 |
| 1 | J1 | USB-C 16P | C393939 | SHOU HAN TYPE-C16PIN | SMD | 416,253 | 0.0691 |
| 1 | U2 | SHT30 temp/RH | C77361 | Sensirion SHT30-DIS-B2.5KS | DFN-8-EP 2.5×2.5 | 19,716 | 2.8088 |
| 1 | VR1 | AMS1117-3.3 | C6186 | AMS AMS1117-3.3 | SOT-223 | 1,918,631 | 0.2061 |
| 2 | D1–D2 | LED red 0603 | C2286 | Hubei Kento KT-0603R | 0603 | 7,681,249 | 0.0077 |
| 2 | R5–R6 | 1 kΩ | C21190 | Uniroyal 0603WAF1001T5E | 0603 | 33,949,081 | 0.0017 |
| 14 | C1–C14 | 100 nF | C1525 | Samsung CL05B104KO5NNNC | 0402 | 34,450,748 | 0.0035 |
| 2 | C15–C16 | 1 µF | C52923 | Samsung CL05A105KA5NQNC | 0402 | 11,312,402 | 0.0064 |
| 2 | C17–C18 | 22 µF | C59461 | Samsung CL10A226MQ8NRNC | 0603 | 3,261,016 | 0.0313 |
| 2 | R1–R2 | 10 kΩ I²C pull-ups | C25744 | Uniroyal 0402WGF1002TCE | 0402 | 18,213,013 | 0.0024 |
| 2 | R3–R4 | 10 kΩ EN/BOOT strap | C25804 | Uniroyal 0603WAF1002T5E | 0603 | 39,190,047 | 0.0017 |

Indicative BOM total ≈ **$6.29** (sum of `unit_cost` × qty; excludes PCB, assembly, fees).

Justification per choice: module variant N4 (4 MB flash, no external flash/crystal/antenna needed — keeps placement count < 50); fixed-output AMS1117 (C6186: original mfr, 1.9 M stock) instead of adjustable (fewer resistors, 2-layer friendly); SHT30-DIS-B over cheaper clones (Sensirion datasheet, verified EasyEDA DFN footprint + 3D); single 0402 100 nF reel ×14 (one JLC line item); Samsung MLCCs (stock > 3 M each, STEP models ship in examples); Uniroyal thick-film (stock > 18 M); J1 16-pin Type-C with 416 k stock (avoid 6-pin variants that lack USB 2.0 D+/D− mid-mount options).

---

## 4. Pre-pick procedure (builder machine has the account; product never does)

**Do once per part (or per catalog refresh), on a builder laptop/CI secret host:**

1. `pip install atopile==0.15.9` (Python 3.14; x86_64 Linux OK, no ARM64 wheel — build the image `FROM --platform=linux/amd64` or from the sdist with Zig).
2. `ato auth login` (browser Clerk flow; token lands in keyring or `~/.config/atopile/clerk_oauth_session.json`). Optional: `CI=1` or `telemetry.yaml: {telemetry: false}` to silence PostHog (`atopile/telemetry`, `~/.config/atopile/`).
3. For each LCSC ID: run a minimal pick (generic `Resistor`/`Capacitor` with constraints, or explicit `x.lcsc_id = "C…"`) and `ato build`. The picker writes `parts/<MFR>_<MPN>/<part>.ato` (`has_part_picked::by_supplier` + `is_atomic_part` + checksum) plus `.kicad_mod`/`.kicad_sym`/`.step`/datasheet PDF (`faebryk/libs/part_lifecycle.py`, `faebryk/libs/picker/lcsc.py:601 attach`).
4. Review each part once (footprint vs datasheet — `easyeda2kicad` output correctness is not guaranteed, https://github.com/uPesy/easyeda2kicad.py; check `is_basic` on jlcpcb.com manually since search hardcodes it False), then commit `parts/` to the repo. To freeze a part against regeneration, keep the `is_auto_generated` trait (checksum) untouched; to hand-edit, remove the trait (file header says so; `PartIsNotAutoGenerated`, `part_lifecycle.py:63`).

**Vendor registry packages the same way:** `ato add` supports `file://<path>` and `git://<repo>.git[@ref][:path]` (`atopile/cli/install.py:145-146`, `atopile/config.py` `FileDependencySpec`/`GitDependencySpec`) — mirror each needed `atopile/*` package into the repo (or a private git host) and rewrite `ato.yaml` `dependencies:` from `registry://` to `file://`/`git://`. What cannot be vendored this way: new-part *search* (stays builder-side via JLC search or the official API).

**Product runtime (no account, no registry, no network needed):**

1. Image: Python 3.14 + `atopile==0.15.9` + KiCad CLI (3D/gerbers only); block/omit registry egress; no token baked in.
2. Design sources import atomic parts from vendored `parts/` (`from "parts/…/….ato" import …_package`, connect raw pins as `part.1`, `part.2`).
3. `ato build` (no `--keep-picked-parts` needed — picks are source data). Verified: full `default` target incl. BOM/manifest/pinout/datasheets in ~3 s with zero registry calls (§2.2).

---

## 5. Licensing

- atopile source + PyPI metadata: **MIT** (https://github.com/atopile/atopile/blob/main/LICENSE, https://pypi.org/pypi/atopile/0.15.9/json). Keeps the notice in redistributed copies.
- `atopile-easyeda2kicad` (declared dependency, `Requires-Dist: atopile-easyeda2kicad>=0.9.9`): **AGPL-3.0** (PyPI metadata; upstream https://github.com/uPesy/easyeda2kicad.py, "AGPL-3.0 license"). Imported on every build via the picker stack — network-service use triggers AGPL §13 source obligations; get counsel before launch. This is the sharpest license edge in the stack.
- KiCad (`kicad-cli`, invoked as a subprocess): program **GPLv3+**, libraries **CC-BY-SA 4.0** (https://www.kicad.org/about/licenses/). Keep the process boundary; GPL source/notice duties apply to the distributed image for KiCad itself.
- JLC search path: the atopile module calls itself "hacky" and spoofs a browser UA with IP fallback (`parts_search_jlc.py:1,64-100`) — no contract, breaks without notice, subject to https://jlcpcb.com/help/article/terms-%26-conditions. Compliant alternative: official Components API at https://api.jlcpcb.com/ (account + application, reviewed on order history).
- Vendored `parts/` content (EasyEDA footprints/symbols/STEP, LCSC datasheet PDFs, registry attribute blobs): redistribution terms **non vérifié** (see §7). The fact that atopile's own examples ship LCSC datasheet PDFs is precedent, not permission.

*Not legal advice.*

---

## 6. Options E–G in brief

**E — Registry shim.** Implement `POST /v0/query`, `GET /v0/component/lcsc/{id}`, `GET /v0/component/mfr/{m}/{p}` (response shapes in `faebryk/libs/picker/api/api.py:139-174`, `models.py`) and point `ato.yaml` at it (`services: {components: {url: …}}`, `atopile/config.py:940`, overridable per project; packages URL at `config.py:809`). Still need *a* token: inject via `atopile.auth.runtime.set_auth_token` (sitecustomize patch) or a pre-seeded `clerk_oauth_session.json` — no env override exists. Cost: hours-to-days for the shim; benefit: unblocks *generic* (`Resistor` + constraints) picking offline. Risk: response-schema drift with each 0.15.x rebuild, and you become the parts-data owner anyway. Only worth it if the product must accept arbitrary LCSC IDs at runtime — it should not in 73 h.

**F — Replace the picker.** The picker is ~3.6 kLOC across `picker.py` (677), `api/picker_lib.py` (384), `api/api.py` (179), `api/models.py` (300), `lcsc.py` (1071), `part_lifecycle.py` (772) plus solver coupling — weeks, not hours. Reject.

**G — Official JLC API.** `api.jlcpcb.com` Components API gives real pricing/stock/specs under a contract; needs an account and approved application. Right call post-hackathon for the *catalog-refresh* job; not needed for the runtime path.

---

## 7. Non vérifié

- Whether a free atopile account has rate limits / CGU clauses restricting automated pre-pick or redistribution of `parts/` data (no public component-API ToS found; app ToS lives behind sign-in).
- EasyEDA/LCSC redistribution terms for vendored footprints/symbols/3D/datasheets (pointers only: https://easyeda.com/page/legal, https://www.lcsc.com).
- JLC `unit_cost` currency (assumed USD, overseas storefront) and `is_basic` status of catalog rows (search returns no flag).
- C2286 red-LED 3D model (examples ship the yellow sibling's STEP).
- USB-C C393939 mating/assembly yield at JLC (stock 416 k, footprint verified, no DFM run done).
- CC pull-down (2× 5.1 kΩ) and EN/BOOT buttons (2× 6×6 mm SMD): needed for a shippable board but no LCSC ref verified — pick during pre-pick (§4), do not invent refs.
- `--keep-picked-parts` end-to-end offline behavior (code chain says registry still required; not measured — no account to create the fixture).
- KiCad 10-authored `.kicad_pcb` round-trip through atopile's KiCad-9-pinned writer.
