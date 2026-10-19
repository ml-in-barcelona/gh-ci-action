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

- On an exact cache hit, it restores the switch.
- Otherwise it restores an older switch and runs `opam update`, or runs
  `ocaml/setup-ocaml` when nothing matches. Then it runs `opam pin add` for
  each pin and `opam install . --deps-only` with the flags below. It saves the
  switch at once, so a failed build does not lose it.

The cache key changes only with the OS, architecture, compiler, `setup-ocaml`
version, the inputs below (except `opam-disable-sandboxing`) and the `*.opam`
files. A Makefile edit reuses the switch.

On Windows, `tar` cannot restore the Cygwin symlinks, so the action always runs
`setup-ocaml` (with its own cache) and the install.

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Compiler version. |
| `pins` | empty | One `<name> <url>` per line. See below. |
| `with-test` | `true` | Adds `--with-test`. |
| `with-doc` | `false` | Adds `--with-doc`. |
| `with-dev-setup` | `false` | Adds `--with-dev-setup`. |
| `extra-packages` | empty | Space-separated packages to install too. |
| `opam-disable-sandboxing` | `true` | Passed to `setup-ocaml`. |

`pins` exists because opam-repository does not accept `pin-depends`. If a line
does not have exactly two fields, the action fails and names that line. Pin to
a commit (`url#sha`), so that a new commit gives a new key.

Outputs: `cache-hit`, `cache-matched-key`.

## dune-cache/load

No inputs. The key holds the OS, architecture, `ocamlc -version` and the
commit. Sets `DUNE_CACHE=enabled`, `DUNE_CACHE_STORAGE_MODE=copy` and
`DUNE_CACHE_ROOT` (`~/.cache/dune`, or `%LOCALAPPDATA%\dune` on Windows).

Output: `cache-hit`.

## dune-cache/save

Composite actions cannot run a step at job end, so add this step yourself. Use
`if: ${{ !cancelled() }}` to save the cache when tests fail. It trims the cache
to `max-size` (default `2GB`) and saves only when the cache grew by more than
1 MB.

## Cram tests

If cram tests run dune, turn off the dune cache inside them. Add
`(env (_ (env-vars (DUNE_CACHE disabled))))` to the root `dune` file.
