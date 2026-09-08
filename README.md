# julia-repo2docker-hub

A [repo2docker](https://repo2docker.readthedocs.io/) environment for running Julia
notebooks on a JupyterHub instance, with a fixed set of packages preloaded and
precompiled at image-build time (rather than on a user's first `using X`).

## What's preloaded

Declared in [`Project.toml`](Project.toml):

- `IJulia` — the Jupyter kernel for Julia
- `Plots` — plotting
- `DataFrames` — tabular data
- `CSV` — CSV I/O
- `DifferentialEquations` — ODE/SDE/DAE solvers

No `Manifest.toml` is committed, so `repo2docker`'s Julia buildpack resolves the
latest compatible versions of each package (and the latest stable Julia release)
at build time via `Pkg.instantiate()`. To pin exact versions for reproducibility,
add a `[compat]` section to `Project.toml`, or generate and commit a
`Manifest.toml` from a local Julia install (`julia --project=. -e 'using Pkg; Pkg.instantiate()'`).

## Precompilation

[`postBuild`](postBuild) runs after the Julia buildpack installs the environment.
It calls `Pkg.instantiate()` / `Pkg.precompile()` and then `using`-loads every
top-level dependency once, so the compiled package cache is baked into the Docker
image layer instead of being paid for by the first notebook session on the hub.

It also sets `JULIA_CPU_TARGET=generic` before precompiling. Without this,
packages get compiled with native code for whichever CPU features the *build*
machine has; if the hub's serving nodes lack one of those features (e.g.
AVX-512), the precompiled cache crashes the container on start with an
"illegal instruction" error — which surfaces on JupyterHub as a restart
back-off loop and a spawn timeout, not a clear error message. `generic` trades
a little runtime performance for working on any node. If your hub's nodes are
known, homogeneous x86_64 hardware, see the comment in `postBuild` for the
official multiversioning target instead.

## Start script

[`start`](start) sets `JUPYTER_RUNTIME_DIR` to the standard per-user location
(`$HOME/.local/share/jupyter/runtime`) before handing off to whatever command
the hub uses to launch the session. repo2docker runs this script as a wrapper
around every container command, so it applies whether the container is started
directly, via `jupyter-repo2docker`, or spawned by JupyterHub.

## Adding or changing packages

Edit `Project.toml`'s `[deps]` section (each entry is `Name = "uuid"`) and rebuild.
`postBuild` will pick up and precompile whatever is listed there — no other file
needs to change.

## Building and testing locally

```bash
pip install jupyter-repo2docker
jupyter-repo2docker .
```

This builds the image and starts a local Jupyter server so you can confirm the
kernel and packages work before deploying to the hub.

## Using on JupyterHub

Point your hub's image-building spawner (e.g.
[BinderHub](https://binderhub.readthedocs.io/) or
[repo2docker-based KubeSpawner integration](https://zero-to-jupyterhub.readthedocs.io/))
at this repository's URL. Each user session will launch with the Julia kernel and
the packages above already available and precompiled — no `Pkg.add` needed.

## Demo

[`Demo.ipynb`](Demo.ipynb) exercises every preloaded package (a `DataFrame`, a
`CSV`-shaped table, an ODE solve with `DifferentialEquations`, and a `Plots` plot)
as a quick smoke test.
