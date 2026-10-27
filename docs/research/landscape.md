# Text → PCB: competitive landscape and NIA-PCB's wedge

> **Research date:** 27 October 2026. **Scope:** products that turn natural language, requirements, or high-level intent into a PCB design, plus tools that automate one stage of the chain. **Evidence rule:** every factual claim links to a primary source (vendor site, documentation, pricing page, GitHub, or paper). Vendor claims are labelled as claims. Missing or stale information is marked **Non vérifié**. Prices are published list prices in USD, excluding tax, on the research date.

**One-sentence thesis:** generation is commoditized; the frontier is a verifiable, accessible path from intent to fabrication — and that is the wedge for NIA-PCB.

---

## Executive summary

1. **The category is crowded and no longer novel.** At least a dozen products accept natural language, code, or high-level intent and emit a schematic, a board, or both: Flux, Cherry Blossom, Trace, Diode, Copperhead, Ada, JITX, Circuit Mind, Celus, Quilter, DeepPCB, EasyEDA + JLCPCB, tscircuit, and Cadence. "Prompt to PCB" is a feature, not a moat.

2. **Generation is commoditized; verification is the differentiator.** The strongest recent entrants sell checks, gates, and review — not just output. Copperhead explicitly says it is "not an autorouter" and gates on ERC/DRC; Ada names unrouted nets and produces an adversarial review; Diode says its schematic and layout are "written as code" and checked by an AI agent and engineers; Circuit Mind sells a "verified schematic" and stops before layout.

3. **Free/open + BYOK is already table stakes.** Atopile (MIT), tscircuit (MIT), Copperhead (Apache-2.0 CLI), Ada (MIT), and EasyEDA + JLCPCB (free editor + bring-your-own coding agent) all undercut a paid wrapper. A new product cannot win on "free" or "bring your own key" alone.

4. **Atopile already has an agent.** Its repository ships a sidebar-agent skill. The compiler, assertions, parametric parts, and `.ato` source are the real asset. NIA-PCB should be the front door to that asset, not a replacement for it.

5. **The defensible wedge is the combination:** French-first, browser-based, atopile-native, and a receipt that binds every output to a check. Not a better model, not a parts library, not a router.

6. **A solo in 73 hours can credibly ship one narrow board class** (2/4-layer, low-speed, off-the-shelf parts), one real repair loop, one conversational redesign with a diff, and one verifiable fab handoff. A general-purpose board generator is not a 73-hour project.

7. **Impact numbers exist but must be used carefully:** about an $85B PCB market in 2025 (Prismark), 297,900 US electrical/electronics engineering jobs at a $125,040 median (BLS), and 396M French speakers (OIF). No reliable public "cost per respin" figure was found; do not invent one.

> **What this means for the 3-minute demo:** **no Gerber without a receipt.** Show a constraint that bites, a conversational redesign with an inspectable diff, and a fab package bound to checks. Every claim must be visible on screen, not asserted on a slide.

---

## 1. Direct products at a glance

Prices are list prices on 27 October 2026, excluding tax; annual and usage terms are in the notes. "Stage" is the vendor's own framing where possible; "Product" means generally available, "Early" means beta/preview.

