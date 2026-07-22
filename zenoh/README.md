# Local py312 rebuilds of rmw_zenoh_cpp 0.1.2

robostack-humble only ships `ros-humble-rmw-zenoh-cpp 0.1.2` (and its
`ros-humble-zenoh-cpp-vendor 0.1.2` dependency) for python 3.11 / numpy 1.26 /
ros2-distro-mutex 0.7. Any environment on python 3.12, whose robostack
generation is mutex 0.9, can't solve the stock 0.1.2 binaries.

`zenoh-rmw-bundle/` is a single [pixi-build](https://pixi.prefix.dev/latest/build/getting_started/)
source package (`pixi-build-rattler-build` backend) whose `recipe.yaml` uses a
multi-output recipe to build both packages from one recipe: `ros-humble-zenoh-cpp-vendor`
and `ros-humble-rmw-zenoh-cpp`, the latter depending on the former via
`pin_subpackage`. Point a `pixi.toml` feature at it as a `path` (or `git`)
dependency, e.g.

```toml
[feature.zenoh.dependencies]
ros-humble-rmw-zenoh-cpp-bundle = { path = "zenoh/zenoh-rmw-bundle" }
```

and a plain

```bash
pixi install -e zenoh
```

builds it from source automatically (cached in `.pixi/build/`). No manual
build step needed.

## Why one recipe instead of two

They used to be two separate pixi-build source packages, with
`ros-humble-rmw-zenoh-cpp`'s own `pixi.toml` overriding its
`ros-humble-zenoh-cpp-vendor 0.1.2.*` requirement via
`[package.host-dependencies]`/`[package.run-dependencies]` pointing at the
other package. That works fine with `path` dependencies, but breaks when the
dependency is switched from `path` to `git`: pixi's source-dependency graph
doesn't resolve that nested git-sourced override, silently falls back to
solving `ros-humble-zenoh-cpp-vendor 0.1.2.*` against the binary conda
channels, and picks up the stock py3.11/mutex-0.7 build — which conflicts
with a `python = "3.12.*"` pin.

Merging both into one `recipe.yaml` with an `outputs:` list sidesteps this
entirely: the vendor→rmw dependency is resolved by rattler-build itself via
`pin_subpackage`, not by pixi's cross-source dependency graph, so it works
identically whether the bundle is fetched via `path` or `git`.

## What was changed vs. the official recipes

The recipes are the official ones extracted from the published packages
(`info/recipe/` of `*-0.1.2-np126py311hffc189e_13.conda`), with these changes:

- Merged into one multi-output `recipe.yaml` (see above)
- `ros2-distro-mutex 0.7.* → 0.9.*` (host + run) so they coexist with the
  current robostack-humble stack
- `variants.yaml`: python 3.11 → 3.12, numpy 1.26 → 2
- `ros-humble-zenoh-cpp-vendor 0.1.2.*` (host/run requirement of the rmw
  output) replaced with `${{ pin_subpackage('ros-humble-zenoh-cpp-vendor', exact=True) }}`

`libzenohc`/`libzenohcxx` stay pinned at **1.4.0**, and the source is the same
`release/humble/rmw_zenoh_cpp/0.1.2-1` tag — so zenoh wire compatibility with
stock 0.1.2 installs on other machines is preserved.

## Layout

- `zenoh-rmw-bundle/recipe.yaml` — two outputs:
  - `ros-humble-zenoh-cpp-vendor` — vendors zenoh-cpp; links conda's libzenohc/cxx 1.4.0
  - `ros-humble-rmw-zenoh-cpp` — the RMW itself; depends on the vendor output
    via `pin_subpackage`
- `zenoh-rmw-bundle/build_vendor.sh` / `build_rmw.sh` — per-output build scripts
- `zenoh-rmw-bundle/patch/` — the vendor patch (applied to the vendor output's source only)
