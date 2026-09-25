# AffordCraft

Code, input lists and evaluation for the submission *AffordCraft: Scalable Construction of Task-Ready Simulation Assets
from Single Images*.

AffordCraft builds a simulation-ready articulated asset from one RGB photograph and a task instruction by **retrieval
and adaptation** instead of generation. It grounds the object and the part the task operates, retrieves library entries
whose recorded bodies and joints can realize the requested motion, lets a multimodal model judge their compatibility (or
abstain), adapts the chosen entry with its bodies and joints intact, and admits the asset only if it satisfies a
physical contract in simulation.

## Method

| Stage | What happens | Code |
|---|---|---|
| 1. Grounding | Qwen3-VL-8B-Instruct reads the image and the instruction and returns the object box, category, operated part, motion family (rigid, revolute, prismatic, unknown) and a metric size estimate with its confidence | `affordcraft/vision.py` (`ground`) |
| 2. Retrieval | DINOv2 cosine similarity between the grounded crop and the catalog previews; up to 20 distinct entries, at most 12 of the requested category and the rest category-blind | `affordcraft/catalog.py` (`CatalogIndex.retrieve`) |
| 3. Selection | Qwen3-VL judges category, mechanism and subtype compatibility, confidence and geometry similarity for five candidates at a time; a candidate is eligible only if all three are compatible, confidence >= 0.55 and similarity >= 0.20; whole batches may be rejected | `affordcraft/vision.py` (`rank`), `affordcraft/search.py`, `affordcraft/contracts.py` |
| 4. Adaptation | source-preserving import of every link and joint; uniform scale from the size estimate when its confidence is >= 0.65; joint origins, axes and prismatic limits transformed with the asset; audited collision decomposition cascade (CoACD, five levels); uniform density 500 kg/m3 where an entry has no mass; a fixed mount only where the source dataset declares an installation | `affordcraft/build.py`, `affordcraft/source_parser.py`, `scripts/materialize_asset.py` |
| 5. Construction check | the physical contract in Isaac Sim 5.1 (PhysX) in its own process; a failed candidate gets at most one structure-preserving repair (collision rebuilt from the source surface), then the next eligible candidate, at most 20 per input | `affordcraft/search.py`, `affordcraft/backend.py`, `scripts/physics_worker.py` |
| 6. Final validation | the selected asset is reloaded in an independent physics process and checked under the same contract; the verdict is final | `affordcraft/physics.py`, `scripts/physics_worker.py` |

The physical contract admits an asset only if it exports to USD, every dynamic body has a collision shape and a valid
positive inertia, it settles under gravity without interpenetration, a free asset rests on real support contact (a
mounted one keeps a valid anchor while its operated part still moves), and it exposes a joint of the requested motion
family at the operated part (`affordcraft/physics.py`, `affordcraft/contracts.py`: 360 steps at 1/120 s).

## Repository layout

```
affordcraft/     the method (grounding, retrieval, selection, adaptation, physical contract)
scripts/         entry points: visual index, inputs, calibration and freezing, runs, shard merging, scoring
tests/           CPU unit tests of the method
data/inputs/     the 2,000 photographs, the 200-input subset and the 50 cluttered images (public image ids and hashes)
data/library/    the 11,372 library entries (collection, source object id, category, motion)
docs/            setup, data and catalog schema, how to run each experiment, provenance of the code
evaluation/      common physical gate for external methods, aggregation, clean-GPU timing, API cost
baselines/       the seven external image-to-asset routes run under the same protocol
policy/          manipulation tasks built from constructed assets (ten composed scenes, ten single-asset tasks)
analysis/        generators of every table and data figure, with the recorded result summaries they read
requirements/    pinned environments
```

## Getting started

```bash
python -m unittest discover -s tests          # CPU only; needs numpy, scipy, trimesh, CoACD (requirements/runtime.txt)
python analysis/tables/tables_comparison.py   # regenerates Table 1 from the recorded results in analysis/results/
```

Then follow `docs/setup.md` (environments, models), `docs/data.md` (inputs, library, catalog schema) and
`docs/running.md` (index, calibration, the single-object, one-factor and cluttered-image experiments).

## Paper results and the code behind them

| Result | Produced by | Tables and figures from recorded results |
|---|---|---|
| Single-photograph construction, AffordCraft and seven external routes (Table 1, resource table) | `scripts/run_pipeline.py`; `baselines/`; `evaluation/gate`, `evaluation/aggregate`, `evaluation/timing`, `evaluation/api_cost` | `analysis/tables/tables_comparison.py` |
| Paired one-factor study (Table 2) | `scripts/run_pipeline.py` with the ten conditions (`docs/running.md`, step 5) | `analysis/ablation_task_fidelity.py`, `analysis/tables/tables_ablation.py` |
| Outcomes of all 2,000 inputs, per-category pass rates | `scripts/run_pipeline.py`, `scripts/merge_shards.py` | `analysis/figures/fig4_fig8.py` |
| Cluttered images (50 COCO images, 237 instances) | `scripts/detect_scene_objects.py`, `scripts/run_pipeline.py --scene-mode`, `scripts/evaluate_scenes.py` | `analysis/tables/tables_comparison.py` |
| Library growth (141 to 11,372 entries) | recorded library-scaling runs | `analysis/figures/fig6_library_compact.py`, `analysis/tables/make_tables.py` |
| Manipulation in composed scenes and single-asset tasks | `policy/scenes`, `policy/single_asset` | `analysis/tables/tables_scenes.py`, `analysis/tables/tables_exp3.py`, `analysis/tables/tables_appendix.py` |
| Accepted assets against the time budget | `evaluation/timing` | `analysis/figures/fig_time_to_pass.py` |

`analysis/README.md` lists every output file and its inputs; `docs/provenance.md` maps each published file to the frozen
file that ran, with its hash.

## Protocol

Every method receives the whole photograph with no reference box or mask. AffordCraft, Articulate-Anything and the
agent also receive the task instruction, which names the category; PAct receives an automatic part mask. Missing
exports count as failures, so every input stays in the denominator. External outputs are scored by the same gate twice:
as delivered (native) and after AffordCraft's common physical adaptation (adapted); see `evaluation/README.md`.

## License

The code in this repository is released under the MIT License (`LICENSE`). The input lists and the library index refer
to third-party collections that keep their own licenses and terms: Open Images V7, COCO, PartNet-Mobility (SAPIEN),
Objaverse, Google Scanned Objects and YCB. External methods are used through their official releases under their own
licenses (`baselines/`).
