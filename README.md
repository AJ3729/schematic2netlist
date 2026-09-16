# schematic2netlist

**A pin-aware benchmark for schematic-to-SPICE reconstruction**, plus the
deterministic pipeline it was built to measure.

Hand-drawn schematics are how analog circuits are first designed, taught and
discussed, yet they are disconnected from simulation: recreating each sketch in
EDA software is slow, and a single mistranscribed terminal produces a deck that
simulates cleanly and describes the wrong circuit.

The obstacle is not recognizing the symbols. That is nearly solved. The
obstacle is recovering the *wiring*, and there has been no public ground truth
on which to measure it. This repository releases that ground truth, and a
metric that measures the part every existing metric throws away.

## The claim

Metrics over unordered net connectivity canonicalize node labels, because net
names are arbitrary, and in doing so they discard terminal order, which is not
arbitrary. Swap a source's polarity or a diode's anode and cathode and the
netlist describes a different circuit. The metric scores it perfect.

On the 192-circuit held-out split, restricted to circuits whose reference
declares an explicit ground (Rule R, 89 circuits):

| | |
| --- | --- |
| scored perfect by the pin-blind cascade | 80 |
| of those, settling to a **different DC operating point** | **48 (60.0%)** |
| of those 48, identified by the pin-aware criterion | **48 (100%)** |

Read as conditionals, on the same reconstructions:

| when the metric calls a circuit perfect, it simulates correctly | |
| --- | --- |
| pin-blind | **40.0%** (32 of 80), Clopper-Pearson [0.29, 0.52] |
| pin-aware | **26 of 26**, [0.87, 1.00] |

The disagreement is entirely one-sided: 66 circuits accepted by the pin-blind
metric and rejected by the pin-aware one, **zero** the other way (exact McNemar
p = 2.7e-20). The resulting overestimation factor is **2.83x** [2.23, 3.82],
and **2.56x** [2.02, 3.44] after restricting to the 117 test circuits whose
topology does not appear in validation, so it is a property of the metrics
rather than of one split.

The 26 of 26 is perfect observed agreement on a tested subset of 26, not
evidence of universal functional correctness. Its lower bound is 0.87.

## The pin-aware metric

This is the contribution the rest of the repository exists to support, so it
gets its own file rather than a convention buried in prose.

[`spec/pin_symmetry.yaml`](spec/pin_symmetry.yaml) declares, for every
component class, the ordered port names in the order the netlist writer emits
them, and the permutation group of genuinely interchangeable terminals. The
identity permutation is implicit.

```yaml
schema_version: 1
classes:
  Resistor:
    ports: [t0, t1]
    group: [[1, 0]]        # bilateral
  Diode:
    ports: [anode, cathode]
    group: []              # asymmetric
  BJT-NPN:
    ports: [collector, base, emitter]
    group: []              # C and E are not interchangeable
```

A component scores correct when its port-to-net assignment matches the
reference under *some* permutation in `group`. So `group: []` means order
matters exactly, and `[[1, 0]]` on a two-port class means the terminals are
interchangeable. Pin-blind metrics are the same construction with `group` set
to the full symmetric group for every class, and the argument of the paper is
that the substitution is not free.

**Every ruling in that file is contestable, which is the point of shipping it
as a file.** The MOSFET drain/source ruling is the live one: we rule them
asymmetric, because a three-terminal symbol does not bring the bulk out
separately, which implies it is tied to the source and creates a body diode.
That affects 45 of 218 decidable multi-terminal devices. A reader who
disagrees changes one line and re-scores, without touching the scorer:

```bash
# flip the ruling, then re-score the reported split
$EDITOR spec/pin_symmetry.yaml     # MOSFET-N: group: [[0, 2, 1]]
python scripts/regen_on_split.py --split test --fill-caches
python scripts/make_paper_tables.py
```

## Contributions

**C1 — a formally specified pin-aware metric**, with pseudocode in the
manuscript and a versioned machine-readable symmetry specification here. Its
verdict of correct survives simulation for 26 of 26 circuits ([0.87, 1.00])
against 40.0% for the pin-blind cascade.

**C2 — a measurement of how much pin-blind metrics overstate.** 60.0% of
topologically perfect reconstructions simulate to a different operating point
and the pin-aware metric identifies every one, an overestimation factor of
2.6 to 2.8x that persists on a topology-disjoint subsample.

