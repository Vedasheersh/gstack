# proteinstack

proteinstack is a Claude Code skill toolkit for computational protein designers.
It is a fork of gstack, adapted for protein design workflows.

## Commands

```bash
bun install          # install dependencies
bun run build        # gen docs + compile binaries
bun run gen:skill-docs  # regenerate SKILL.md files from templates
bun run skill:check  # health dashboard for all skills
```

## Project structure

```
proteinstack/
├── design-binder/       # /design-binder skill — full binder design pipeline
├── analyze-structure/   # /analyze-structure skill — target characterization
├── validate-design/     # /validate-design skill — fold + score a sequence
├── setup-protein-env/   # /setup-protein-env skill — one-time tool discovery
├── backends/            # per-backend knowledge templates
│   ├── rfdiffusion.md
│   ├── bindcraft.md
│   ├── pxdesign.md
│   └── _base.md
├── browse/              # headless browser (inherited from gstack)
├── bin/                 # CLI utilities
├── setup                # one-time setup script
└── CLAUDE.md            # this file — the domain brain
```

---

# Protein Design Domain Brain

This section is the most important part of this file. It makes Claude reason like
a senior computational protein designer, not a generic AI assistant.

**Always apply this knowledge** when working in this repo or when any proteinstack
skill is active. Do not require the user to re-explain protein design concepts.

---

## Amino Acid Reference

Single-letter / three-letter codes and key properties:

| 1L | 3L  | Property |
|----|-----|----------|
| A  | ALA | Small, hydrophobic, helix-favoring |
| C  | CYS | Thiol side chain; disulfide bonds; reactive |
| D  | ASP | Negatively charged (acidic) at pH 7 |
| E  | GLU | Negatively charged (acidic) at pH 7 |
| F  | PHE | Aromatic, hydrophobic, bulky |
| G  | GLY | No side chain; flexible; helix-breaking |
| H  | HIS | Positively charged at low pH; metal coordination |
| I  | ILE | Branched hydrophobic; beta-sheet favoring |
| K  | LYS | Positively charged (basic) |
| L  | LEU | Hydrophobic; helix-favoring |
| M  | MET | Hydrophobic; Met-start for expression |
| N  | ASN | Polar uncharged; N-glycosylation site |
| P  | PRO | Rigid; helix-breaking; turns |
| Q  | GLN | Polar uncharged |
| R  | ARG | Positively charged; H-bond donor (3 NH groups) |
| S  | SER | Polar; phosphorylation site |
| T  | THR | Polar; beta-branched |
| V  | VAL | Branched hydrophobic; beta-sheet favoring |
| W  | TRP | Largest side chain; aromatic; membrane-anchoring |
| Y  | TYR | Aromatic; H-bond donor/acceptor; phosphorylation |

---

## File Formats

**PDB (.pdb):**
- ATOM/HETATM records: columns are atom serial, name, residue name, chain ID,
  residue seq number, insertion code, X/Y/Z coordinates, occupancy, B-factor
- Chain IDs matter: A, B, C... Target is usually A, binder is usually B (or A if monomer)
- Auth numbering vs seq numbering: PDB uses author-assigned numbers that may have gaps
  or insertions (e.g., "47A"). Structural tools use 0-indexed or 1-indexed seq numbering.
- HETATM: non-polymer atoms (ligands, waters, ions). Ligands may be relevant for
  active site design.
- Multiple MODEL records: NMR ensembles. Use MODEL 1 unless told otherwise.

**mmCIF (.cif):**
- Modern replacement for PDB format. Handles large structures better.
- Required for structures > 99,999 atoms.

**FASTA (.fasta / .fa):**
- `>header\nSEQUENCE` — single-letter amino acid codes, 60-80 chars per line
- Multiple sequences: separate with `>header` lines
- For binder design output: one FASTA per candidate or all candidates in one file

**A3M (.a3m):**
- Multiple sequence alignment format used by AlphaFold2 for evolutionary context
- First sequence is the query; subsequent lines are aligned homologs
- Gaps: `-` (aligned gap), lowercase (inserted residues)

**NPZ / pickle:**
- NumPy compressed arrays used by RFdiffusion for internal representations
- Not human-readable; pass directly to tools

---

## Score Interpretation

### AlphaFold2 / ESMFold / Boltz

**pLDDT** (per-residue confidence, 0–100):
- >90: Very high confidence. Trust the structure.
- 70–90: Confident. Minor uncertainties in loops/termini.
- 50–70: Low confidence. Likely disordered or flexible.
- <50: Very low. Residues are probably disordered. Do not interpret structurally.
- For binders: require >80 average pLDDT, especially at the interface.

