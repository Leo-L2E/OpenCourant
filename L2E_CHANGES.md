# Leo-L2E/OpenCourant: changes on top of OpenCourant

This fork follows [OpenCourant/OpenCourant](https://github.com/OpenCourant/OpenCourant) `main` and adds fixes found while building and validating OpenCourant on Windows. It serves as the baseline ("oracle") for the XFE.rs project. Everything stays under AGPL-3.0, and the QA decks keep their original licence (CC BY-NC 4.0).

| | |
|---|---|
| Upstream base | OpenCourant `main` at `b19ec63f3` (2026-10-15) |
| Fork `main` | upstream base + the changes below |
| Maintainer | Leo Laine (LL Engineering AB) |

---

## 1. Changes in this fork

### 1.1 QA include decks: licence headers turned into Radioss comments (2026-10-13)
**Commit:** `bbfed4e64` (branch `fix/qa-inc-copyright-comments`), merged in `83c17bbf4`.

**Problem:**
- Upstream commit `3fb66e10d` (2026-07-16) stamped a 20-line licence header on 73 Radioss *input* include files under `qa-tests/` (72 `qadiags.inc`, 1 `materials.inc`).
- The header uses the Fortran-style prefix `Copyright>` with no comment character.
- In a Radioss deck only lines starting with `#` or `$` are comments. When `#include qadiags.inc` follows a block with a variable number of lines (`/SHELL`, `/QUAD`), Starter reads the licence text as element data and stops:

```
ERROR ID : 100101   ** ERROR IN INPUT FORMAT
-- BLOCK: /SHELL/3
-- LINE: Copyright>        OpenRadioss
STRING "Copyright>" found instead of an integer
```

Affected MiniQA tests on win64:
- 1.31 (`LOIS/LOI66/test_shell`)
- 1.33 (`LOIS/LOI76/shell/test1/compression`)
- 1.36 (`OPTIONS/desacti/test2/version_5`)

**Fix:**
- Every header line in those 73 files is prefixed with `#` (`#Copyright>`). The header text itself is unchanged.
- OpenCourant made the same change for one file in `6b1eb42a9` (`SPH/WavesCompression`).
- `scripts/copyright/check_headers.py --changed` accepts all 73 files as the *legacy* variant. This also holds with the `openradioss-final` fork point (`1a0d691b8`).

**Result:** MiniQA went from 74/78 to **77/78**. The three tests pass.

---

## 2. Issues found but not changed here

| # | Issue | Status |
|---|---|---|
| A | **MiniQA 1.59** (`FLUIDE_NEW/EBCS/INIV`) fails only on `MASERR`: reference −0.9946E-12, run −1.85E-16. The tolerance is *relative*, so two values that are both zero to round-off are compared: 0.9998 > 0.00066. | Test-definition issue, not a solver error. Candidate upstream change: an absolute floor for near-zero references. |
| B | **SMP and MPI engines give different solutions** on a 344 k-element ALE ground-shock model with `/INTER/TYPE2`, `TYPE18` (FSI), `TYPE24`, `/EBCS/NRF` and `/BCS/NRF`. With 6 domains, the added mass at cycle 0 is 3 506 against 3 642 for 1 domain. After about 80 cycles the time step becomes nodal-controlled and 4–10× smaller. The Starter listings differ only in the decomposition. | Under investigation. Until it is explained, run such models with SMP and 1 domain. |
| C | `build_windows.bat -nt=all` maps to `ninja -j 0`, i.e. *unlimited* parallel jobs. The help text says it "consumes all resources". | Not changed. On a 20-thread laptop, `-nt 18` builds Starter + Engine in about 14 min, against 95 min with the default `-nt 1`. |

---

## 3. Verified configuration (2026-10-13)

The fixes were verified on OpenCourant `fca6eaa4a` + `bbfed4e64`. The upstream commits up to `b19ec63f3` only change the Linux Intel CMake flag files, so they do not affect this Windows result.

| Item | Result |
|---|---|
| Machine | Intel Core i9-13900H (20 threads), 64 GB, Windows 11 |
| Toolchain | Intel oneAPI 2025.1.3 (ifx/icx 2025.1.1, Intel MPI 2021.15), Visual Studio 2022 17.7.6 (MSVC 14.37), CMake 3.28, Ninja 1.11, extlib v82 |
| Build (`-arch win64 -release -nt 18`) | `starter_win64.exe` in 3 min 50 s, `engine_win64.exe` (SMP) in 10 min 19 s, `engine_win64_impi.exe` in 11 min 19 s. 0 errors |
| Smoke test ACCELERO | Final cycle identical to `ref.extract` → PASS |
| MiniQA (`qa-tests\test_script.bat`, CTest, SMP 1×4) | 74/78 before, **77/78** after the fix (1.59 = issue A) |
| 344 k-element ALE ground-shock model, 90 ms | Normal termination, SMP 1×20 threads, 1 h 20 min |

### Windows notes
- If the environment variable `NoDefaultCurrentDirectoryInExePath` is set (some automation shells do this), `setvars.bat` fails with *"'vars.bat' is not recognized"*. Clear it in the build script before `call setvars.bat`.
- A copy of `qa-tests` taken from OneDrive carries the ReadOnly attribute on every folder. `or_execute.py` then cannot delete its run folders, and every test is reported as failed. Run `attrib -r <qa-tests>\* /s /d` first.