**C3 — a connectivity ground truth for Digitize-HCD.** 192 circuits, 2,564
components, 5,090 terminals, 1,496 nets, 2,047 intersection sites individually
adjudicated. Every net traced from `null`, never bootstrapped from pipeline
output, recording the topology *visibly drawn* rather than a corrected circuit:
a floating node stays floating, a missing ground stays missing. The per-image
decision record is released, so each netlist is reproducible from its decisions
rather than taken on trust. An independent second pass over 58 circuits agrees
at **0.971 net F1** on the 20-circuit random stratum ([0.948, 0.991]) and on
**6.4%** of ordered terminal lists, which is the asymmetry of this paper in
human terms.

**C4 — a metric cascade with its own controls**, validated by null,
perturbation, ambiguity and passive controls. The passive control is the one
worth checking: the identical machinery on resistors, capacitors and inductors,
whose terminal order carries no meaning, must detect exactly zero, and does,
0 of 665, under every condition and their union.

**C5 — a deterministic pipeline as the object of study**, whose learned port
head raises pin-order accuracy from 0.6972 to 0.8945 while leaving all five
pin-blind metrics **bit-identical**. That invariant is the cleanest available
demonstration of what those metrics cannot see.

## Pipeline results

On the 192 held-out circuits that no parameter was ever selected on, seed 0:

| metric | |
| --- | --- |
| strict success, **pin-blind** | **0.5312** (102 of 192) [0.4583, 0.5990] |
| strict success, **pin-aware** | **0.1875** (36 of 192) [0.1354, 0.2448] |
| net F1 | **0.8878** |
| terminal-pair F1 | **0.8217** |
| per-component connected accuracy | **0.6637** |
| normalized graph edit distance (lower is better) | **0.1723** |
| SPICE valid | **0.9948** |
| DC-solvable, before → after ledgered repair | 0.5312 → **0.7344** (2.46 declared assumptions per circuit) |

Over three detector seeds, pin-blind strict success is 0.5365 +/- 0.0138 and
detection reaches mAP@0.5 0.9910 +/- 0.0004 and mAP@0.5:0.95 0.7603 +/- 0.0005.

Every parameter was selected on a *separate* 190-image validation split.
Running the identical configuration on both shows no out-of-sample degradation,
and the two splits are matched on every difficulty proxy measured. See
[`results/split_swap/val_vs_test.json`](results/split_swap/val_vs_test.json),
regenerated by `scripts/compare_splits.py`.

## Three defects in our own protocol

Two were removed. The third can be measured but not removed.

**Split-role contamination.** Parameter selection had been performed against
the split being reported. The roles were exchanged: the 190 tuned images became
validation, and a 192-image split that never entered selection became the
reported test split.

**Detector early-stopping leakage.** The original detector was early-stopped on
images that later became test, a test-minus-val gap of **+0.0169 mAP@0.5 /
+0.0231 mAP@0.5:0.95**. On 2026-08-05 it was retrained from a packet containing
only `train` and `val`, with a guard asserting the test images' absence before
any weight loaded. The replacement's gap is **+0.0088 / +0.0033**, and test
accuracy did not fall: mAP@0.5:0.95 improved from 0.7309 to 0.7603 +/- 0.0005
across three seeds. The old weights are kept only so the contamination
measurement stays reproducible, and must never produce a reported number. See
`REPRODUCE.md` section 0.

**Repeated topologies across the split boundary.** Digitize-HCD is drawn by 176
volunteers from a shared set of teaching circuits and carries no circuit
identifier, so an image-level split cannot avoid scattering repeats. A
Weisfeiler-Lehman hash over the 383 traced images finds only 255 distinct
topologies, and **75 of 192 test circuits (39.1%)** share a topology with a
validation circuit. The 895 training images have no connectivity ground truth,
so that boundary cannot be tested and we claim neither disjointness nor its
opposite. Its effect on the central comparison is *measured*, not deferred to a
limitation: restricting to the 117 topology-disjoint circuits moves absolute
accuracy by 21 points and the overestimation factor from 2.83x to 2.56x, while
leaving the direction untouched at zero pin-aware-only circuits on either set.

## Quick start

```bash
python -m venv venv && source venv/bin/activate
pip install -e '.[dev]'
```

`ngspice` must be on PATH (`brew install ngspice` on macOS).

```bash
python scripts/run_pipeline.py --image data/cleaned_1024/circuit_1199.jpg
```

Reproduce the entire result set, then regenerate every table, macro and figure
from it:

```bash
python scripts/regen_on_split.py --split test --fill-caches
python scripts/make_paper_tables.py
python scripts/make_paper_figures.py
```

## Where the numbers are

