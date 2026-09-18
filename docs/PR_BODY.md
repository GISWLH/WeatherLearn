## Summary

Packaging and test hygiene after a full-library audit of WeatherLearn (surgical diffs only; no OOP rewrite).

- Make `pip install -e .` and `python -m unittest discover` work by adding missing package `__init__.py` files.
- Relax `pyproject.toml` dependency pins to installable `>=` ranges (exact pins broke on Python 3.12/3.13).
- Prefer `timm.layers` (with fallback) to silence the `timm.models.layers` deprecation.
- Make FuXi `depth` configurable (default **48**, backward compatible).
- Skip full `Pangu()` OOM-prone unit test unless `WEATHERLEARN_RUN_HEAVY=1`.
- Document Issue #8 respectfully: last layer is **Linear patch FC + bilinear interpolate** (matches the FuXi paper); discussion had conflated the two steps. See `docs/fuxi_last_layer.md`.

## Test plan

- [x] `pip install -e . --no-deps` (or with relaxed deps)
- [x] `python -c "from weatherlearn.models import Fuxi, Pangu, Pangu_lite"`
- [x] `python -m unittest discover -s tests -v` → 43 tests, 1 skipped (`test_pangu`), OK
- [x] All FuXi tests pass, including new `test_depth_configurable`
- [ ] Optional: `WEATHERLEARN_RUN_HEAVY=1 python -m unittest ... TestMain.test_pangu` on a high-RAM / GPU machine
- [ ] Optional: HF Space buttons — lite / tiny721 smoke + FuXi unit tests

## Notes for reviewers

- License remains **BY-NC-SA 4.0**. No pretrained weight URLs added.
- Larger items (ONNX finetune path cleanup, optional `cdsapi` extra, FengWu, full OOP refactor) intentionally deferred — catalogued in the accompanying audit notes.
- Related discussion: Issue #8.

## 中文说明

本 PR 为「完整复刻」审计后的最小修复：补齐包结构、放宽依赖钉死、FuXi `depth` 可配、重测试默认跳过，并补充 Issue #8（FC vs 插值）与论文一致的说明。不改动作者计划中的整体 OOP 重构。


## Paper alignment (this follow-up)

Branch `fix/paper-alignment` compares WeatherLearn to Pangu-Weather (arXiv:2211.02556) and FuXi (arXiv:2306.12873). Artifacts: `papers/SPECS.md`, `PAPER_DIFF.md`.

### Bugs fixed (surgical)
1. **Pangu window axis order** — paper `Wpl×Wlat×Wlon = 2×12×6`. Official pseudocode uses `(2,6,12)` on `(pl,lon,lat)`; this repo uses `(pl,lat,lon)`, so the default must be `(2,12,6)` (was `(2,6,12)`, lat/lon swapped). `shift_size` now derives as half-window.
2. **`PatchEmbed3D` height padding** — remainder used `l_patch_size` instead of `h_patch_size` (harmless for default `(2,4,4)` @ 721; wrong when `h≠l`).

### Not bugs / scope gaps
- FuXi last layer (Issue #8): **FC + bilinear 720→721** matches paper — not reopened.
- FuXi Short/Medium/Long **cascade** and Pangu **1h/3h/6h/24h** hierarchical inference: system-level, not implemented (documented intentional).
- Patch-embed GeLU: paper text vs official pseudocode disagree → left as ambiguous (follow official: no GeLU).

### Tests
`python -m unittest discover -s tests -v` → **45 tests, 1 skipped** (`test_pangu` heavy), OK.
