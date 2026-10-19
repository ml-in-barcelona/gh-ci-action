# gh-ci-action

Composite actions for OCaml CI: a cached opam switch and a cached dune build
cache. Works on Linux, macOS and Windows.

## Usage

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: ml-in-barcelona/gh-ci-action@v1
    with:
      ocaml-compiler: ${{ matrix.ocaml-compiler }}
      pins: |
        my-lib https://github.com/me/my-lib.git#<sha>
      with-doc: true
      with-dev-setup: true
      extra-packages: dune-release odoc-driver

  - uses: ml-in-barcelona/gh-ci-action/dune-cache/load@v1

  - run: make test

  - uses: ml-in-barcelona/gh-ci-action/dune-cache/save@v1
    if: ${{ !cancelled() }}
```

## Root action

Gives you an opam switch with the dependencies of the `*.opam` files in the
repository root.

- On an exact cache hit, it restores the switch and installs the system
  packages that the opam packages need (`opam install --depext-only`), because
  a fresh VM does not have them. On Linux it first runs `opam update --depexts`
  (`sudo apt-get update`), because opam cannot find system packages with empty
  apt lists. Opam loads the whole switch for this, so it adds about 20 s.
- Otherwise it restores an older switch and runs `opam update`, or runs
  `ocaml/setup-ocaml` when nothing matches. Then it runs `opam pin add` for
  each pin and `opam install . --deps-only` with the flags below. It runs
  `opam clean` and saves the switch at once, so a failed build does not lose it.

An older switch is restored only when these match: OS, architecture, runner
image name (`ImageOS`, or the OS release on self-hosted runners), compiler,
`setup-ocaml` version and `opam-disable-sandboxing`. The exact key also changes
with the runner image version, the `cache-period`, the other inputs below and
the hash of `*.opam`, `*.opam.locked` and `dune-project`. A Makefile edit reuses
the switch. A new image version or a new period restores the old switch and
refreshes it with `opam update`.

On Windows the action does not use its own cache. It runs `setup-ocaml` (with
its own cache) and the install, and `cache-hit` is `false`. Restoring `~/.opam`
with `actions/cache` failed there, but only onto an existing `C:\.opam` tree.
A restore on a fresh runner was not tried.

If the same job calls the action again with the same compiler, it skips
`setup-ocaml`, because the first call already exported its environment.

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Compiler version. |
| `pins` | empty | One `<name> <url>` per line. See below. |
| `with-test` | `true` | Adds `--with-test`. |
| `with-doc` | `false` | Adds `--with-doc`. |
| `with-dev-setup` | `false` | Adds `--with-dev-setup`. |
| `extra-packages` | empty | Space-separated packages to install too. |
| `opam-disable-sandboxing` | `true` | Passed to `setup-ocaml`. It is in the cache key, because the restored opam root keeps the setting. |
| `cache-period` | `%Y-%m` | `date` format in the exact key. `%G-%V` refreshes weekly, empty never. |

`pins` exists because opam-repository does not accept `pin-depends`. If a line
does not have exactly two fields, the action fails and names that line. Pin to
a commit (`url#sha`), so that a new commit gives a new key.

Outputs: `cache-hit` (`true` or `false`), `cache-matched-key` (empty on a full miss).

## dune-cache

`dune-cache/load` restores the dune cache. It has no inputs. The key holds the
OS, architecture, `ocamlc -version` and the commit. It sets
`DUNE_CACHE=enabled`, `DUNE_CACHE_STORAGE_MODE=copy` and `DUNE_CACHE_ROOT`
(`~/.cache/dune`, or `%LOCALAPPDATA%\dune` on Windows). Output: `cache-hit`.

`dune-cache/save` saves the cache. Composite actions cannot run a step at job
end, so add this step yourself. Use `if: ${{ !cancelled() }}` to save the cache
when tests fail. It trims the cache to `max-size` (default `2GB`) and saves
only when the cache grew by more than 1 MB.

## Cram tests

If cram tests run dune, turn off the dune cache inside them. Add
`(env (_ (env-vars (DUNE_CACHE disabled))))` to the root `dune` file.