| you want | look at |
| --- | --- |
| the symmetry ruling for every component class | [`spec/pin_symmetry.yaml`](spec/pin_symmetry.yaml) |
| the exact command behind every reported number | [REPRODUCE.md](REPRODUCE.md) |
| which artifact backs which table or figure | [results/README.md](results/README.md) |
| dataset provenance, splits, ground-truth format | [data/README.md](data/README.md) |
| how the ground truth was verified, and what it cost | [docs/GT_VAL_VERIFICATION_REPORT.md](docs/GT_VAL_VERIFICATION_REPORT.md) |
| the manuscript | [paper/](paper/) |

**No number in the manuscript is hand-typed.** `scripts/make_paper_tables.py`
generates every table and every `\newcommand` macro from committed `results/`
artifacts, and `scripts/audit_paper_numbers.py` fails if a literal result value
appears anywhere in the LaTeX.

## Splits

Three frozen splits (seed 0, stratified by component-count tertile by rarest
class present): **train 895 / val 190 / test 192**.

The two evaluation splits **exchanged names on 2026-08-03**. Every tuned
parameter had been selected on the 190, which made any figure reported there
in-sample; the 192 never entered selection and now carry the fully verified
ground truth. No image moved between splits, only the labels did.

Consequently: **sweep with `--split val`, report with `--split test`.** Every
script takes `--split` with a default set by its role, and exploratory and
selection scripts default to `val`, so nothing can leak into a reported number
by omission (see
[`src/schematic2netlist/splits.py`](src/schematic2netlist/splits.py)). Anything
under `results/` dated before the swap is a validation number whatever its
filename says.

## Repository layout

```
spec/pin_symmetry.yaml      the symmetry ruling per class; the C1 artifact
configs/default.yaml        every threshold, documented; nothing is hardcoded
src/schematic2netlist/      the installable package
  preprocess.py             deskew / shadow / binarize / crop / resize
  detect.py                 local Ultralytics, Roboflow, or per-image cache
  textmask.py               heuristic text masking (ablation axis)
  wires.py                  non-wire masking + wire extraction
  nodes.py                  connected-component node inference
  snapping.py               terminal snapping: boundary | ports | legacy v1/v2
  ports.py                  port templates and the learned port head
  netlist.py                node naming + SPICE export
  erc.py, repair.py         electrical rule checks + ledgered minimal repair
  metrics.py, benchmark.py  metric cascade, alignment, bootstrap CIs
  splits.py                 which split a script reads, and why
  pipeline.py               per-image orchestration
  determinism.py            seeding + run metadata (config, git SHA, env)
scripts/                    CLIs; REPRODUCE.md says what each produces
paper/                      IEEE Access manuscript, generated tables, figures
results/                    committed summaries and per-image CSVs
tests/                      pytest
docs/                       ground-truth verification report; dev notes
data/                       gitignored except splits/, the test GT, and README
```

## Reproducibility

The pipeline is byte-identical across five independent runs on the same 192
images with fresh interpreters and distinct `PYTHONHASHSEED`: exact-output and
topology agreement 1.0000, zero circuits changing topology. That is a control
on every other number here rather than a finding, and it is recorded so the
frontier-model comparison is read against a measured baseline.

- Every run directory holds a `run_meta.json` with the full config, git SHA,
  seed and environment versions. That is what lets
  `scripts/regen_on_split.py` replay a historical ablation arm exactly rather
  than approximately.
- `seed` in the config seeds `random`, `numpy` and `torch`/Ultralytics.
- Configuration comparisons use a **paired** per-image bootstrap
  (`scripts/compare_runs.py`), not independent-CI overlap. The val-vs-test
  comparison is necessarily unpaired, and says so.
- Standing guards: frame-size guard (`schematic2netlist.frames`),
  detection-cache alignment (`scripts/check_cache_alignment.py`), data
  freshness (`scripts/audit_data_freshness.py`), and the no-hand-typed-numbers
  audit.

What a reader cannot regenerate: re-running the pipeline needs the corpora,
detector training needs a GPU (weights are released, so no result requires
retraining), the frontier-model anchor needs paid API access, and extending the
second annotation needs a person.

## Data and license

Images are [Digitize-HCD](https://doi.org/10.17632/rngcz5wtv8) (Mendeley Data,
CC BY 4.0); cross-dataset material is
[CGHD](https://zenodo.org/records/10056817) (CC BY 4.0). Both remain under
their original licenses. The code here is MIT ([LICENSE](LICENSE)); the
connectivity ground truth, the symmetry specification and the split manifests
are released with it as the benchmark contribution.
