# Agent reference

Eleven subagents, shipped as the `astro` plugin. Each entry below gives its scope, what it deliberately
does *not* do, the tools it holds, and what it hands back.

Shared conventions — units, coordinate frames, time systems, provenance, table formats — live in
each project's `CLAUDE.md`, shipped by the companion `astro-template` repository, and are not
repeated in the individual prompts.

**Contents:** [project-scout](#project-scout) · [lit-scout](#lit-scout) · [archive-scout](#archive-scout) ·
[photometry-analyst](#photometry-analyst) · [spectroscopy-analyst](#spectroscopy-analyst) ·
[timeseries-analyst](#timeseries-analyst) · [inference-modeler](#inference-modeler) ·
[theory-sim-analyst](#theory-sim-analyst) · [figure-smith](#figure-smith) ·
[paper-wright](#paper-wright) · [results-referee](#results-referee) ·
[Working with agents](#working-with-agents) · [Adding a new agent](#adding-a-new-agent)

---

## `project-scout`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch

The front end. Turns an interest into a question, then tries to kill it cheaply.

**Does:** rewrites a vague idea as one falsifiable sentence with a threshold; computes the
feasibility arithmetic before anything is downloaded — required S/N, sample size for the desired
precision, cadence and baseline, and the systematics floor below which more data does not help;
names the selection function of every dataset the project would depend on; checks whether the work
has already been done, and done well.

**Does not:** retrieve data or analyse it. It works in estimates and published numbers.

**Hard rule:** every feasibility claim is a number with units and a stated source, separated into
what was verified and what was estimated. "It should be fine" is not an answer.

**Returns:** a scoping note in `docs/`, plus an entry in `docs/NOTEBOOK.md`. It contains the
question, the arithmetic, the data that would be needed and whether it exists, what is already
published, the three most likely ways the project fails with the cheapest test for each, and a
verdict: go, go-with-a-narrower-question, or stop. Its most valuable output is "stop", delivered
early with a number attached.

```
> I want to do something with RR Lyrae in the bulge. Is there a project here?
> Can OGLE photometry alone constrain the metallicity gradient, or do we need spectra?
```

---

## `lit-scout`

**Model:** sonnet · **Tools:** WebSearch, WebFetch, Bash, Read, Write, Edit, Glob, Grep

Literature research. NASA ADS is the primary source (via API when `ADS_DEV_KEY` is set, otherwise
the web interface); arXiv for preprints, which it flags as unrefereed.

**Does:** searches by topic, object, author or method; traces citations backward to find where a
claim originated and forward to find whether it was corrected or superseded; identifies the canonical
reference for a method; surfaces disagreements between groups rather than averaging them; assembles
BibTeX.

**Does not:** analyse data, or summarise a paper from its title alone. When it could only read
abstracts, it says which papers those were.

**Hard rule:** every reference carries a bibcode or DOI it actually retrieved. If it cannot verify a
paper exists, it says so. Fabricated citations would make the agent worse than useless, so this is
stated as a non-negotiable in the prompt.

**Returns:** `docs/lit/<topic>.md` — a synthesis separating settled from contested, an annotated
reference list, explicit disagreements with competing values side by side, and the gaps it searched
for and did not find. `docs/lit/refs.bib` when you are writing. Always dated, with the queries run.

```
> What's the current state of the Hubble tension, and which measurements actually disagree?
> Find the original derivation of the Toomre Q criterion and anything that's revised it since.
```

---

## `archive-scout`

**Model:** sonnet · **Tools:** Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch

Data retrieval from public archives: MAST, ESO, SDSS, Gaia/VizieR, SIMBAD, IRSA, Chandra, ALMA, ZTF,
and OGLE via `scripts/ogle.py` (see [`ogle.md`](ogle.md) — OGLE has no VO service, so it is the one
archive reached through project code rather than astroquery).

**Does:** resolves targets properly (never trusting a typed coordinate, and propagating proper motion
to the observation epoch for fast-moving stars); scopes a query with a metadata-only pass before
downloading anything; prefers TAP/ADQL over scraping; writes the retrieval as a rerunnable script;
records full provenance.

**Does not:** analyse what it downloads. It hands off to a measurement agent.

**Traps it handles by default:** Gaia parallax is not distance and needs a zero-point correction;
`ruwe < 1.4` for astrometric reliability; VizieR's default `ROW_LIMIT` of 50; MAST `calib_level` and
proprietary expiry dates; SIMBAD fluxes being a heterogeneous mix of epochs and photometric systems;
cross-match radii that are meaningless at Galactic-plane source densities.

**Returns:** `scripts/fetch_<dataset>.py`, files under `data/raw/<archive>/<target>/`, and a
`MANIFEST.md` with the query text, service URL, retrieval date, counts, checksums, data rights, and
the acknowledgement string plus DOI the paper will need.

```
> Get all TESS 2-min light curves for TOI-700 and tell me which sectors exist.
> Cross-match this source list against Gaia DR3 and flag anything with high RUWE.
> Pull the OGLE light curves for bulge RR Lyrae within 5 arcmin of this position.
```

---

## `photometry-analyst`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, NotebookEdit, WebFetch

Imaging: CCD/IR reduction, astrometry, source detection, aperture and PSF photometry, crowded fields,
surface photometry of extended sources.

**Does:** reads headers before reducing anything; follows overscan → bias → dark → flat → fringe →
cosmic rays → WCS → coadd, combining calibration frames with sigma-clipped medians; masks saturated
pixels, bleed trails, cosmic rays and chip gaps rather than averaging over them; estimates background
on a 2D mesh; derives an aperture correction from bright isolated stars; fits both zero point and
colour term against a reference catalog.

**Insists on:** an explicit photometric system (AB / Vega / ST) on every magnitude, and a real error
column — read noise, source and background Poisson, flat-field and background-estimation error
propagated. A hand-rolled `sqrt(N)` on a background-subtracted flat-fielded image is wrong, and the
prompt says so.

**Returns:** a rerunnable script or notebook, an ECSV/FITS photometry table with units and errors,
diagnostics (curve of growth, residual image after model subtraction, ZP residuals vs colour, error
vs magnitude), and a stated limiting magnitude and completeness.

```
> Reduce these r-band frames and do aperture photometry on the target and comparison stars.
> This field is crowded — PSF-fit it and calibrate against PanSTARRS.
```

---

## `spectroscopy-analyst`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, NotebookEdit, WebFetch

Spectra: extraction and calibration, telluric and sky subtraction, continuum normalisation, line
fitting, radial velocities, redshifts, IFU cubes.

**Does:** states the wavelength convention (air vs vacuum) and reference frame every time — the
confusion is a ~0.03% error, about 90 km/s, larger than most velocity signals people chase; applies
the barycentric correction and reports its sign; uses optimal (Horne) extraction with variance
weights; validates the wavelength solution against sky emission lines as an independent test;
normalises the continuum with asymmetric sigma clipping and shows the fit overlaid.

**Insists on:** variance arrays carried through every step, and no double resampling — each
interpolation smooths the profile and correlates neighbouring pixels, which invalidates a naive χ².
For IFU cubes it notes that drizzling makes per-spaxel errors underestimates.

**Returns:** calibrated spectra with wavelength, flux, uncertainty and mask; a fit table with errors;
fit-and-residual plots over the fitted region; and what limits the result — S/N per resolution
element, telluric contamination, continuum placement, template mismatch. Non-detections come back as
upper limits with a confidence level, not as a fitted amplitude consistent with zero.

```
> Measure the radial velocity of this star by cross-correlating against a matched template.
> Fit the Hα and [N II] lines and put this on a BPT diagram.
```

---

## `timeseries-analyst`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, NotebookEdit, WebFetch

Time domain: light-curve detrending, period searches, transits and eclipses, rotation, flares, RV
orbits, transients.

**Does:** handles time systems explicitly — BJD_TDB, HJD, MJD, TESS BTJD (BJD−2457000), Kepler BKJD
(BJD−2454833) are not interchangeable, and leap seconds drift UTC-based times enough to wreck a
decade-long ephemeris; masks known transits before detrending, or fits systematics and signal jointly
with a GP, then confirms the depth did not change; respects quality bitmasks, momentum dumps and
scattered-light windows.

**Insists on:** the window function overlaid on every periodogram — most "detections" at 1 d, 29.5 d,
1 yr and their aliases are the sampling, not the star. Significance comes from bootstrap or
permutation, not an analytic FAP that assumes white noise the data does not have. Before a planet is
claimed: odd/even depths, secondary eclipse, centroid shift, depth vs aperture, and dilution from
neighbours in the 21″ TESS pixels.

**Returns:** the detrended curve with its mask, the periodogram with window function and a stated
significance threshold, the phase-folded and binned curve, residuals, and posteriors where a fit was
run. Ephemerides as T0 and P with uncertainties and a reference epoch. Marginal signals come with
what would settle them.

```
> Search these light curves for a rotation period and check it isn't an alias.
> Fit a transit model with limb-darkening priors and give me the planet radius.
```

---

## `inference-modeler`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, NotebookEdit, WebFetch

Statistical inference: likelihoods, priors, MCMC, nested sampling, model comparison, GPs,
hierarchical models, upper limits.

**Does:** writes down the generative model, likelihood and every prior *before* touching a sampler —
most bad astronomical inference is a correct sampler applied to an unstated model; runs a
prior-predictive check; matches the likelihood to the noise (Poisson at low counts, not Gaussian);
puts the selection function in the likelihood when the sample is flux-limited rather than in a
closing caveat; chooses the sampler for the problem — nested for evidence or multimodality, NUTS for
high dimensions with gradients, emcee for low-dimensional and well-behaved.

**Insists on:** convergence reported as numbers, not eyeballed — R̂ < 1.01, effective sample size,
autocorrelation time with chain length > 50τ, divergences for HMC. A posterior predictive check on
every fit: if simulated data does not resemble real data, the uncertainties are meaningless no matter
how clean the chains look. Uniform priors are not uninformative, and the prompt says why.

**Returns:** a model script with likelihood and priors readable in one place, saved chains (HDF5 or
ArviZ NetCDF), printed diagnostics, corner plot with the degeneracies named, posterior-predictive
plot, and a parameter table with credible intervals as median and 16th/84th percentiles — not
presented as mean ± σ when the posterior is asymmetric. A posterior that equals the prior is reported
as a result.

```
> Fit this SED with a two-component model and tell me if the second component is justified.
> Build a hierarchical model for the population and account for the flux limit.
```

---

## `theory-sim-analyst`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, NotebookEdit, WebFetch

Theory and simulation: analytic estimates, N-body and hydro output (yt, GADGET/AREPO/GIZMO/Athena),
halo catalogs, orbit integration (REBOUND, galpy), synthetic observables.

**Does:** starts with the order-of-magnitude estimate — dynamical, cooling, free-fall, relaxation
timescales; virial, Jeans, Eddington, Toomre criteria — because a numerical result that disagrees
with a sound estimate is a bug until proven otherwise, and this catches more bugs than any test
suite; converts code units at read time and states the cosmology and whether lengths are comoving;
picks integrators deliberately (IAS15 for close encounters, WHFast for long-term secular) and checks
energy conservation.

**Insists on:** resolution as a stated physical claim — mass and spatial resolution, softening
length, and which results were convergence-tested. Subgrid prescriptions get named, because
conclusions that depend on them are conclusions about the model, not the universe. To confront data
it forward-models into the observable plane — PSF, pixel scale, noise, band-pass, projection,
selection — rather than comparing an intrinsic 3D quantity to a projected flux-limited measurement.

**Returns:** analysis scripts with units carried through, plots with the analytic expectation
overlaid, convergence status, and an explicit list of assumptions and the regime where the result
holds. Symbolic scaling relations come before the numbers so they can be checked by inspection.

```
> What's the expected timescale for this, order of magnitude, before we simulate anything?
> Analyse this AREPO snapshot and make a mock image matched to our HST observation.
```

---

## `figure-smith`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, NotebookEdit, WebFetch

Publication figures — the ones that carry the argument, not the ones made while thinking.

**Does:** states what the figure is evidence for before writing any code, then builds it from a
script in `scripts/` reading `data/processed/` and writing `paper/figures/`; sizes it to the
journal's text width so it is never rescaled afterwards; shows uncertainties; checks it in greyscale
and for deuteranopia.

**Does not:** exploratory plotting — each analyst does its own. It also does not invent the claim;
if the figure turns out not to support the stated claim, it says so rather than adjusting the
figure until it does.

**Hard rule:** no figure exists that a script cannot regenerate. Hand-tweaked figures go stale
silently when the analysis is rerun.

**Watches for:** truncated axes that inflate a small effect, unlabelled log scales, bin widths the
conclusion depends on, overplotting that hides the sample, best-fit lines drawn without their
uncertainty band, and missing residual panels.

**Returns:** the script, the PDF under `paper/figures/`, a draft caption, and a report of which
failure modes it checked.

```
> Make the period–amplitude figure for the paper from data/processed/rrab_params.ecsv.
> The referee says Fig. 3 oversells the correlation. Rework it.
```

---

## `paper-wright`

**Model:** inherit · **Tools:** Bash, Read, Write, Edit, Glob, Grep, WebFetch

The manuscript, in Acta Astronomica style. Conventions are in
`paper/STYLE.md`; the class file `acta.cls` is the journal's and is never
edited.

**Does:** drafts and restructures sections; keeps every number in the text traceable to a file in
`data/processed/` or `results/`; regenerates the reference list from `docs/lit/refs.bib` via
`scripts/bib2acta.py`; cross-checks citations in both directions, which Acta needs because it has
no `\cite`; builds the PDF; drafts referee responses point by point.

**Does not:** produce results, and does not judge whether a result is sound — that is
`results-referee`.

**Hard rule:** no number is typed by hand. Values, uncertainties, units and significant figures are
checked against their source files before a draft is called done.

**Watches for:** the abstract claiming more than the results section supports, which is the most
common thing a referee catches; a 2.5σ excess written as a detection; upper limits quoted without a
confidence level; "consistent with" drifting into "confirms"; an assumption the conclusion depends
on that appears only in the methods.

**Returns:** the edited `ms.tex`, a built `ms.pdf`, and a report of which numbers it verified
against which files, which citations failed the cross-check, and every place it judged the text was
overclaiming.

```
> Draft the observations section from the manifests in data/raw/ogle/.
> Check every number in ms.tex against data/processed/, then build it.
> The referee report is in docs/referee_1.md. Draft a response.
```

---

## `results-referee`

**Model:** inherit · **Tools:** Read, Grep, Glob, Bash, WebFetch — **read-only, never edits**

An adversarial check on a finished result, run before it is trusted, written up, or submitted.

**Does:** works through units and dimensions (flux vs flux density, AB/Vega/ST, h-dependence, which
L☉); coordinate and time systems (ICRS vs FK5, epoch and proper motion, air vs vacuum, BJD_TDB vs
HJD, barycentric sign); error propagation through nonlinear steps and correlated errors; statistical
claims (realistic noise model, trials factor, marginal detections described as detections, model
selected on its own validation data); selection effects; plot honesty; reproducibility; and whether
the conclusion actually follows or only a weaker version does.

**Does not:** fix anything. It reports precisely enough that someone else can.

**Returns:** findings grouped as **blocking** (the result is wrong or unsupported), **major** (may
hold, but the evidence as presented does not establish it), and **minor** (presentation,
reproducibility). Each with file, line, what is wrong, and the concrete scenario where it gives a
wrong answer. It distinguishes "this is wrong" from "I could not verify this", and names the checks
it could not perform.

A clean report is only meaningful if it names what was actually verified — the prompt requires that.

```
> Referee scripts/fit_orbit.py and the results in results/ before I write this up.
> Check the error budget in this analysis — I think the systematics are underestimated.
```

---

## Working with agents

**Routing.** Describe the task; the right agent is usually selected from its `description`. Name one
explicitly (`use results-referee to…`) when you want a particular perspective — most usefully to have
one agent check another's work.

**Parallelism.** Independent work runs concurrently. Literature and data retrieval at the start of a
project are the obvious pair; so are photometry and spectroscopy on the same target.

**Context.** Each agent starts cold and returns a summary, not its full transcript. Give it enough to
work from — the target, the file paths, what you already know. Continue an existing agent rather than
spawning a fresh one when the context matters.

**Sequence.** The natural flow, though nothing enforces it:

```
project-scout  →  lit-scout  →  archive-scout  →  {photometry,spectroscopy,timeseries}-analyst
      ↑                                                        ↓
      └── stop / narrow ──                      inference-modeler  ←→  theory-sim-analyst
                                                               ↓
                                                        results-referee
                                                               ↓
                                              figure-smith  →  paper-wright
                                                               ↓
                                                  results-referee (on the manuscript)
```

`results-referee` is invoked twice on purpose: once on the analysis, once on the written paper. A
sound result can still be overstated in prose, and the second pass is cheap.

## Adding a new agent

One markdown file per agent in `plugins/astro/agents/` (or in a project's own `.claude/agents/`
when it is specific to that project):

```markdown
---
name: kebab-case-name
description: When to invoke this agent. Concrete triggers, and what it is NOT for.
tools: Bash, Read, Write, Edit, Glob, Grep      # omit to inherit all
model: inherit                                   # or sonnet, opus, haiku
---

System prompt.
```

What makes these prompts work:

- **The `description` is what routing matches on.** Concrete triggers ("the x-axis is time or phase")
  beat topic labels, and an explicit "NOT for…" prevents the agent being pulled into neighbouring
  work.
- **Write the subfield's real failure modes,** not generic best practice. "Overlay the window
  function" is worth more than "be careful with periodograms".
- **Say what the deliverable is,** including the diagnostics, so results arrive checkable.
- **Restrict tools when the role calls for it.** `results-referee` is read-only because a referee
  that edits is no longer an independent check.
- **Put shared conventions in `CLAUDE.md`,** not in every prompt.

Files are picked up on the next session, or on `/reload-plugins`. Precedence runs project
`.claude/agents/` → user `~/.claude/agents/` → plugins, so a project copy silently overrides the
plugin's version of the same `name`.
