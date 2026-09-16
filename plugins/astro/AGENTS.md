# Research agents

Eleven agents covering the path from "I have a vague interest" to "the paper is submitted".
Invoke by name, or describe the task and the right one will be selected. Installed as a plugin
they are also addressable scoped, as `astro:lit-scout` and so on.

For installing, editing and versioning these, see the [repository README](../../README.md).
For what each one does in detail, [`docs/agents.md`](../../docs/agents.md).

| Agent | Use it when | Model |
|---|---|---|
| `project-scout` | Before anything else — is this a real question, and can the data answer it? | inherit |
| `lit-scout` | "What does the literature say?" — ADS/arXiv search, citation tracing, BibTeX | sonnet |
| `archive-scout` | "Get me the data" — MAST, ESO, Gaia/VizieR, SIMBAD, IRSA; cross-matching, downloads | sonnet |
| `photometry-analyst` | The input is images or the output is magnitudes/fluxes | inherit |
| `spectroscopy-analyst` | The input is a spectrum, echelle order set, or IFU cube | inherit |
| `timeseries-analyst` | The x-axis is time or phase — light curves, periods, transits, RV orbits | inherit |
| `inference-modeler` | "Fit this model and give me real uncertainties" — MCMC, nested sampling, model comparison | inherit |
| `theory-sim-analyst` | Derivations, order-of-magnitude checks, N-body/hydro simulation analysis, orbits | inherit |
| `figure-smith` | A result is ready to be *shown* rather than computed | inherit |
| `paper-wright` | The results are final and the task is prose, or a referee has replied | inherit |
| `results-referee` | Before anything is trusted, written up, or sent. Read-only, adversarial | inherit |

## Typical flow

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

`project-scout` runs first and its most useful verdict is "stop" — a project killed on arithmetic in
an afternoon costs nothing. `results-referee` appears twice: once on the analysis, once on the
written paper, since a sound result can still be overstated in prose. It is read-only by design and
never edits files — treat a clean report as meaningful only if it names what it actually verified.

Agents can run in parallel when their work is independent — e.g. `lit-scout` on the background
literature while `archive-scout` pulls the data.
