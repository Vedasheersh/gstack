# proteinstack

> Claude Code skills for computational protein designers.

**proteinstack** is a fork of [garrytan/gstack](https://github.com/garrytan/gstack), adapted for protein design workflows. It injects deep protein design domain knowledge into Claude Code's context and provides opinionated skills that wrap the standard computational protein design toolkit: RFdiffusion3, ProteinMPNN, BindCraft, AlphaFold2, and ESMFold.

The goal: an AI assistant that reasons like a senior computational designer — not a generic chatbot that needs you to explain what ipTM means.

---

## Install

```bash
git clone https://github.com/Vedasheersh/gstack.git ~/.claude/skills/proteinstack
cd ~/.claude/skills/proteinstack && ./setup
```

Then add to your project's `CLAUDE.md`:

```markdown
## proteinstack

Use proteinstack skills for all protein design workflows.
Available skills: /setup-protein-env, /analyze-structure, /design-binder, /validate-design

Run /setup-protein-env first to configure your tool paths.
```

---

## Skills

| Skill | Description |
|-------|-------------|
| `/setup-protein-env` | Discover installed tools, write `.proteinstack/config.yaml` |
| `/analyze-structure` | Characterize a target PDB: hotspots, surface, B-factors, design recommendations |
| `/design-binder` | Full binder design pipeline: backbone → MPNN → validation → ranked report |
| `/validate-design` | Fold a sequence and score the interface (ESMFold or AF2 multimer) |

Inherited from gstack (all work with protein context):
`/investigate`, `/retro`, `/review`, `/qa`, `/browse`, `/office-hours`, and more.

---

## Supported Backends

| Backend | Use for |
|---------|---------|
| **RFdiffusion3** | De novo binder design, motif scaffolding, large targets |
| **BindCraft** | End-to-end binder design, smaller targets (< 100 AA binder) |
| **PXDesign** | (configure via `/setup-protein-env`) |
| **ProteinMPNN / FAMPNN** | Sequence design for any backbone |
| **ESMFold** | Fast single-sequence validation (screening) |
| **AlphaFold2 multimer** | Accurate interface validation (final candidates) |
| **Boltz** | Alternative structure prediction |

---

## What Makes This Different

- **Domain brain in CLAUDE.md** — Claude knows what pLDDT > 80 means, what a hydrophobic surface patch looks like, and which failure modes are common for each backend. You don't re-explain this every session.
- **Provenance tracking** — every run writes `provenance.json` with full config, tool versions, and per-candidate verdicts with structural reasoning.
- **Backend-agnostic** — use whichever tools you have installed; skills adapt.
- **Reasoning, not dispatch** — the agent explains *why* candidate 3 is better than candidate 11, not just sorts by ipTM.

---

## Roadmap

- [ ] `/validate-design` skill
- [ ] `/retro-protein` — cross-run outcome analysis
- [ ] `/design-enzyme` — active site design (RFdiffusion3 / partial diffusion)
- [ ] `/design-cyclic` — cyclic peptide design
- [ ] `/lab-protocol` — generate bench protocol from design candidates
- [ ] `/design-memory` — institutional memory across runs

---

## Based On

- [garrytan/gstack](https://github.com/garrytan/gstack) — the developer experience model this forks
- [RosettaCommons/RFdiffusion](https://github.com/RosettaCommons/RFdiffusion) — backbone generation
- [dauparas/ProteinMPNN](https://github.com/dauparas/ProteinMPNN) — sequence design
- [martinpacesa/BindCraft](https://github.com/martinpacesa/BindCraft) — end-to-end binder design
- [deepmind/alphafold](https://github.com/deepmind/alphafold) — structure prediction
- [facebookresearch/esm](https://github.com/facebookresearch/esm) — fast structure prediction
