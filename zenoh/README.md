# Local rebuilds of rmw_zenoh_cpp

Two bundles live here. Both exist for the same reason: robostack ships
`rmw_zenoh_cpp` built against one `ros2-distro-mutex` generation, and we need it
against another. Both keep `libzenohc` at the version the stock package uses, so
the rebuild stays wire-compatible with stock installs on other machines.

| bundle | distro | version | mutex | libzenohc | why |
|---|---|---|---|---|---|
| `zenoh-rmw-bundle-jazzy/` | jazzy | 0.2.10 | 0.16 -> **0.14** | 1.9.0 | Eigen 3.4 for OKVIS2-X |
| `zenoh-rmw-bundle/` | humble | 0.1.2 | 0.7 -> **0.9** | 1.4.0 | python 3.12 / numpy 2 |

The humble one is retired -- there are no humble environments left -- and is kept
only as reference for the technique.

## zenoh-rmw-bundle-jazzy (current)

robostack-jazzy's stock `ros-jazzy-rmw-zenoh-cpp 0.2.10` requires
`ros2-distro-mutex 0.16`, and that robostack generation builds its ROS packages
against **Eigen 5**. OKVIS2-X's vendored `opengv` only supports Eigen 3.4 -- its
`FindEigen.cmake` cannot even parse Eigen 5's version macros -- and Eigen 3.4 is
only installable in the **mutex 0.14** generation. So an environment that builds
OKVIS cannot also install stock rmw_zenoh 0.2.10.

`libzenohc`/`libzenohcxx` are plain conda-forge packages, independent of the ROS
generation. Rebuilding 0.2.10 against mutex 0.14 while keeping libzenohc at
**1.9.0** therefore yields an RMW that speaks the same zenoh wire protocol as the
stock 0.2.10 the rest of the fleet runs, but installs next to Eigen 3.4.

Consume it by depending on one of its *outputs* (not on the bundle name):

```toml
[feature.zenoh.dependencies]
ros-jazzy-rmw-zenoh-cpp = { git = "https://github.com/Uspfe/common_ros2_interfaces.git", branch = "main", subdirectory = "zenoh/zenoh-rmw-bundle-jazzy" }
```

and add `preview = ["pixi-build"]` to the consuming workspace. `pixi install`
builds it automatically (cached in `.pixi/bld/`); the vendor output finds conda's
libzenohc via `-DUSE_SYSTEM_ZENOH=ON`, so there is no Rust compile and the build
takes seconds.

### What was changed vs. the official recipes

The recipes are the official ones extracted from `info/recipe/` of
`ros-jazzy-{rmw-zenoh-cpp,zenoh-cpp-vendor}-0.2.10-np2py312h6b96cb9_21.conda`,
with only these differences:

- merged into one multi-output `recipe.yaml` (same reason as the humble bundle,
  below: `pin_subpackage` instead of a nested pixi source dependency)
- `ros2-distro-mutex 0.16.* jazzy_*` -> `0.14.* jazzy_*`, host and run, both outputs
- the rmw output's `ros-jazzy-zenoh-cpp-vendor` requirement replaced with
  `${{ pin_subpackage('ros-jazzy-zenoh-cpp-vendor', exact=True) }}`
- build number 21 -> 1021, so it never collides with an official rebuild

Sources stay on the official release tags (`release/jazzy/*/0.2.10-1`), so this is
the same code, just built for an older robostack generation.

### Gotchas

- `variants.yaml` values must be **lists**; the `variant_config.yaml` extracted
  from a published package uses scalars and is rejected.
- Depend on an output name (`ros-jazzy-rmw-zenoh-cpp`), not the recipe name
  (`ros-jazzy-rmw-zenoh-cpp-bundle`) -- pixi rejects the latter.

## zenoh-rmw-bundle (humble, retired)

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
