# allesfitter on modern Python: what was changed and what was tested

Date: 2026-10-21
Repo: https://github.com/moosesteve21/allesfitter, a fork of https://github.com/MNGuenther/allesfitter (upstream last commit 2023-04-14, v1.2.10)
Branch: `modernise-py312`, the fork's **default branch**. The fork's `master` is left as an unmodified mirror of upstream.
Local clone: `/home/sprice/allesfitter` (`origin` = fork, `upstream` = MNGuenther/allesfitter)

## Summary

allesfitter 1.2.10 now runs on **Python 3.12 with numpy 2.5, scipy 1.18, matplotlib 3.11, pandas 3.0, astropy 8, emcee 3.1.6 and dynesty 3.1**. Getting there took small, behaviour-preserving source fixes (24 lines of API changes plus 52 escaped-backslash string literals) and a manual build of `ellc`. An end-to-end MCMC and nested-sampling fit of the bundled transit tutorial completes and recovers the injected parameters. Many features are **not yet tested**; see the list near the end.

## Environment

| Item | Value |
|---|---|
| Python | 3.12.15 (uv-managed), venv at `/home/sprice/allesfitter/.venv` |
| allesfitter | installed editable (`uv pip install -e .`), so the venv uses the repo source directly |
| Dependency ranges | `requirements.txt` |
| Exact tested versions | `requirements-lock.txt` |
| ellc | 1.8.15, git commit `a0afdce` of github.com/pmaxted/ellc, built manually (see below) |

**Why Python 3.12 and not newer:** `celerite` 0.4.3 publishes wheels only up to cp312, and `george` 0.4.4 only up to cp313. On newer Python they would have to be compiled from C++ source. Every other dependency has wheels for 3.12. All PyPI packages were installed with `--only-binary :all:`, so no third-party build scripts ran.

## Source changes to allesfitter

All of these preserve behaviour; they replace APIs that were removed upstream.

| Change | Why | Files |
|---|---|---|
| `np.float(x)` → `float(x)`, `np.int(x)` → `int(x)` | Aliases removed in numpy 1.24. Crashed on every fit (`basement.py`). | `basement.py`, `detection/transit_search.py`, `detection/injection_recovery_output.py`, `v2/detection/injection_recovery_output.py`, `exoworlds_rdx/lightcurves/lightcurve_tools.py` |
| `np.VisibleDeprecationWarning` / `np.RankWarning` → `np.exceptions.*` | Removed from the main numpy namespace in 2.0. Broke `import allesfitter`. | `basement.py`, `computer.py`, `general_output.py`, `mcmc.py`, `nested_sampling.py` |
| `tick.label.set_fontsize` → `tick.label1.set_fontsize` | `Tick.label` removed in matplotlib 3.8. Crashed corner plots. | `mcmc_output.py`, `nested_sampling_output.py`, `deriver.py` |
| `from matplotlib.cm import get_cmap` → `from matplotlib.pyplot import get_cmap` | `matplotlib.cm.get_cmap` removed in 3.9 | `exoworlds_rdx/lightcurves/gp_decor.py` |
| Invalid escapes in LaTeX label strings (`'\mathrm'`) → `'\\mathrm'` | SyntaxWarning on 3.12; will become an error in a future Python. The edit was made with the tokenizer and verified to leave the parsed AST (and therefore every string value) **identical**. | `deriver.py` (46), `priors/estimate_noise_wrap.py` (4), `plotting.py` (1), `prepare_ttv_fit.py` (1) |

Added: `requirements.txt`, `requirements-lock.txt`, this file.

## Building ellc (required: every fit imports it)

The PyPI sdist of ellc 1.8.15 is broken. Its `setup.py` runs `make`, but the makefile (named lowercase `makefile`) is not in `MANIFEST.in`, so `pip install ellc` fails. ellc loads its Fortran library through `ctypes`, not numpy's C API, so it does not depend on the numpy version.

Requires `gfortran` (`make` is not needed). To rebuild:

```bash
git clone https://github.com/pmaxted/ellc.git && cd ellc   # tested at a0afdce
S=f_src
nice -n 19 gfortran -O2 -fPIC -shared $S/constants.f90 $S/utils.f90 $S/coords.f90 \
  $S/gauss_legendre.f90 $S/solve_real_poly.f90 $S/ellipse.f90 $S/stellar.f90 \
  $S/spots.f90 $S/ellc.f90 -o libellc.so
cp libellc.so ellc/
uv pip install --python /path/to/.venv .     # the .so is picked up via MANIFEST.in
```

This is exactly what the upstream makefile does. Because ellc defines no extension modules, `setup.py` will not try to run `make` during install once the `.so` is in place.

