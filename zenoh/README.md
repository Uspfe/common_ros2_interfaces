# Local py312 rebuilds of rmw_zenoh_cpp 0.1.2

robostack-humble only ships `ros-humble-rmw-zenoh-cpp 0.1.2` (and its
`ros-humble-zenoh-cpp-vendor 0.1.2` dependency) for python 3.11 / numpy 1.26 /
ros2-distro-mutex 0.7. Any project on python 3.12, whose robostack
generation is mutex 0.9, will find the stock 0.1.2 binaries unsolvable.

Both packages are [pixi-build](https://pixi.prefix.dev/latest/build/getting_started/)
source packages (`pixi-build-rattler-build` backend). To use them, add them as
path dependencies in a pixi project's `pixi.toml`, e.g.:

```toml
[dependencies]
ros-humble-rmw-zenoh-cpp = { path = "zenoh/ros-humble-rmw-zenoh-cpp" }
```

A plain

```bash
pixi install
```

then builds them from source automatically (cached in `.pixi/build/`). No manual
build step needed.

## What was changed vs. the official recipes

The recipes are the official ones extracted from the published packages
(`info/recipe/` of `*-0.1.2-np126py311hffc189e_13.conda`), with two changes:

- `ros2-distro-mutex 0.7.* → 0.9.*` (host + run) so they coexist with the
  current robostack-humble stack
- `variants.yaml` (per package): python 3.11 → 3.12, numpy 1.26 → 2

`libzenohc`/`libzenohcxx` stay pinned at **1.4.0**, and the source is the same
`release/humble/rmw_zenoh_cpp/0.1.2-1` tag — so zenoh wire compatibility with
stock 0.1.2 installs on other machines is preserved.

## Layout

- `ros-humble-zenoh-cpp-vendor/` — vendors zenoh-cpp; links conda's libzenohc/cxx 1.4.0
- `ros-humble-rmw-zenoh-cpp/` — the RMW itself; its `pixi.toml` declares
  `[package.host-dependencies]`/`[package.run-dependencies]` on the vendor
  source package, which satisfies the recipe's
  `ros-humble-zenoh-cpp-vendor 0.1.2.*` requirement (instead of the py311 binary)
