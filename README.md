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

- Exact cache hit: restores the switch and installs the system packages its
  opam packages need (`opam install --depext-only`), because a fresh VM does
  not have them. On Linux it first runs `opam update --depexts`. This adds
  about 20 s.
- Otherwise: restores an older switch and runs `opam update`, or runs
  `ocaml/setup-ocaml` when nothing matches. Then it pins, runs
  `opam install . --deps-only`, runs `opam clean` and saves the switch at once,
  so a failed build does not lose it.

An older switch is restored only when OS, architecture, runner image name,
compiler and `setup-ocaml` version match. The exact key also changes with the
runner image version, the month, the other inputs and the hash of `*.opam`,
`*.opam.locked` and `dune-project`. Opam sandboxing is always disabled.

A second call in the same job with the same compiler skips the restore,
`setup-ocaml` and the save. Its `cache-hit` is `false`.

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Compiler version. |
| `pins` | empty | One `<name> <url>` per line. See below. |
| `with-test` | `true` | Adds `--with-test`. |
| `with-doc` | `false` | Adds `--with-doc`. |
| `with-dev-setup` | `false` | Adds `--with-dev-setup`. |
| `extra-packages` | empty | Space-separated packages to install too. |

`pins` exists because opam-repository does not accept `pin-depends`. Pin to a
commit (`url#sha`), so that a new commit gives a new key. A line without
exactly two fields fails the action.

Output: `cache-hit` (`true` when this call restored the exact key, else `false`).

## Timings

Duration of the `Setup` step in
[run 39245144226](https://github.com/ml-in-barcelona/gh-ci-action/actions/runs/39245144226)
(2026-10-19, commit `5087fce`):

| OS | Cold | Prefix restore | Warm | Second call |
| --- | --- | --- | --- | --- |
| `ubuntu-latest` | 115 s | not measured | 45 s | 20 s |
| `macos-latest` | 146 s | not measured | 30 s | 27 s |
| `windows-latest` | 190 s | not measured | 46 s | 17 s |

Cold means the action's cache has no entry, so the action runs `setup-ocaml`
and installs the dependencies. `setup-ocaml` still found its own opam cache in
this run, so a first run on a new repository can take longer. Prefix restore
means an older switch restores and `opam update` refreshes it. Warm means an
exact hit on a fresh runner. Second call is a later call in the same job.

The fixture in `test/fixture` has only `dune` as a dependency. A real project
takes longer when cold. The `dune-cache` actions are not timed here: the
fixture builds in about 1 s.

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