**Verified:** `ellc.lc` gives a transit depth of exactly 1.0% for radius ratio 0.1, as expected. `ellc.rv` gives K₁ = 0.16847 km/s against an analytic 0.1685 km/s for a = 10 R☉, P = 3 d, q = 0.001.

## What was tested

Test case: `tutorials/01_crash_course` (single planet, one photometric instrument, quadratic limb darkening, sampled offset baseline and error). It was run on a copy with reduced settings to keep CPU use low: `multiprocess,False`, MCMC with 40 walkers × 500 steps (burn 100, thin 5), and static nested sampling with 100 live points and `ns_tol 1`.

| Stage | Result |
|---|---|
| `import allesfitter` | OK |
| `allesfitter.mcmc_fit` (emcee 3.1.6, HDF5 backend) | OK |
| `allesfitter.mcmc_output` (tables, LaTeX, chains, corner, fit plots) | OK |
| `allesfitter.ns_fit` (dynesty 3.1.0, static, `single`/`rwalk`) | OK |
| `allesfitter.ns_output` (tables, derived parameters, corner, trace, fit plots) | OK |

Recovered parameters (truth in brackets):

| Parameter | MCMC | Nested sampling |
|---|---|---|
| R_b/R★ (0.1) | 0.103 | 0.106 |
| P (3.4 d) | 3.4016 d | 3.4011 d |
| T₀ (1.1 + 1 period = 4.5, epoch shifted by `shift_epoch`) | 4.5021 | 4.5021 |

These runs were deliberately short. They show that the code paths work, **not** that the posteriors have converged.

After all fixes, the only warnings left at runtime are 4 FutureWarnings, all covered by the version pins:
- **scipy:** `scipy.stats.anderson` (`statistics.py:64`) relies on result attributes that **scipy 1.19 removes**. Pinned `scipy<1.19`.
- **statsmodels:** `adfuller` (`statistics.py:114`) changes its return type in **statsmodels 0.16**. Pinned `statsmodels<0.16`.

Patching those two calls would allow the pins to be lifted.

## Not tested (treat as unverified)

- Multiprocessing mode (`multiprocess,True`). It uses `multiprocessing.Pool` for both emcee and dynesty.
- `ns_modus,dynamic` (DynamicNestedSampler) and other bound/sample options. allesfitter was written against dynesty 1.x/2.x, and dynesty 3 is the most likely source of subtle behaviour changes.
- GP baselines and stellar variability (celerite, george). The packages import and a celerite likelihood evaluates, but no allesfitter GP fit has been run.
- RV fits, joint photometry+RV, eccentric orbits, SB2 binaries, flares, TTVs, spots, phase curves.
- `exoworlds_rdx`, `detection/` and `v2/` modules. These need `exoworlds` and `tdpy`, which are the author's own packages and not on PyPI.
- The Jupyter GUI (`allesfitter.GUI()`, which calls `jupyter notebook` via `os.system`).
- Loading results files written by older allesfitter/dynesty versions (`save_ns.pickle.gz` pickles are tied to the dynesty version).

Before relying on any of these, run the matching tutorial in `tutorials/` and compare against the results checked into that tutorial's `results/` folder.

## Notes for use in another project

- **Python version:** use Python ≤ 3.12 unless celerite/george can be compiled.
- **Installing:** in a Python 3.12 venv, install from the fork. Its default branch has the fixes, so no branch name is needed:
  ```bash
  uv pip install "git+https://github.com/moosesteve21/allesfitter.git"
  uv pip install -r https://raw.githubusercontent.com/moosesteve21/allesfitter/modernise-py312/requirements.txt
  ```
  Then build ellc as above. `setup.py` declares no dependencies, so the requirements file has to be installed explicitly. To pin an exact version, append `@<commit>` to the git URL. For local development, use `uv pip install -e /home/sprice/allesfitter` instead, or point at the existing venv `/home/sprice/allesfitter/.venv`.
- **Pulling upstream changes** (unlikely; upstream is inactive): `git fetch upstream`, then merge `upstream/master` into `modernise-py312`.
- **CPU usage:** in `settings.csv`, `multiprocess_cores,all` means `cpu_count()-1` (15 on this 16-thread machine), and every tutorial ships with `multiprocess,True` + `all`. On a shared machine, set an explicit integer. Also set `OMP_NUM_THREADS=1` (and `OPENBLAS_NUM_THREADS=1`) so BLAS does not add threads inside each worker.
- **Headless runs:** set `MPLBACKEND=Agg`.
- **Global plotting side effect:** importing allesfitter calls `seaborn.set(...)`, which changes matplotlib defaults for the whole process.
- **Warnings filter:** allesfitter overrides `warnings.formatwarning` globally on import.