| Product | Input | What it outputs | Price (list) | Stage | Honest limit |
|---|---|---|---|---|---|
| **[Flux](https://www.flux.ai/)** | Prompt/chat in a browser editor | Schematic, PCB, Gerber/BOM/CPL, 3D | [Explore $50/mo; Build $100; Pro $250; Teams $500/mo](https://www.flux.ai/p/pricing) | Product | 8 layers max; closed; credit-based AI |
| **[Cherry Blossom](https://www.trycherryblossom.com/)** | Prompt | Schematic, PCB, 3D, Gerbers, BOM/CPL, KiCad/STEP | [Free editor; Pro $20/mo; Ultra $140/mo](https://www.trycherryblossom.com/pricing) | Early beta | No import, no manual layout, no simulation; not high-speed/RF |
| **[Trace](https://buildwithtrace.com/)** | Prompt/chat in a desktop app | Schematic, placement, routing, DRC, Gerber/BOM/CPL | [Free editor; Pro $29.99/mo; Ultra $169.99/mo; PAYG $20](https://buildwithtrace.com/pricing) | Product | macOS desktop; closed; AI credits |
| **[Diode](https://www.diode.computer/)** | Requirements/chat + fabrication | Schematic and layout "as code", checks, fab/assembly quote | [Enterprise from $10,000/design + fab](https://www.diode.computer/) | Product (enterprise) | Build app login-gated; public detail limited |
| **[Copperhead](https://copperhead.sh/)** | Spec/chat in a CLI | Schematic, first-draft layout, ERC/DRC gates, Gerbers/STEP | [CLI free; Cloud $49/user/mo; Team $49/user + $199/mo](https://copperhead.sh/pricing/) | Early | Explicitly not an autorouter |
| **[Ada / Silkscreen](https://github.com/machmoon/Ada)** | Prompt in CLI or web UI | KiCad schematic, placement, partial 2-layer routing, Gerber/Excellon/BOM/CPL, review | Free/MIT; BYOK Gemini | Research/hackathon | Partial router; 1 GitHub star; not production |
| **[JITX](https://www.jitx.com/product)** | Python code + AI edits | Schematic, board structure, routing, checks, HFSS | [Free design plan; enterprise custom](https://www.jitx.com/plans) | Product | Code-first, not prompt-first; tool closed |
| **[Circuit Mind](https://www.circuitmind.io/product)** | Requirements/architecture | Verified schematic, BOM, checks | Demo-gated; no public price | Product | No layout, routing, or Gerber |
| **[Celus](https://www.celus.io/)** | Natural-language requirements | Matched components, schematic, BOM, CAD handoff | Enterprise; no public price | Product | No layout |
| **[Quilter](https://www.quilter.ai/product)** | Upload schematic + board outline | Placement, routing, validation, native CAD outputs | [Per project/per pin; no public rate](https://www.quilter.ai/pricing) | Product | Not a schematic tool; upload required |
| **[DeepPCB](https://deeppcb.ai/)** | Upload a placed board | Routing only | [$30/h; $280/10h; $800/30h](https://deeppcb.ai/pricing/) | Product | 8 layers/1,200 airwires; not for production/high-speed/RF |
| **[EasyEDA + JLCPCB API Skill](https://jlcpcb.com/blog/ai-pcb-design-easyeda)** | Prompt via Claude Code/Codex | Native schematic, PCB, routing, ERC, JLC fab | Free editor; BYOK | Product/experiment | ~10 min setup; JLC-oriented |
| **[tscircuit](https://tscircuit.com/)** | TSX code | Schematic/PCB as code, real-time autorouting, Gerbers/KiCad | Free/MIT | Framework | Not a chat product |
| **[Cadence Allegro X AI](https://www.cadence.com/en_US/home/tools/pcb-design-and-analysis/allegro-x-design-platform.html)** | AI inside Allegro | Placement/routing, checks | Enterprise; no public price | Product | Enterprise-only |
| **[Atopile](https://github.com/atopile/atopile)** | `.ato` code + sidebar agent | Design intent, compiler/assertions, BOM/fab outputs; layout in KiCad | Free/MIT | Substrate | No documented autorouter; exact `ato build` Gerber behavior **Non vérifié** |

**Reading the table.** The "prompt" column hides very different products: browser ECAD with an AI copilot (Flux), a generated-board service with no manual layout (Cherry), a desktop AI-native suite (Trace), a design-and-fab bundle (Diode), a CLI that stops short of routing (Copperhead), a research-grade agent (Ada), and code-first tools (JITX, tscircuit, atopile). Treating them as one category is the first analytical mistake.

---

## 2. What is actually automated

Legend: **●** shipped and central; **◐** partial, assisted, or limited; **○** absent or out of scope; **?** non vérifié.

| Product | NL input | Schematic | Placement | Routing | Fab outputs | Verification | Open core |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Flux | ● | ● | ● | ● | ● | ◐ | ○ |
| Cherry Blossom | ● | ● | ● | ◐ | ● | ◐ | ○ |
| Trace | ● | ● | ● | ● | ● | ◐ | ○ |
| Diode | ● | ● | ● | ◐ | ● | ● | ○ |
| Copperhead | ● | ● | ◐ | ○ | ● | ● | ● |
| Ada / Silkscreen | ● | ● | ● | ◐ | ● | ● | ● |
| JITX | ◐ | ● | ● | ● | ● | ● | ○ |
| Circuit Mind | ● | ● | ○ | ○ | ○ | ● | ○ |
| Celus | ● | ● | ○ | ○ | ○ | ◐ | ○ |
| Quilter | ○ | ○ | ● | ● | ● | ● | ○ |
| DeepPCB | ○ | ○ | ○ | ● | ○ | ◐ | ○ |
| SnapMagic | ◐ | ◐ | ○ | ○ | ○ | ○ | ○ |
| EasyEDA + JLCPCB | ● | ● | ● | ● | ● | ● | ○ |
| tscircuit | ○ | ● | ◐ | ● | ● | ◐ | ● |
| Cadence Allegro X AI | ◐ | ● | ● | ● | ● | ● | ○ |
| Atopile | ◐ | ● | ○ | ○ | ◐ | ● | ● |

**Caveat.** This is a documentation-based assessment, not a hands-on benchmark. A **●** requires the capability to be a headline feature and documented; a **◐** is deliberately conservative. The matrix is not a quality score.

**Three patterns.**

- **Schematic generation is table stakes; layout is the filter.** Circuit Mind and Celus stop at a verified schematic and BOM. Flux, Cherry, Trace, Diode, Copperhead, Ada, Quilter, DeepPCB, and EasyEDA cross into layout. The hard part is not drawing a schematic; it is making the board manufacturable.
- **Routing is specialized and often delegated.** Quilter and DeepPCB sell routing as the product; Flux, Trace, and EasyEDA ship it inside a suite; Copperhead explicitly refuses to be an autorouter; Ada ships a partial 2-layer grid router that names what it could not route; atopile delegates place-and-route to KiCad. A weekend project should not build a router.
- **Verification is the new battleground, but a receipt is rare.** Many tools run ERC/DRC. Few bind the result to the exported artifacts in a way a buyer can inspect. Copperhead gates on ERC/DRC, Ada produces an adversarial review and a constraint "promotion receipt", and Diode says a human and an agent check the generated design. That is the bar NIA-PCB must clear.

---

## 3. Product notes

### Flux.ai

**What it automates.** Browser ECAD with an AI copilot. The vendor presents AI-assisted placement and routing, and exports Gerber, BOM, and CPL. The homepage says the platform supports up to 8 layers and shows scale counters: 1,099,361 builders, 6,425,482 projects, and 821,334 components (vendor marketing counts, not audited). [Homepage](https://www.flux.ai/)

**Input/output.** Prompt/chat inside an editor you can also draw in; output is a board plus fabrication files and a 3D view.

**Price.** Explore $50/mo (40 ACUs), Build $100/mo (120 ACUs), Pro $250/mo (400 ACUs); annual equivalents $40/$80/$200. Teams $500/mo for 5 seats and 600 pooled ACUs; extra seats $100/mo or $980/yr; additional ACUs $2 each; Teams annual $4,900. [Pricing](https://www.flux.ai/p/pricing)

**Limits.** 8 layers maximum; closed source; AI usage is credit-metered; the published entry plan is $50/mo, and a free tier was not verified. **Non vérifié:** the exact router and the existence of a portable verification receipt.

**Stage.** Product, with one of the largest visible communities in the category.

**Read for NIA-PCB.** Flux is the benchmark for browser UX and the most direct competitor. It is the reference for "AI PCB in the browser". Do not attack it head-on on features or model quality; compete on openness, inspectability, French-first access, and a receipt that survives export.

**Sources:** [flux.ai](https://www.flux.ai/) · [pricing](https://www.flux.ai/p/pricing)

### Cherry Blossom

**What it automates.** Prompt to a complete board: schematic, PCB, 3D, Gerbers, BOM/CPL, and KiCad/STEP export. The editor is closed, but the underlying Circuit JSON format is open. The vendor calls it an "early beta". [Homepage](https://www.trycherryblossom.com/) · [open standards](https://www.trycherryblossom.com/open-standards)

**Input/output.** A prompt; output is a generated board and a fab package. There is no import and no manual layout, so the product is a generator, not an editor.

**Price.** Free tier with the full editor; Pro $20/mo; Ultra $140/mo; overage $5 per 1M credits. [Pricing](https://www.trycherryblossom.com/pricing)

**Limits.** No import, no manual layout, no simulation; the vendor says it is not for high-speed or RF. Vendor claims: 10,000 PCBs built and 94% first-pass fab yield. **Non vérifié:** the router internals; the yield claim is not independently audited.

**Stage.** Early beta.

**Read for NIA-PCB.** This is the clearest proof that "prompt to fab package" is already a $20/mo product. The open Circuit JSON play is smart, but the editor remains closed. NIA-PCB should not compete on price; it should compete on source-level inspectability (`.ato`), verification, and French access.

**Sources:** [trycherryblossom.com](https://www.trycherryblossom.com/) · [pricing](https://www.trycherryblossom.com/pricing) · [open standards](https://www.trycherryblossom.com/open-standards) · [prompt-to-PCB use case](https://www.trycherryblossom.com/use-cases/prompt-to-pcb) · [GitHub org](https://github.com/cherryblossomhq)

### Trace

**What it automates.** A macOS desktop AI-native PCB suite: schematic, datasheet parsing, placement, routing, DRC, and Gerber/BOM/CPL. The vendor says it supports up to 32 layers; the site also claims 4,000+ engineers (vendor claim). [Homepage](https://buildwithtrace.com/)

**Input/output.** Prompt/chat in a desktop app; output is a native board and fabrication files.

**Price.** The editor is free; Pro $29.99/mo (~75 agent requests), Ultra $169.99/mo (~425), pay-as-you-go $20; 7-day trial with $30 credit. BYOK is supported. [Pricing](https://buildwithtrace.com/pricing)

**Limits.** macOS desktop only; closed source; AI usage is metered. **Non vérifié:** the quality of the router on dense boards; the "4,000+ engineers" figure is a vendor claim.

**Stage.** Product.

**Read for NIA-PCB.** Trace shows that BYOK is table stakes and that a desktop app can feel AI-native. Its distribution constraint (macOS) is an opening for a browser product. The differentiator is not "AI + schematic"; it is the verification surface and the French front door.

**Sources:** [buildwithtrace.com](https://buildwithtrace.com/) · [pricing](https://buildwithtrace.com/pricing)

### Diode

**What it automates.** Design plus fabrication and assembly. The vendor says requirements become a schematic and layout "written as code", checked by an AI agent and engineers, with a self-serve instant fab/assembly quote. [Homepage](https://www.diode.computer/) · [build app](https://build.diode.computer/)

**Input/output.** Requirements/chat; output is a checked design and a manufacturing quote. The build app is login-gated, so the exact automation is not publicly inspectable.

**Price.** Enterprise from $10,000 per design plus fabrication; the self-serve app provides instant quotes. [Homepage](https://www.diode.computer/)

**Limits.** Enterprise pricing and a gated app; public detail is thin. The tool itself is closed; open-source reference designs are published, but they are not the tool. **Non vérifié:** how much of the layout is routed automatically, and what the checks cover.

**Stage.** Product, enterprise-oriented; Brooklyn-based; the site links to an Anthropic collaboration and an a16z Series A, and publishes open-source reference designs. Treat the collaboration and funding as vendor/announcement claims.

**Read for NIA-PCB.** Diode has the strongest "verification + manufacturing" bundle in the category: it sells the checked artifact, not the drawing. NIA-PCB cannot bundle fabrication in 73 hours, but it can bundle the receipt and the handoff. That is the piece of Diode's value that a hackathon can credibly demo.

**Sources:** [diode.computer](https://www.diode.computer/) · [build.diode.computer](https://build.diode.computer/)

### Copperhead

**What it automates.** An Apache-2.0 CLI that is KiCad-native: spec, architecture, parts, schematic, first-draft layout, outputs, firmware, and a development plan. It gates on ERC/DRC and exports Gerbers/STEP. It is explicitly **"Not an autorouter."** [Homepage](https://copperhead.sh/) · [GitHub](https://github.com/copperheadhq/copperhead)

**Input/output.** Spec/chat in a terminal; output is a KiCad-native design plus fabrication outputs and the ERC/DRC gate results.

**Price.** CLI free; Cloud $49/user/mo; Team $49/user/mo plus $199/mo platform; free for open-source hardware; BYOK or managed models. [Pricing](https://copperhead.sh/pricing/)

**Limits.** Early-stage; the CLI is the primary surface; no router by design. 632 GitHub stars at the time of writing. **Non vérifié:** how much of the "first-draft layout" is placement versus routing; the answer is likely placement plus a template, not routing.

**Stage.** Early.

**Read for NIA-PCB.** Copperhead is the closest philosophical competitor: open, CLI, verification gates, and honest about routing. NIA-PCB's difference must be browser UX, French, atopile-native `.ato`, and a visual diff — not "open" or "verification" alone, because Copperhead already has both.

**Sources:** [copperhead.sh](https://copperhead.sh/) · [pricing](https://copperhead.sh/pricing/) · [GitHub](https://github.com/copperheadhq/copperhead)

### Ada / Silkscreen

**What it automates.** An MIT-licensed, CLI-first "AI hardware engineer" with a web UI. It turns a prompt into a KiCad `.kicad_sch`, uses CP-SAT for placement, runs a partial 2-layer grid router that explicitly names unrouted nets, and exports Gerber/Excellon/BOM/CPL. It also runs an adversarial review, SPICE, MCP integration, and a constraint "promotion receipt". [GitHub](https://github.com/machmoon/Ada)

**Input/output.** Prompt in a CLI or web UI; output is a KiCad project, fabrication files, and a review/receipt.

**Price.** Free/MIT; bring your own Gemini key.

**Limits.** Research/hackathon stage; 1 GitHub star at the time of writing; the router is partial (2-layer grid) and openly reports what it could not route. It is not a production tool.

**Stage.** Research/hackathon.

**Read for NIA-PCB.** Ada proves that the receipt/adversarial-review idea is technically feasible for a small team — and that the idea alone is not enough to differentiate. NIA-PCB must make the receipt visual, French, atopile-native, and tied to a fab package. Ada also sets a useful honesty bar: name the unrouted nets instead of pretending the board is complete.

**Sources:** [github.com/machmoon/Ada](https://github.com/machmoon/Ada)

### JITX

**What it automates.** A Python-based design environment with AI edits: schematics, board structures, routing solvers, fencing, pin assignment, HFSS-in-the-loop analysis, and checks. The tool runs locally. [Product](https://www.jitx.com/product)

**Input/output.** Code plus AI edits; output is a design database, schematics, a routed board, and analysis/check results.

**Price.** A free open-source plan covers designs only, released under CERN OHL-Permissive v2; enterprise/program pricing is custom. [Plans](https://www.jitx.com/plans)

**Limits.** Code-first, not prompt-first; the tool itself is closed; the free plan is for open-source designs, not a free tool for arbitrary commercial work. The site displays Honeywell, Lockheed, and OpenAI logos; treat those as customer/logo claims. **Non vérifié:** the exact scope of the free plan and the licensing of the generated design database.

**Stage.** Product, high-end/regulated.

**Read for NIA-PCB.** JITX is the "design as code" ancestor and a reminder that high-end teams pay for correctness, not for a chat box. Atopile is the open, lower-friction analogue. Do not compete on HFSS or enterprise integration in 73 hours.

**Sources:** [jitx.com/product](https://www.jitx.com/product) · [jitx.com/plans](https://www.jitx.com/plans)

### Circuit Mind

**What it automates.** ACE converts requirements and architecture into a "verified schematic", BOM, and a set of checks. It stops before layout, routing, and Gerber generation. [Product](https://www.circuitmind.io/product)

**Input/output.** Requirements/architecture; output is a verified schematic and BOM, not a board.

**Price.** Demo-gated; no public price. **Non vérifié:** pricing.

**Limits.** No layout, routing, or fabrication outputs; the buyer needs an ECAD flow to finish the board.

**Stage.** Product.

**Read for NIA-PCB.** Circuit Mind validates "verified schematic" as a category and shows that stopping before layout can be a feature. NIA-PCB goes further — to Gerber — but only if it brings the same verification discipline. The moment the demo shows a board without checks, it looks less serious than Circuit Mind.

**Sources:** [circuitmind.io/product](https://www.circuitmind.io/product)

### Celus

**What it automates.** Natural-language requirements become matched components, a schematic, and a BOM, with native handoff to Altium, Cadence, Siemens, Fusion, and KiCad. It does not do layout. [Homepage](https://www.celus.io/)

**Input/output.** Requirements text; output is a component-matched schematic and BOM in a mainstream CAD format.

**Price.** Enterprise; no public price. **Non vérifié:** pricing.

**Limits.** No layout; enterprise sales motion. The site displays quotes from NXP and BrainChip; treat them as customer claims.

**Stage.** Product, enterprise.

**Read for NIA-PCB.** Component selection and schematic capture are a real business, but not a 73-hour hackathon competitor. Celus is a potential partner/data source, not a head-to-head rival.

**Sources:** [celus.io](https://www.celus.io/)

### Quilter

**What it automates.** Physics-driven AI placement and routing from an uploaded complete schematic and board outline, plus validation. It outputs to Altium, Cadence, Siemens, and KiCad. [Product](https://www.quilter.ai/product)

**Input/output.** Upload a schematic and outline; output is a placed/routed board in a native CAD format.

**Price.** Per project, priced by unrouted pins; no public rate. The pricing page describes a free tier that contributes designs to training — review the terms before uploading proprietary work. [Pricing](https://www.quilter.ai/pricing)

**Limits.** Not a schematic tool; you must bring a complete schematic and outline. The vendor says it handles up to 10,000 pins. **Non vérifié:** the per-project rate and the exact data-retention terms.

**Stage.** Product, enterprise.

**Read for NIA-PCB.** Quilter is the routing specialist. If NIA-PCB needs routing beyond a template, integrate a real router or a specialist; do not build one in 73 hours. Quilter also proves that "physics-driven" is a credible marketing frame for verification.

**Sources:** [quilter.ai/product](https://www.quilter.ai/product) · [pricing](https://www.quilter.ai/pricing)

### DeepPCB

**What it automates.** A learned routing engine. You upload a placed board; it returns a routed board. It is not a schematic or placement tool. [Homepage](https://deeppcb.ai/)

**Input/output.** Placed board in; routed board out.

**Price.** $30/hour, $280 for 10 hours, $800 for 30 hours; free 30-minute trial. [Pricing](https://deeppcb.ai/pricing/)

**Limits.** Up to 8 layers and 1,200 airwires; the vendor explicitly says it is not for production, high-speed, or RF. It returns a routed board; a Gerber export was not verified. **Non vérifié:** the quality of the routed result on dense boards.

**Stage.** Product.

**Read for NIA-PCB.** Even a dedicated routing AI is narrow, cheap, and honest about its limits. Routing is not the wedge; the wedge is the checked handoff around it.

**Sources:** [deeppcb.ai](https://deeppcb.ai/) · [pricing](https://deeppcb.ai/pricing/)

### SnapMagic (formerly SnapEDA)

**What it automates.** A free parts library, plus an AI copilot for schematic autocomplete and BOM optimization (early access). It does not automate full layout. [Homepage](https://www.snapmagic.com/)

**Input/output.** Part searches and schematic/BOM assistance; output is symbols, footprints, and 3D models.

**Price.** The library is free; InstaPart credits are $135/5, $250/10, and $400/20. Copilot pricing/availability is early access. [Pricing](https://www.snapeda.com/pricing/)

**Limits.** No full board generation; Copilot is early access. **Non vérifié:** Copilot pricing and general availability.

**Stage.** Product (library) + early access (Copilot).

**Read for NIA-PCB.** Parts/data is a separate business with its own moat. Do not build a parts library in 73 hours; integrate one. SnapMagic is a potential supplier, not a competitor.

**Sources:** [snapmagic.com](https://www.snapmagic.com/) · [snapeda.com/pricing](https://www.snapeda.com/pricing/)

### EasyEDA Pro + JLCPCB API Skill

**What it automates.** A free editor plus a local bridge and an "API Skill" that lets Claude Code or Codex drive EasyEDA: prompt to native schematic, PCB, routing, and ERC, tied to JLCPCB's BOM and fabrication flow. The vendor says setup takes about 10 minutes. [JLCPCB blog](https://jlcpcb.com/blog/ai-pcb-design-easyeda)

**Input/output.** Prompt via a coding agent; output is a standard EasyEDA project and a JLCPCB-oriented fab package.

**Price.** Free editor; bring your own Claude Code/Codex key.

**Limits.** Requires a local bridge and a coding agent; the workflow is JLC-oriented; the exact routing quality is **Non vérifié** and was not independently tested. The editor is closed.

**Stage.** Product/experiment.

**Read for NIA-PCB.** This is the "free + BYOK + fab" bundle already existing. NIA-PCB cannot differentiate on "AI + JLCPCB". It can differentiate on a compiler-backed design, a receipt, French, and a conversational diff that a non-programmer can read.

**Sources:** [jlcpcb.com/blog/ai-pcb-design-easyeda](https://jlcpcb.com/blog/ai-pcb-design-easyeda)

### tscircuit

**What it automates.** A TypeScript/TSX framework for defining circuits as code, with real-time autorouting and exports to Gerber, KiCad, and Circuit JSON. The project calls itself a "framework for AI-generated electronics". [Homepage](https://tscircuit.com/)

**Input/output.** TSX code; output is a schematic/PCB, routed traces, and fabrication files.

**Price.** Free/MIT.

**Limits.** It is infrastructure, not a chat product; you still need a UI, a compiler/check layer, and a workflow. 2,883 GitHub stars at the time of writing. **Non vérifié:** the maturity of the autorouter on real boards.

**Stage.** Framework.

**Read for NIA-PCB.** A potential substrate or partner, not a UX competitor. It reinforces that "open + code + routing" is already available; the missing piece is the accessible, verified front door.

**Sources:** [tscircuit.com](https://tscircuit.com/)

### Seeed CodeCraft

**What it automates.** A hardware-focused AI coding assistant for firmware and device projects, not PCB layout. A July 2026 Seeed post describes "turn words into working hardware in minutes"; the official article returned HTTP 403 during research, so only a search snippet was available. [Article URL](https://www.seeedstudio.com/blog/2026/07/02/codecraft-is-here-turn-words-into-working-hardware-in-minutes/)

**Input/output.** Natural language to firmware/device project scaffolding; not a board.

**Price.** **Non vérifié.**

**Limits.** Not a PCB layout tool. **Non vérifié:** features, price, and availability.

**Stage.** **Non vérifié.**

**Read for NIA-PCB.** Adjacent, not a direct competitor. If anything, it is a distribution partner for the firmware side of a hardware project; it does not solve the board.

**Sources:** [Seeed article](https://www.seeedstudio.com/blog/2026/07/02/codecraft-is-here-turn-words-into-working-hardware-in-minutes/) (HTTP 403; snippet only)

### Cadence Allegro X AI

**What it automates.** AI-assisted placement and routing inside the Allegro X enterprise PCB flow. [Product page](https://www.cadence.com/en_US/home/tools/pcb-design-and-analysis/allegro-x-design-platform.html)

**Input/output.** Enterprise Allegro design data; output is a placed/routed board and checks.

**Price.** Enterprise; no public price. **Non vérifié:** pricing and exact AI feature boundaries.

**Limits.** Enterprise-only; not accessible to the hackathon audience.

**Stage.** Product.

**Read for NIA-PCB.** The incumbent validates the direction: AI placement/routing is real. It also shows why an open, low-friction front door matters; the incumbents' AI is locked inside expensive flows.

**Sources:** [Cadence Allegro X](https://www.cadence.com/en_US/home/tools/pcb-design-and-analysis/allegro-x-design-platform.html)

### Atopile

**What it automates.** A declarative `.ato` language and compiler with a solver, assertions, and parametric parts. `ato build` produces BOM/fab outputs; layout happens in KiCad. The repository ships a sidebar-agent skill, so atopile is not "AI-free" — it already has an agent surface. [GitHub](https://github.com/atopile/atopile) · [agent skill](https://github.com/atopile/atopile/blob/main/.claude/skills/agent/SKILL.md) · [docs](https://docs.atopile.io/)

**Input/output.** `.ato` source + agent edits; output is a compiled design, assertions, BOM/fab data, and a KiCad layout workflow.

**Price.** Free/MIT.

**Limits.** No documented autorouter: the README puts place-and-route in KiCad. The exact Gerber behavior of `ato build` is **Non vérifié**; plan to use KiCad CLI for Gerbers/drill. 4,023 GitHub stars and 256 forks at the time of writing.

**Stage.** Substrate/open-source project.

**Read for NIA-PCB.** This is the substrate, not the competitor. Do not fork it; expose it. The compiler and assertions are the trust anchor; the `.ato` file is the inspectable artifact; KiCad is the escape hatch. The built-in agent means "we added an LLM" is not a differentiator — the front door, the diff, the receipt, and the French UX are.

**Sources:** [github.com/atopile/atopile](https://github.com/atopile/atopile) · [agent skill](https://github.com/atopile/atopile/blob/main/.claude/skills/agent/SKILL.md) · [docs.atopile.io](https://docs.atopile.io/)

---

## 4. The pricing spectrum

| Band | Examples | What it tells us |
|---|---|---|
| **Free/open** | Atopile (MIT), tscircuit (MIT), Copperhead CLI (Apache-2.0), Ada (MIT), [KiCad](https://www.kicad.org/), EasyEDA (free editor) | The floor is $0. "Free" is not a differentiator; the paid value must be hosting, verification, handoff, or team workflow. |
| **$20–$50/mo** | Cherry Pro $20, Trace Pro $29.99, Copperhead Cloud $49/user, Flux Explore $50 | A solo builder can buy a serious AI PCB tool for the price of a music subscription. Price is not the barrier. |
| **$100–$250/mo** | Flux Build $100, Cherry Ultra $140, Trace Ultra $169.99, Flux Pro $250 | Pro/team tiers are already normalized. A hackathon product cannot undercut this on features alone. |
| **Teams** | Flux Teams $500/mo (5 seats), Copperhead Team $49/user + $199/mo platform | Collaboration and pooled usage are the upsell. |
| **Usage** | DeepPCB $30/h; SnapMagic credits $135/5; Cherry $5/1M credits | Compute and data are metered; users accept it. |
| **Enterprise/custom** | Diode from $10,000/design + fab; JITX, Circuit Mind, Celus, Quilter, Cadence (no public price) | The ceiling is high when the product carries risk (verification, fabrication, compliance). |

**Implication for NIA-PCB.** Be free and BYOK at the point of demo; monetize trust and handoff later (hosted verification runs, team receipts, fab handoff, support). Do not charge for "generation" in a market where generation is $20/mo or free.

---

## 5. What a solo can credibly ship in 73 hours

**Principle: don't build the engine; build the cockpit.** The engine already exists: atopile compiles `.ato` and checks assertions; KiCad does layout, DRC, and Gerbers; pre-routed templates handle the hard geometry; a BYOK model handles language; existing parts data handles symbols and footprints. The unique work is the orchestration — prompt to `.ato`, a constraint harness, a conversational diff, a receipt, and a French UI.

### Three credible angles, ranked

1. **Verification-first front door (recommended).** Prompt → `.ato` → compile/assertions → template layout → receipt → fab package. The differentiator is the receipt; the demo is a failure-and-repair loop. It uses atopile's compiler as the engine and KiCad as the layout/DRC engine. It is the only angle that produces all three demo beats.
2. **Conversational redesign studio.** Prompt → `.ato` diff → BOM delta → before/after board, without a full fab handoff. It is easier to build (no Gerber/DRC pipeline) but easier to dismiss as a chat wrapper; it needs the compiler failure to be credible.
3. **French-first fab handoff.** Templates + prompt + Gerber/BOM/CPL export, with French UX. It is useful, but the moat is thin: EasyEDA + JLCPCB and Cherry already do fab handoff, and translation is copyable. It works as a feature, not as the product.

**Why angle 1 wins for NIA-PCB.** It is the only one where the three differentiators are all on screen in three minutes: the constraint that bites (compiler), the conversational redesign (diff + BOM + board), and the verifiable handoff (receipt + exports). It also matches the team's existing advantage: atopile-native, not a new ECAD.

**Scope to one board class.** Credible in 73 hours: a 2-layer (or simple 4-layer), low-speed, 5V/3.3V board with fewer than ~50 components, all off-the-shelf, no RF, no high-speed serial, no dense BGA, no precision analog. A good demo board is an ESP32-C3 + USB-C + I²C sensor + 3.3V LDO; an STM32G0 + RS-485 + 24V input is a credible industrial variant. Do not promise arbitrary boards.

### 73-hour scope budget

| Hours | Workstream | Deliverable | What the demo proves |
|---|---|---|---|
| 0–4 | Pick the board and the boundary | One board class, one BOM, one template | "We do this one thing." |
| 4–14 | Prompt → `.ato` | Structured prompt, part selection, generated source | The prompt produces inspectable code, not a picture. |
| 14–26 | Compile/assert harness | `ato build`, assertions, ERC; error formatting | A false constraint stops the build. |
| 26–38 | Layout from a template | KiCad template bound to the netlist; DRC | The board is real and opens in KiCad. |
| 38–48 | Conversational redesign | `.ato` diff, BOM delta, before/after board | Chat changes the design, not the render. |
| 48–58 | Receipt and exports | Gerber/drill/BOM/CPL + check results + hashes | The output is bound to proof. |
| 58–68 | UI and French polish | Chat left, board right, receipt panel | A non-expert can use it. |
| 68–73 | Rehearsal and backup | 3-minute script, recorded fallback | The demo survives the room. |

The hours are a scope budget, not a Gantt chart; adjust them as the build teaches you. The important part is that no workstream is "build a router" or "train a model".

### What not to build in 73 hours

- A general-purpose autorouter, a new ECAD, a new compiler, or a parts library.
- 4+ layer high-speed, RF, analog precision, mechanical enclosures, compliance, or sourcing at scale.
- A new LLM or a fine-tune. Use BYOK and spend the time on the harness and the UX.
- A "one-click fab" that hides the checks. The receipt is the product, not the button.

### Acceptance criteria

1. `ato build` succeeds on the demo board, and the assertions are visible.
2. ERC/DRC runs and the report is attached to the exported artifacts.
3. Gerber, drill, BOM, and CPL open correctly; the Gerbers open in KiCad.
4. One deliberate constraint failure is shown and repaired on screen.
5. One conversational redesign produces a real `.ato` diff and a BOM delta.
6. The UI is French-first and usable without a terminal.
7. A 3-minute demo runs without a slide; a recorded fallback exists.

### Risk register

| Risk | Mitigation |
|---|---|
| No router | Use pre-routed templates for the demo board; say "template" before a judge asks; integrate KiCad/third-party routing only if time remains. |
| The model invents a part | Constrain the prompt to a curated BOM; validate against the library; show the part list. |
| Live compile fails | Keep the failure as a feature if it is a real constraint; otherwise use the recorded run. |
| Network/model latency | Cache responses; precompute the happy path; keep a local BYOK fallback. |
| Scope creep | One board class, one repair, one diff, one receipt. |
| "You're just a wrapper" | Show the `.ato` source, the compiler error, the diff, and the receipt; the wrapper is the cockpit, and the cockpit is the product. |

---

## 6. "Push the Frontier": the impact narrative

**The frontier is not generation. It is trust at the point of fabrication.** Anyone can make a plausible board image. The scarce thing is a board that compiles, passes its checks, and can be inspected by the person who pays for the fabrication. NIA-PCB pushes the frontier of **who** can make a board and **what they can prove** before they pay for it.

### Numbers to use (and how)

| Metric | Number | Source | How to use it |
|---|---|---|---|
| Global PCB market, 2025 | ~$85B | [Prismark](https://www.prismark.com/what-s-new) | Context: a large, real market. Do not call it the TAM. |
| Global PCB market, 2030 | $123.3B | [Prismark](https://www.prismark.com/what-s-new) | Direction of travel; note the 2025 figure is reported as $85.2B on one page and $85.8B on another. |
| US electrical/electronics engineering jobs, 2025 | 297,900 | [BLS](https://www.bls.gov/ooh/architecture-and-engineering/electrical-and-electronics-engineers.htm) | Engineering time is the scarce input. |
| Median annual wage | $125,040 ($60.12/h) | [BLS](https://www.bls.gov/ooh/architecture-and-engineering/electrical-and-electronics-engineers.htm) | Translate hours into money. |
| Job growth, 2025–35 | 8% | [BLS](https://www.bls.gov/ooh/architecture-and-engineering/electrical-and-electronics-engineers.htm) | Steady demand, not hype. |
| Openings per year | ~16,300 | [BLS](https://www.bls.gov/ooh/architecture-and-engineering/electrical-and-electronics-engineers.htm) | A recurring need, not a one-off. |
| A 40-hour board design at the median | ≈ $2,400 | Calculation: 40 × $60.12 | Label it a calculation; it excludes overhead and is not a respin cost. |
| French speakers | 396M | [OIF 2026](https://www.francophonie.org/lancement-du-rapport-la-langue-francaise-dans-le-monde-2026-8426) | The access wedge is real and measurable. |
| French speakers in Africa | 65% | [OIF 2026](https://www.francophonie.org/lancement-du-rapport-la-langue-francaise-dans-le-monde-2026-8426) | The fastest-growing part of the francophonie. |
| French learners | ~170M | [OIF 2026](https://www.francophonie.org/lancement-du-rapport-la-langue-francaise-dans-le-monde-2026-8426) | A pipeline of future builders. |
| French speakers by 2050 | 590M; 9 in 10 in Africa | [OIF 2026](https://www.francophonie.org/lancement-du-rapport-la-langue-francaise-dans-le-monde-2026-8426) | The long-term access story. |
| Atopile community | 4,023 stars, 256 forks | [GitHub](https://github.com/atopile/atopile) | An existing open-source base, not a cold start. |
| Competitor prices | $20–$250/mo | Vendor pricing pages (above) | Free/BYOK at demo; do not compete on price. |

### The argument in four beats

1. **Generation is commoditized.** Flux, Cherry, Trace, Diode, Copperhead, Ada, EasyEDA + JLCPCB, and tscircuit all generate or route boards. A better prompt wrapper is not a frontier.
2. **Verification is scarce.** Copperhead gates on ERC/DRC, Ada ships an adversarial review and a receipt, Diode checks with an agent and engineers, Circuit Mind sells a verified schematic. The market is moving from "look at this board" to "prove this board".
3. **Access is scarcer.** 396M French speakers, 65% in Africa, 170M learners — and the leading tools are English-first, paid, desktop-bound, or enterprise-gated. A French-first, browser-based front door is not a translation; it is distribution.
4. **The receipt is the product.** A Gerber is a promise. The receipt is the evidence. NIA-PCB binds the `.ato` source, the compiler/assertion results, ERC/DRC, the BOM/CPL, and the exported artifacts into one inspectable package.

**One-sentence pitch:** "We push the frontier of who can make a board, and of what they can prove about it before they pay for fabrication."

**Do not claim:** a dollar cost per respin (no reliable public figure was found), "first", "arbitrary boards", "production-ready", or "auto-routed" unless a real router runs on screen. The honest version is stronger.

---

## 7. The three-minute, screen-only demo

**Setup.** One browser window with NIA-PCB: chat on the left, board on the right, receipt panel below. A second window with KiCad and a Gerber viewer. A terminal only if needed to show `ato build`. Pre-load the demo board and the repair case; record a backup video. No slides.

### Script

**0:00–0:20 — The promise, in French.**
Type: *"Je veux une carte 2 couches pour un ESP32-C3, alimentée en USB-C, avec un capteur I²C et un LDO 3,3 V. Contraintes : USB-C 5 V, découplage < 2 mm, taille < 40 × 30 mm."*
Say: "Not a chatbot that draws a picture. A compiler that has to agree."
On screen: the prompt, then the generated `.ato` source.

**0:20–0:50 — The constraint that bites.**
Show the `.ato` file and its assertions. Add a deliberately impossible constraint (for example, "VBUS must survive 20 V" with a parts set that cannot). Run `ato build`. A red error names the constraint and the offending part. The agent proposes a repair (a higher-voltage regulator or a protection device); accept it; the build turns green.
Say: "This is the difference: the board cannot be generated if the constraint is false."
Proof: constraints are executable, not prose; the repair is a real change.

**0:50–1:30 — The conversational redesign.**
Ask: *"Remplace l'USB-C par une alim 12 V sur bornier, garde le reste."*
Show the `.ato` diff (connector, protection, regulator), the BOM delta (remove USB-C; add jack, TVS, buck), and the before/after board. The board comes from a pre-routed 12V template; say it: "The layout is a pre-routed template re-bound to the new netlist; the router is KiCad/third-party, not a model."
Proof: the chat changes the design, not just the render; the diff is inspectable.

**1:30–2:30 — The receipt.**
Click "Verify". Show the list: compile, assertions (e.g., 14/14), ERC, DRC, BOM/CPL match, Gerber/drill present — each with a hash and a timestamp. Open the 2D Gerber, the 3D view, and the KiCad project. Show one failed check (e.g., a clearance) and the fix/re-run, or one accepted warning with a written reason.
Say: "No Gerber without a receipt. The receipt is the product."
Proof: verification is visible and bound to the artifacts.

**2:30–2:50 — The fab handoff.**
One click: Gerber/drill/BOM/CPL zip; a JLCPCB/PCBWay quote link; show the zip contents and the layer count.
Say: "From a French sentence to a fab-ready package in under three minutes — with the checks attached."

**2:50–3:00 — The close.**
"The world doesn't need another board generator. It needs a front door that can prove what it made. Open, French-first, atopile-native: NIA-PCB."
Show the repo URL/QR code. Stop.

### Why each beat is a differentiator, not a slide

| Beat | Differentiator | What the judge sees | Why it matters |
|---|---|---|---|
| 0:20–0:50 | **The constraint that bites** | A red compiler/assertion error, then a repaired design | Proves the board is compiled, not hallucinated. |
| 0:50–1:30 | **Conversational redesign** | `.ato` diff + BOM delta + before/after board | Proves the chat changes the engineering, not the picture. |
| 1:30–2:50 | **Verifiable handoff** | Receipt + Gerber/drill/BOM/CPL + KiCad | Proves the output is inspectable and fab-ready. |

### Fallbacks

- If the live compile fails in an unplanned way, keep it if it is a real constraint; otherwise cut to the recorded run. Never fake a green build.
- If the network or model is slow, use the cached happy path and the recorded fallback.
- If a judge asks about routing, say "pre-routed template" before they finish the question. Do not imply a model routed the board.
- If time slips, cut the 3D view and keep the receipt + Gerber. The receipt is the point.
- If a judge asks "why not Flux/Cherry?", answer: open `.ato`, French-first, and a receipt that survives export — not "better AI".

---

## 8. Positioning for NIA-PCB

> **The open, French-first, verification-forward front desk for atopile:** describe a board, watch it compile, edit it in conversation, and leave with Gerbers you can inspect.

**Positioning statement.** NIA-PCB is not another board generator. It is the French-first, open, verification-forward front desk for atopile: a browser cockpit where a prompt becomes an inspectable `.ato` design, a conversation becomes a reviewable diff, and a Gerber is never exported without a receipt.

### The gap

The market already has a general-purpose browser ECAD (Flux), a generated-board service (Cherry), an AI-native desktop suite (Trace), a design-and-fab bundle (Diode), an open CLI (Copperhead), a research agent (Ada), code-first tools (JITX, tscircuit), and enterprise schematic/routing specialists (Circuit Mind, Celus, Quilter). Atopile sits underneath as an open compiler with assertions and a real language, but its surface is a CLI/repo and its built-in agent is not a French-first browser product. The gap is not "AI PCB". It is **an accessible, inspectable, verifiable front door to an open compiler**.

### The bet

The winning artifact is not the board image; it is the **receipt**: `.ato` source + compiler/assertion results + ERC/DRC + BOM/CPL + exported Gerbers, bound together and reproducible. If the receipt is good, the user can trust the output, a team can review it, and a fab can build it. If the receipt is bad, the product is a demo.

### Why this is defensible (for a weekend and beyond)

- **Open-source trust.** Atopile is MIT; `.ato` is inspectable; there is no lock-in. Most competitors' tools are closed (Flux, Cherry, Trace, Diode, Quilter, Cadence).
- **French-first distribution.** 396M French speakers, 65% in Africa, 170M learners; the public sites reviewed here are English-first. This is not a translation layer; it is a different distribution.
- **Integration depth.** Atopile + KiCad + a fab house. The last mile — compile, verify, export, hand off — is the moat.
- **A portable receipt.** The model is replaceable; the receipt format is not. A team can archive it, diff it, and attach it to a purchase order.

### What we are not

- Not a new ECAD, router, compiler, parts library, or LLM.
- Not "the first AI PCB tool" — that claim is unprovable and unnecessary.
- Not a general-purpose board generator; we do one board class well.
- Not a fab; we hand off to one.
- Not a replacement for atopile; we are its front door.

### The three differentiators, and why they fit in three minutes

1. **The constraint that bites.** A real compiler/assertion failure, then a repair. This proves the board is computed, not hallucinated.
2. **The conversational redesign.** An `.ato` diff, a BOM delta, and a before/after board. This proves the chat changes the engineering, not the render.
3. **The verifiable handoff.** A receipt bound to Gerber/drill/BOM/CPL, openable in KiCad. This proves the output is inspectable and fab-ready.

Each is a screen beat, not a slide. Each maps to a section of the demo script above.

### Why now

- **Generation is commoditized.** Flux, Cherry, Trace, Diode, Copperhead, Ada, EasyEDA + JLCPCB, and tscircuit all generate or route boards. A better prompt wrapper is not a frontier.
- **Verification is the new axis.** Copperhead gates on ERC/DRC; Ada ships an adversarial review and a receipt; Diode checks with an agent and engineers; Circuit Mind sells a verified schematic. The market is moving from "look at this board" to "prove this board".
- **Atopile already has an agent.** "We added an LLM" is not a differentiator. The front door, the diff, the receipt, and the French UX are.
- **French-first access is an open position.** The products reviewed here compete on generation, routing, or enterprise workflow; French-first access is not their positioning. The OIF numbers make the audience concrete.

### Business model (hypothesis, not a claim)

- Free/BYOK for individuals.
- Paid hosted verification runs, team receipts, fab handoff, and support.
- Open-core: the compiler and `.ato` stay open; the hosted cockpit and receipt service are paid.
- Do not charge for generation at the demo; the market already offers free tiers and $20/mo entry points.

### Risks and mitigations

| Risk | Why it matters | Mitigation |
|---|---|---|
| Flux/Cherry/Trace add a receipt | They have distribution and can copy a feature | Our wedge is openness + French + atopile-native depth, not "verification" alone. |
| Atopile ships a better agent | The substrate could absorb the cockpit | Integrate deeply; be the best front door, not a fork. |
| No router | The demo cannot claim full automation | Pre-routed templates; say so; integrate KiCad/third-party only if time remains. |
| "You're a wrapper" | Judges have seen chat wrappers | Show the `.ato` source, the compiler error, the diff, and the receipt. The cockpit is the product. |
| Demo fragility | Live compiles fail | Recorded fallback; cached parts; keep a real failure as a feature. |
| Scope creep | 73 hours is short | One board class, one repair, one diff, one receipt. |

### What winning looks like

The judge can retell it in one sentence: **"It compiles, it diffs, it receipts."** One board, one deliberate failure, one repair, one conversational diff, one receipt, one fab package. No overclaim; every claim is visible on screen.

---

## 9. Method and limitations

- Sources were selected from official product sites, pricing pages, documentation, GitHub repositories, and primary market/labor data. No paid accounts were purchased and no product was tested hands-on; this is a documentation-based landscape.
- Analysis and recommendations (positioning, demo script, 73-hour plan) are editorial, not factual claims.
- Vendor claims (scale counters, yield, customer logos, "first") are labelled as claims. They are not independently audited.
- Prices are list prices in USD on 27 October 2026, excluding tax; they can change without notice.
- Where a page was login-gated, JavaScript-rendered, or returned an error, the gap is recorded below rather than filled with a guess.
- GitHub star/fork counts are date-stamped and move daily.

---

## 10. Unverified / non vérifié

| Item | Why it is unverified | What to do |
|---|---|---|
| Circuit Mind pricing | The pricing page was not publicly available (HTTP 401 during research); no public price found. | Say "demo-gated; no public price". Do not invent a number. |
| Celus pricing | No public pricing page. | Say "enterprise; no public price". |
| Quilter per-project/per-pin rate | The pricing page explains the model but publishes no rate. | Say "priced per project by unrouted pins; no public rate". |
| Quilter data terms | The free tier is described as contributing designs to training; exact retention terms were not reviewed in full. | Read the terms before uploading proprietary designs. |
| Seeed CodeCraft | The official article returned HTTP 403; only a search snippet was available. | Treat as adjacent and unconfirmed; do not build a claim on it. |
| Diode build-app automation | The build app is login-gated; the homepage describes the flow but not the router or check coverage. | Say "schematic and layout written as code, checked by an agent and engineers" (vendor claim). |
| SnapMagic Copilot pricing/GA | Early access; no public price or general-availability date. | Say "early access". |
| Atopile `ato build` Gerber behavior | The README says BOM/fab outputs and layout in KiCad; the exact Gerber path was not verified. | Plan to use KiCad CLI for Gerbers/drill; test before the demo. |
| Atopile built-in autorouter | Its absence is inferred from the README (place-and-route in KiCad), not from an explicit "we have no router" statement. | Do not claim atopile routes; say "layout in KiCad". |
| Cost of a PCB respin | No reliable public figure was found. | Do not use a dollar figure; use engineering hours or the BLS-derived calculation, labelled. |
| Prismark 2025 market size | One page says $85.2B, another says $85.8B. | Say "about $85B" and note the discrepancy. |
| Flux scale counters | 1,099,361 builders / 6,425,482 projects / 821,334 components are homepage marketing counts. | Label as vendor claims; do not present as audited. |
| Cherry yield claim | 10,000 PCBs and 94% first-pass fab yield are vendor claims. | Label as vendor claims. |
| Trace "4,000+ engineers" | Vendor claim; no independent source. | Label as a vendor claim. |
| JITX customer logos | Honeywell/Lockheed/OpenAI appear as logos/customer claims. | Label as logo claims; do not imply contracts. |
| Diode Anthropic/a16z | The homepage links to an Anthropic collaboration and an a16z Series A. | Label as announcement/vendor claims; do not invent the amount. |
| Celus customer quotes | NXP/BrainChip quotes appear on the site. | Label as customer quotes. |
| Flux free tier | The pricing page shows a $50/mo entry plan; a free tier was not verified. | Do not promise a free Flux plan. |
| Cherry router internals | The site presents a generated board but does not document the router. | Say "generated board"; do not name a router. |
| Diode routing coverage | Public pages do not say whether every net is autorouted. | Say "layout generated and checked"; do not say "fully autorouted". |
| JITX free-plan scope | The free plan is designs-only under CERN OHL-Permissive v2; exact tool rights were not verified. | Read the plan/licence before claiming "free tool". |
| Cadence Allegro X AI pricing | Enterprise; no public price. | Say "enterprise; no public price". |
| EasyEDA/JLCPCB routing quality | The workflow is documented in a blog; it was not tested. | Say "documented workflow; not independently tested". |
| OIF figures | Definitions of "speaker" and "learner" vary; the 2026 report page is the source. | Cite the OIF page; do not extrapolate beyond it. |
| BLS figures | US-only; median wage; not a global labor number. | Say "US" and "median". |
| Market size vs. TAM | Prismark's PCB market is not the addressable market for a hackathon tool. | Use it as context, not as a TAM claim. |
| GitHub counts | Stars/forks are date-stamped and change daily. | Say "at the time of writing". |
| Ada production readiness | Research/hackathon stage; 1 star; partial router. | Do not present Ada as a production alternative; use it as prior art. |

---

## 11. Sources

All URLs accessed 27 October 2026.

**Direct products and pricing**

- Flux: https://www.flux.ai/ · https://www.flux.ai/p/pricing
- Cherry Blossom: https://www.trycherryblossom.com/ · https://www.trycherryblossom.com/pricing · https://www.trycherryblossom.com/open-standards · https://www.trycherryblossom.com/use-cases/prompt-to-pcb · https://github.com/cherryblossomhq
- Trace: https://buildwithtrace.com/ · https://buildwithtrace.com/pricing
- Diode: https://www.diode.computer/ · https://build.diode.computer/
- Copperhead: https://copperhead.sh/ · https://copperhead.sh/pricing/ · https://github.com/copperheadhq/copperhead
- Ada / Silkscreen: https://github.com/machmoon/Ada
- JITX: https://www.jitx.com/product · https://www.jitx.com/plans
- Circuit Mind: https://www.circuitmind.io/product
- Celus: https://www.celus.io/
- Quilter: https://www.quilter.ai/product · https://www.quilter.ai/pricing
- DeepPCB: https://deeppcb.ai/ · https://deeppcb.ai/pricing/
- SnapMagic / SnapEDA: https://www.snapmagic.com/ · https://www.snapeda.com/pricing/
- EasyEDA + JLCPCB: https://jlcpcb.com/blog/ai-pcb-design-easyeda
- tscircuit: https://tscircuit.com/
- Seeed CodeCraft: https://www.seeedstudio.com/blog/2026/07/02/codecraft-is-here-turn-words-into-working-hardware-in-minutes/ (HTTP 403)
- Cadence Allegro X AI: https://www.cadence.com/en_US/home/tools/pcb-design-and-analysis/allegro-x-design-platform.html
- Atopile: https://github.com/atopile/atopile · https://github.com/atopile/atopile/blob/main/.claude/skills/agent/SKILL.md · https://docs.atopile.io/

**Tools referenced**

- KiCad (open-source PCB editor): https://www.kicad.org/

**Market, labor, and language data**

- Prismark: https://www.prismark.com/what-s-new
- US Bureau of Labor Statistics: https://www.bls.gov/ooh/architecture-and-engineering/electrical-and-electronics-engineers.htm
- Organisation internationale de la Francophonie (OIF), 2026 report: https://www.francophonie.org/lancement-du-rapport-la-langue-francaise-dans-le-monde-2026-8426