**ipTM** (interface TM-score, 0–1, multimer only):
- >0.75: Strong predicted interface. High confidence the complex is real.
- 0.6–0.75: Reasonable. Worth keeping for experimental testing.
- <0.6: Weak predicted interface. Usually reject.
- Standard Baker lab cutoff: ipTM > 0.7 combined with pLDDT > 80.

**pTM** (whole-complex TM-score, 0–1):
- Overall fold quality. Less informative than ipTM for binder design.

**pAE** (predicted aligned error, matrix, Å):
- Lower = more confident relative positioning between residues.
- **Interface pAE**: average pAE between binder and target residues — the key metric.
  Interface pAE < 10 Å is good; < 5 Å is excellent.
- High pAE within the binder = uncertain binder structure.
- High pAE across the interface = uncertain binding mode.

### ProteinMPNN

**Sequence score** (negative log-likelihood, more negative = better fit to backbone):
- Typical range: -1.0 to -2.5 for well-designed sequences.
- Score < -1.5: sequence fits the backbone well.
- Score > -0.8: sequence may be poorly matched; consider resampling.
- Very negative scores (<-3.0): possible overfitting; check sequence complexity.

**Recovery**: fraction of native-like residues. High recovery on interface = tight packing.

### RFdiffusion

**Trajectory loss**: reported during diffusion. Lower final loss = better convergence.
Not directly comparable across runs with different parameters.

**Self-consistency pLDDT** (after MPNN + AF2 validation):
The true quality signal. A backbone that produces high self-consistency pLDDT means
the designed sequence reliably folds to the intended structure.

### BindCraft

Uses AF2 multimer internally. Reports:
- ipTM, pLDDT, pAE — same interpretation as above.
- **i_pAE**: interface pAE specifically.
- **binder_score**: composite metric; higher = better binder candidate.
- **pass/fail**: BindCraft applies its own filters; respect these unless you have reason not to.

---

## Interface Heuristics

**Buried surface area (BSA):**
- < 500 Å²: very weak; probably not a binder.
- 600–800 Å²: weak but measurable (μM range Kd).
- 800–1200 Å²: typical small protein binder (nM range).
- > 1200 Å²: large interface; usually strong binders.
- Calculate with: `freesasa`, `naccess`, or from AF2 output.

**Shape complementarity (Sc):**
- 0.5–0.6: moderate fit.
- > 0.65: good shape match (antibody-antigen interfaces are ~0.65).
- < 0.5: poor fit; binder and target have mismatched surfaces.

**H-bonds at interface:**
- More H-bonds = more specific, more stable.
- Buried H-bonds (desolvated) are especially stabilizing.
- Unsatisfied buried polar atoms are energetically costly — flag these.

**Van der Waals clashes:**
- Any clash < 0.4 Å contact distance between non-bonded atoms is a hard reject.
- Check with: `clashscore` in MolProbity, or `check_structure.py` from BioPython.

**Electrostatics:**
- Complementary charges at interface: good.
- Like charges facing each other: repulsive — bad for binding.
- Salt bridges (E/D ↔ K/R): stabilizing but pH-sensitive.

---

## Common Failure Modes

**Aggregation / poor expression:**
- Hydrophobic patches on the solvent-exposed surface of the binder.
  Flag any patch > 6 consecutive hydrophobic residues on the surface.
- Low-complexity sequences (e.g., AAAAAAA, GGGGG runs) → poor expression.
- Charged N-terminus (non-Met start) → ribosome issues in bacterial expression.
- Missing Met at N-terminus for bacterial expression.
- Too many Cys residues without designed disulfides → random disulfide bonds.

**Hallucinated structure (common in diffusion models):**
- High RFdiffusion confidence but low self-consistency pLDDT after MPNN + AF2.
  This means the diffused backbone cannot actually be folded by a real sequence.
- Flag when: self-consistency pLDDT < 70, or the AF2 structure differs from the
  designed backbone by RMSD > 2 Å.

**Poor interface:**
- High binder pLDDT but low ipTM → binder folds well but doesn't bind the target.
- Large BSA but low shape complementarity → steric fit without true complementarity.
- MPNN placed mostly Gly/Ala at the interface → no specific contacts.

**Target flexibility:**
- Designing against a disordered loop or flexible region → low reproducibility.
  Flag when the target region has B-factors > 50 in the crystal structure, or
  pLDDT < 70 in the AF2 prediction.

**Linker problems (for multi-domain designs):**
- Too rigid (no Gly/Ser) → strain between domains.
- Too flexible (all Gly/Ser, >12 residues) → poor AF2 ipTM.
- Standard linker: (GGGGS)n or (EAAAK)n for rigid connections.

---

## Backend Reference

### RFdiffusion3 (preferred for most tasks)

The current state of the art for backbone generation. All-atom, handles DNA/RNA/ligands.
10x faster than RFdiffusion2.

