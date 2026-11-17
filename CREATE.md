# SWAT+ 62.0.0 – CREATE common build

One SWAT+ executable for all CREATE case studies.

- **First commit:** the official SWAT+ rev **62.0.0**, unmodified. It is byte-identical to [swat-model/swatplus](https://github.com/swat-model/swatplus) tag `62.0.0` (commit `de210d6`).
- **Later commits:** the CREATE fixes listed below, one commit per fix.
- **Compiler:** Intel classic `ifort` on Windows, the compiler behind the official `swatplus-…-ifo-win_amd64-Rel.exe` that SWAT+ Editor ships as its default. gfortran is not used.

The exe writes its screen output to the console like the official one, so no `fort.6` file appears and SWATdoctR works without the `fort.6` patch.

To see every change against the official source: `git diff <first commit> HEAD -- src`.

## Versions

| Release | igrow fix: perennial start day | Status |
|---|---|---|
| `v62.0.0-create.2` | 0.15 of the base-0 heat units (same day as annual crops) | **current, use this** |
| `v62.0.0-create.1` | end of dormancy (day length) | superseded: do not calibrate perennials with `days_mat > 0` with it |

Both releases contain the same LAI fix, build fix and banner. They differ only in the igrow start day.

**Why create.2.** An A/B test on CREATE_PL (15,136 HRUs, 2000–2005) compared three exes: the original 62.0.0, the end-of-dormancy start and the 0.15 heat-unit start.
- **create.1** removed the HRU-order dependence, but made every `fesc` meadow stunted: PHU at harvest about 10, no regrowth after the first cut, yield −40%.
- **The 0.15 heat-unit start** also removes the order dependence, and gives every meadow the behaviour of the original's normal group: PHU at harvest about 0.5, normal regrowth. The formerly stunted HRUs 3804–5907 now reach 12,000 instead of 1,400 kg/ha max. biomass. `fesc_mgt` keeps its original results.

Both exes print `Revision 62.0.0 - CREATE`. Use `BUILD_INFO.txt` or the release tag to tell them apart.

## Changes to the model source

| # | File | Problem in 62.0.0 | Fix | Upstream |
|---|------|-------------------|-----|----------|
| 1 | `src/pl_dormant.f90` | `rto` is hard-coded to 0 and the `alai_min` floor was removed, so LAI of **every perennial** (evergreens too) drops to exactly 0 at dormancy. | One line after the LAI reduction: `lai = max(lai, min(lai_init, alai_min))`. LAI stays at `lai_min` from `plants.plt`, as in rev 61, and is never raised above its pre-dormancy value. Biomass handling unchanged. (Priya, 2026-11-03.) | [#245](https://github.com/swat-model/swatplus/issues/245) (open) |
| 2 | `src/plant_init.f90` | `igrow` has an initialiser, so it is `SAVE`d between calls. For a `perennial` with `days_mat > 0` it is never set, so `phumat` is summed from the planting day of whatever annual crop was initialised before it. Perennial PHU then depends on HRU order (Łupawa: two meadow groups with ~12× different `phumat`). | Reset `igrow = 0` for every plant, and let `perennial` start heat-unit accumulation on the same day as annual crops: when 0.15 of the year's base-0 heat units are reached. (Problem reported by Mikołaj, 2026-11-16. Start-day rule chosen after an A/B test on CREATE_PL, 2026-11-17; see *Versions*.) | not yet reported |

### Build-only change (no effect on results)

| File | Problem | Fix |
|------|---------|-----|
| `src/actions.f90`, `src/hru_control.f90`, `src/mgt_sched.f90` | Their `external ::` lists name `mgt_newtillmix` and `mgt_newtillmix_cswat3`, routines that no longer exist and are never called. gfortran ignores this; Intel ifort on Windows emits a link reference for each name, so linking fails with `LNK2001: unresolved external symbol MGT_NEWTILLMIX` / `MGT_NEWTILLMIX_CSWAT3`. | Remove the two names from the declaration lists. No executable statement changes. For `hru_control.f90` this is the same change upstream made in 62.0.1. |

### Identification (no effect on results)

| File | Change |
|------|--------|
| `src/main.f90.in` | The start-up banner, on screen and in `simulation.out`, reads `Revision 62.0.0 - CREATE`, so every run shows it was made with the CREATE exe. The reference exe keeps the original banner. |

### Known, not changed

From the `igrow` analysis. Neither is triggered by perennials whose `days_mat` ends before day 365:

1. In the final heat-unit loop of `plant_init`, the northern-hemisphere branch calls `xmon(iday)` without wrapping days above 365. `xmon` then leaves the month unchanged.
2. After the dormancy search loop for `null`/`null1`, `igrow = iday` is used even when the loop ended without finding the end of dormancy.

## Build

[`.github/workflows/build-create-ifort-windows.yml`](.github/workflows/build-create-ifort-windows.yml) runs on every push that changes `src/`, when a release is published, and manually (**Actions → Run workflow**, where you can also pick the ifort version).

**Compiler:** Intel classic ifort **2021.6**, the version the official 62.0.1 `ifo` exe reports in its banner. Release build, static Intel runtime, on the `windows-2022` GitHub runner.

Two exes are built with identical compiler and settings:

| Artifact | Source | Use |
|---|---|---|
| `swatplus-62.0.0-create-ifo-win_amd64` | this repository: 62.0.0 + all CREATE changes | **the common CREATE exe** |
| `swatplus-62.0.0-reference-ifo-win_amd64` | original 62.0.0 + build-only commits (subject starts with `Build fix`) | A/B comparison only |

Each artifact contains:

- the exe
- `BUILD_INFO.txt`: source commit, ifort version, exe SHA-256, list of changes
- `changes-vs-62.0.0.diff`: the full source diff against the official 62.0.0
- `smoke_test_Ames_sub1.log`

**Releases.** Publish a release on GitHub (**Releases → Draft a new release**, tag `v62.0.0-create.N` on `main`). The workflow then builds both variants from that tag and attaches to the release:

- both exes
- one zip per variant with the exe, `BUILD_INFO.txt`, the diff and the smoke log

This gives partners a stable download link without a GitHub login.

A build fails if:

- the compiler is not Intel classic ifort;
- the exe needs Intel runtime DLLs (the official exe is statically linked);
- the exe does not start and write its screen output to the console;
- `fort.6` is written.

**About the smoke test.** The test data shipped with the 62.0.0 source (`refdata/Ames_sub1`, `refdata/Osu_1hru`) does not run to completion with any released 62.0.x exe. The official SWAT+ Editor exes, both `ifo` (Intel) and `gnu`, report `corn not found in plants.plt` and stop with an access violation in `soils_init.f90` line 295 (checked 2026-11-17 with `swatplus-62.0.1-ifo/gnu-win_amd64-Rel.exe`). The CREATE exe stops at the same place, so the smoke test only checks that the exe starts, writes to the console and reads its inputs. The real check is the A/B run below.

The upstream workflows (`build.yml`, `doc.yml`, `test.yml`) were removed in this repository because they build with gfortran. They remain in the first commit.

## Checking the fixes in a real project (A/B run)

Run the `create` and the `reference` exe on copies of the same TxtInOut. Both are built identically, so every difference in output comes from the CREATE model fixes:

- **LAI (1):** in `hru_pw_*` / `basin_pw_*` output, perennial LAI no longer drops to 0.000 in winter; it stays at about `lai_min`.
- **igrow (2):** only matters for perennials with `days_mat > 0`. With `fesc` `days_mat` 100–130, `phuplant` in `mgt_out` is about equal for HRU 3804 and HRU 7539 (Łupawa). With `days_mat = 0` everywhere, fix 2 changes nothing.

## Adding a fix

1. Change the file in `src/` and commit it alone. In the message, describe the problem and the fix, and link the upstream issue if there is one.
   - A change that is only needed to compile and does not change results gets a commit subject starting with **`Build fix`**. The reference exe includes it.
2. Add a row to the tables above.
3. Push. The workflow rebuilds. When the case studies should switch to it, publish a release `v62.0.0-create.N`.
