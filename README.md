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

## How it works

The root action gives you an opam switch with your dependencies installed.

- **Cache hit:** it restores the switch and does nothing else.
- **Cache miss:** it restores an older switch if one exists, or else runs
  `ocaml/setup-ocaml`. Then it installs the dependencies of the `*.opam` files
  in the repository root:
  1. `opam pin add` for each line of `pins`.
  2. `opam update`, only when it restored an older switch.
  3. `opam install . --deps-only`, with the flags and `extra-packages` you set.

  It saves the switch right after the install, so a failed build does not lose
  it.

The cache key changes only when one of these changes: OS, architecture,
compiler, `setup-ocaml` version, any input except `opam-disable-sandboxing`,
or a `*.opam` file. Other edits, such as to a Makefile, reuse the switch.

**`pins`:** opam-repository does not accept `pin-depends`, so give pins here.
Write one `<name> <url>` per line. If a line does not have exactly two fields,
the action fails and names that line. Pin to a commit (`url#sha`), so that a new
commit gives a new key.

**Windows:** the action does not use its own cache, because `tar` cannot
restore the Cygwin symlinks. It always runs `setup-ocaml` (which has its own
cache) and the install.

**`dune-cache/save`:** composite actions cannot run a step at job end, so you
add this step yourself. Use `if: ${{ !cancelled() }}` to save the cache when
tests fail. It trims the cache to `max-size` and saves only when the cache
grew by more than 1 MB.

**Cram tests:** if cram tests run dune, turn off the dune cache inside them.
Add `(env (_ (env-vars (DUNE_CACHE disabled))))` to the root `dune` file.

## Inputs

Root action:

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Compiler version for `setup-ocaml`. Part of the key. |
| `pins` | empty | One `<name> <url>` per line. Part of the key. |
| `with-test` | `true` | Adds `--with-test`. Part of the key. |
| `with-doc` | `false` | Adds `--with-doc`. Part of the key. |
| `with-dev-setup` | `false` | Adds `--with-dev-setup`. Part of the key. |
| `extra-packages` | empty | Space-separated packages installed with the dependencies. Part of the key. |
| `opam-disable-sandboxing` | `true` | Passed to `setup-ocaml`. |

Outputs: `cache-hit`, `cache-matched-key`.

`dune-cache/load`: no inputs. The key holds the OS, architecture, the
`ocamlc -version` of the current switch, and the commit.

Output: `cache-hit`. Sets `DUNE_CACHE=enabled`, `DUNE_CACHE_STORAGE_MODE=copy`
and `DUNE_CACHE_ROOT` (`~/.cache/dune`, or `%LOCALAPPDATA%\dune` on Windows).

`dune-cache/save`:

| Input | Default | Meaning |
| --- | --- | --- |
| `max-size` | `2GB` | Limit for `dune cache trim --size`. |