**Key parameters:**
```yaml
contigmap:
  contigs: ["A1-150/0 B1-80"]   # target chain A, binder chain B (80 residues)
  inpaint_seq: ["B1-80"]          # design sequence for chain B
hotspot_res: ["A47", "A51", "A89"]  # residues binder must contact
diffuser:
  T: 50                           # timesteps (50 = default)
  partial_T: 10                   # partial diffusion (motif scaffolding)
inference:
  num_designs: 100               # number of backbone samples
  noise_scale_ca: 1.0            # noise level (higher = more diversity)
  noise_scale_frame: 1.0
```

**Contigmap syntax:**
- `"A1-150"` = keep residues 1–150 of chain A fixed (target)
- `"B1-80"` = design 80 residues for chain B (binder)
- `"0"` = chain break
- `"/0 "` separates chains in the contig string
- `"50-100"` = design a chain of length 50–100 (variable length)

**Output:** backbone PDB files (no sequence — just backbone atoms). Feed to ProteinMPNN.

### ProteinMPNN / FAMPNN

Designs sequences for a given backbone.

**Key parameters:**
```bash
--pdb_path <backbone.pdb>
--chain_id_jsonl <chains.json>       # which chains to design
--fixed_positions_jsonl <fixed.json> # residues to keep fixed (e.g., hotspot contact residues)
--num_seq_per_target 8               # sequences per backbone (8 is standard; use 16+ for hard targets)
--sampling_temp 0.1                  # 0.1 = conservative, 0.3 = diverse, 0.5 = highly diverse
--seed 37
--out_folder <output_dir>
```

**When to use FAMPNN:** Same API as MPNN but full-atom aware. Use for ligand-binding
designs or when side-chain placement matters at design time.

**Output:** FASTA file + scores file (log-likelihood per sequence).

### AlphaFold2 (for validation)

**Multimer mode** for interface validation (required for binder assessment):
```bash
--fasta_paths=binder_target.fasta   # both chains in one FASTA, separated by colon or as multimer
--model_preset=multimer
--max_template_date=2020-01-01      # prevent template leakage for de novo designs
```

**Key output files:**
- `ranked_*.pdb`: structures ranked by confidence
- `ranking_debug.json`: ipTM, pTM, pLDDT scores for each model
- `result_model_*.pkl`: full prediction outputs including pAE matrices

**ESMFold** (faster, single-sequence, good for screening):
```bash
python esm/scripts/fold.py -i sequences.fasta -o output_dir/ --num-recycles 4
```
Use ESMFold for initial screening (speed), AF2 multimer for final validation (accuracy).

### BindCraft

Best for smaller binders (< 100 residues) and when you want an end-to-end pipeline.
Internally runs AF2 multimer for scoring — no separate validation step needed.

**Config file (YAML):**
```yaml
target_pdb: target.pdb
target_hotspot_res: "A47,A51,A89"
binder_len: 60                   # residue count
num_designs: 50
filters:
  plddt: 80
  iptm: 0.6
  i_pae: 15
```

**Output:** `final_designs/` folder with ranked PDBs, `results.csv` with all scores.

### PXDesign

Use for: [fill in when configured — run /setup-protein-env to detect installation]

---

## Design Vocabulary

| Term | Meaning |
|------|---------|
| **Hotspot residues** | Target residues the binder must physically contact (H-bond, VdW) |
| **Scaffold** | The structural framework the binder is built on (e.g., a helical bundle) |
| **Motif grafting** | Transplanting a binding epitope onto a new backbone scaffold |
| **Partial diffusion** | Starting RFdiffusion from a noisy known structure instead of pure noise |
| **De novo design** | Designing from scratch with no template structure |
| **Self-consistency** | MPNN sequence folded back by AF2 gives the original backbone — validates the design |
| **pAE island** | Low-pAE cluster in the pAE matrix indicating a confident interface |
| **Buried unsatisfied polar** | Polar atom at the interface with no H-bond partner — energetically costly |
| **BSA** | Buried surface area — Å² of surface removed from solvent upon complex formation |
| **Sc** | Shape complementarity (0–1); how well the surfaces fit together |
| **ipTM** | Interface TM-score — AF2 multimer's confidence in the binding mode |
| **Nanobody** | Single-domain antibody (~110 AA), good specificity and expression |
| **Miniprotein binder** | Very small (<60 AA) de novo designed binder |
| **Cyclic peptide** | Circular backbone peptide; more stable, cell-permeable |
| **Linker** | Short sequence connecting two protein domains (e.g., GGGGSx3) |
| **Expression tag** | His-tag, SUMO, etc. added for purification — don't design over them |

---

## Standard Acceptance Thresholds

These are the standard cutoffs used in Baker lab publications and the field.
Flag candidates that fail any of these. Override only with explicit justification.

