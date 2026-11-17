# SWAT+ 62.0.0 – CREATE common build

One SWAT+ executable for all CREATE case studies.

- **First commit:** the official SWAT+ rev **62.0.0**, unmodified. It is byte-identical to [swat-model/swatplus](https://github.com/swat-model/swatplus) tag `62.0.0` (commit `de210d6`).
- **Later commits:** the CREATE fixes listed below, one commit per fix.
- **Compiler:** Intel classic `ifort` on Windows, the compiler behind the official `swatplus-…-ifo-win_amd64-Rel.exe` that SWAT+ Editor ships as its default. gfortran is not used.

The exe writes its screen output to the console like the official one, so no `fort.6` file appears and SWATdoctR works without the `fort.6` patch.

To see every change against the official source: `git diff <first commit> HEAD -- src`.

## Changes to the model source

| # | File | Problem in 62.0.0 | Fix | Upstream |
|---|------|-------------------|-----|----------|
| 1 | `src/pl_dormant.f90` | `rto` is hard-coded to 0 and the `alai_min` floor was removed, so LAI of **every perennial** (evergreens too) drops to exactly 0 at dormancy. | One line after the LAI reduction: `lai = max(lai, min(lai_init, alai_min))`. LAI stays at `lai_min` from `plants.plt`, as in rev 61, and is never raised above its pre-dormancy value. Biomass handling unchanged. (Priya, 2026-11-03.) | [#245](https://github.com/swat-model/swatplus/issues/245) (open) |
| 2 | `src/plant_init.f90` | `igrow` has an initialiser, so it is `SAVE`d between calls. For a `perennial` with `days_mat > 0` it is never set, so `phumat` is summed from the planting day of whatever annual crop was initialised before it. Perennial PHU then depends on HRU order (Łupawa: two meadow groups with ~12× different `phumat`). | Reset `igrow = 0` for every plant, and let `perennial` start heat-unit accumulation at the end of dormancy, like `null`/`null1`. (Mikołaj, 2026-11-16.) | not yet reported |

### Build-only change (no effect on results)

| File | Problem | Fix |
|------|---------|-----|
| `src/actions.f90`, `src/hru_control.f90`, `src/mgt_sched.f90` | Their `external ::` lists name `mgt_newtillmix` and `mgt_newtillmix_cswat3`, routines that no longer exist and are never called. gfortran ignores this; Intel ifort on Windows emits a link reference for each name, so linking fails with `LNK2001: unresolved external symbol MGT_NEWTILLMIX` / `MGT_NEWTILLMIX_CSWAT3`. | Remove the two names from the declaration lists. No executable statement changes. For `hru_control.f90` this is the same change upstream made in 62.0.1. |

### Known, not changed

From the `igrow` analysis. Neither is triggered by perennials whose `days_mat` ends before day 365:

1. In the final heat-unit loop of `plant_init`, the northern-hemisphere branch calls `xmon(iday)` without wrapping days above 365. `xmon` then leaves the month unchanged.
2. After the dormancy search loop for `null`/`null1`/`perennial`, `igrow = iday` is used even when the loop ended without finding the end of dormancy.

## Build

[`.github/workflows/build-create-ifort-windows.yml`](.github/workflows/build-create-ifort-windows.yml) runs on every push that changes `src/`, on `v*` tags, and manually (**Actions → Run workflow**, where you can also pick the ifort version).

The artifact `swatplus-62.0.0-create-ifo-win_amd64` contains:

- `swatplus-62.0.0-create-ifo-win_amd64-Rel.exe`
- `BUILD_INFO.txt`: source commit, ifort version, exe SHA-256, list of changes
- `create-changes-vs-62.0.0.diff`: the full source diff against the official 62.0.0
- `smoke_test_Ames_sub1.log`

Pushing a tag `v*` (e.g. `v62.0.0-create.1`) also attaches these files to a GitHub release, which gives partners a stable download link.

The workflow fails if:

- the compiler is not Intel classic ifort;
- the exe needs Intel runtime DLLs (the official exe is statically linked);
- the Ames_sub1 test run does not end with "Execution successfully completed";
- `fort.6` is written.

The upstream workflows (`build.yml`, `doc.yml`, `test.yml`) were removed in this repository because they build with gfortran. They remain in the first commit.

## Checking the fixes in a real project

- **LAI (1):** in `hru_pw_*` / `basin_pw_*` output, perennial LAI no longer drops to 0.000 in winter; it stays at about `lai_min`.
- **igrow (2):** with `fesc` `days_mat` 100–130, `phuplant` in `mgt_out` is about equal for HRU 3804 and HRU 7539 (Łupawa).

## Adding a fix

1. Change the file in `src/` and commit it alone. In the message, describe the problem and the fix, and link the upstream issue if there is one.
2. Add a row to the table above.
3. Push. The workflow rebuilds; tag a release (`v62.0.0-create.N`) when the case studies should switch to it.