| Metric | Reject if | Keep if | Notes |
|--------|-----------|---------|-------|
| Binder pLDDT | < 70 | > 80 | Average over binder chain |
| ipTM | < 0.55 | > 0.70 | AF2 multimer |
| Interface pAE | > 20 Å | < 10 Å | Average over interface residue pairs |
| MPNN score | > -0.8 | < -1.5 | Log-likelihood |
| BSA | < 400 Å² | > 700 Å² | |
| Self-consistency RMSD | > 2.5 Å | < 1.5 Å | Designed backbone vs AF2 backbone |

---

## Reasoning Like a Senior Protein Designer

When reviewing designs, always ask:

1. **Does the binder actually contact the hotspots?** Check that the hotspot residues
   are within 4.5 Å of binder atoms in the predicted complex.

2. **Is the interface specific or promiscuous?** Specific = complementary shape +
   H-bonds + hydrophobic core. Promiscuous = large BSA but mostly hydrophobic burial
   with no H-bonds.

3. **Will this express?** Check N-terminus (Met?), surface hydrophobics, sequence
   complexity, disulfide count.

4. **Is the binding mode consistent?** Across multiple AF2 models, does the binder
   land in the same place? Inconsistent binding modes = uncertain design.

5. **Does the backbone make sense?** Visualize secondary structure. Is there a
   coherent fold (helices, sheets, loops in sensible arrangement)? Random coil = bad.

6. **What's the failure mode here?** Every design has a most likely failure mode.
   Name it: "this will likely aggregate due to the exposed hydrophobic patch at
   residues 23–28" or "the interface is small — might bind but with weak affinity."

When ranking candidates, don't just sort by ipTM. Explain *why* the top candidate
is better than rank 2. Use the metrics above plus structural reasoning.

---

## Provenance Tracking

Every proteinstack skill writes a `provenance.json` file to the output directory:

```json
{
  "timestamp": "<ISO 8601>",
  "skill": "<skill name>",
  "backend": "<rfdiffusion3|bindcraft|pxdesign>",
  "target_pdb": "<path>",
  "hotspots": ["<chain><resnum>", ...],
  "parameters": { ... },
  "n_designs_generated": 100,
  "n_designs_passed_filters": 12,
  "candidates": [
    {
      "rank": 1,
      "file": "candidate_001.pdb",
      "sequence": "MAEFLY...",
      "plddt": 91.2,
      "iptm": 0.74,
      "interface_pae": 7.3,
      "mpnn_score": -1.83,
      "bsa": 920,
      "verdict": "KEEP",
      "reason": "Strong interface with hotspot contacts at A47, A51. High ipTM + low pAE island."
    }
  ],
  "tool_versions": {
    "rfdiffusion": "3.0.x",
    "proteinmpnn": "1.0.x",
    "alphafold": "2.3.x"
  },
  "proteinstack_version": "<version>"
}
```

Always write this file. It is the seed for cross-run analysis and institutional memory.

---

## Membrane Topology Tools

**PPM** (Positioning of Proteins in Membranes) — Lomize lab, University of Michigan.
The software that powers the OPM database. Use this directly rather than querying OPM.

- PPM 3.0: `ppm3 -f input.pdb` — outputs membrane-oriented PDB in working directory
- PPM 2.0: `ppm -f input.pdb -o output/`
- Output format: REMARK records with hydrophobic thickness + DUM dummy atoms at
  Z = ±(thickness/2) marking membrane core boundaries
- Install: https://opm.phar.umich.edu/ppm_server (standalone download)

Parse membrane boundaries from PPM output:
- `Hydrophobic thickness` REMARK → total bilayer thickness
- DUM HETATM atoms → upper (max Z) and lower (min Z) membrane boundaries
- Extracellular residues: Cα Z > upper_boundary + 3 Å
- TM core: Cα Z between boundaries
- Intracellular: Cα Z < lower_boundary - 3 Å

## Backend Configuration

On first use, run `/setup-protein-env`. It writes `.proteinstack/config.yaml`:

```yaml
backends:
  rfdiffusion3:
    enabled: true
    path: /path/to/RFdiffusion
    weights: /path/to/weights/
    gpu: true
  bindcraft:
    enabled: false
    path: null
  pxdesign:
    enabled: false
    path: null
mpnn:
  path: /path/to/ProteinMPNN
  version: "1.0.1"
alphafold:
  enabled: true
  path: /path/to/alphafold
  db_path: /path/to/databases
esmfold:
  enabled: true
  path: /path/to/esm
gpu:
  available: true
  device: "cuda:0"
```

Skills read this config to determine which backend to use. If a requested backend
is not configured, the skill will offer alternatives or prompt to run
`/setup-protein-env` first.
