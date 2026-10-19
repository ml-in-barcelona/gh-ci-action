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

The root action restores the opam switch. When no cache matches, it runs
`ocaml/setup-ocaml`. When the exact key misses (nothing cached, or only an
older key matched), it installs the dependencies of the `*.opam` files in the
workspace root and saves the switch at once, before the build can fail:

1. `opam pin add <name> <url> -yn` for each line of `pins`.
2. `opam update`, only when an older key was restored (its repository index is
   stale).
3. `opam install . --deps-only`, plus `--with-test` (on by default),
   `--with-doc`, `--with-dev-setup` and `extra-packages` when set.

The key holds the OS, architecture, compiler, `setup-ocaml` version, a hash of
`pins`, `with-test`, `with-doc`, `with-dev-setup`, `extra-packages`, and the hash of
`*.opam`. Nothing else changes it: editing a Makefile does not start a new
switch.

`pins` is an input because opam-repository rejects `pin-depends` in `.opam`
files, so published packages cannot declare them. Each line is `<name> <url>`,
two fields; the action fails and names the line otherwise. Pin to a commit
(`url#sha`) so the key changes with the pin.

On Windows the root action skips its own cache: `tar` cannot recreate the
Cygwin symlinks on restore. There it always runs `setup-ocaml`, with its
built-in cache, and the install. `setup-ocaml` sets `OPAMRETRIES` for
downloads.

`dune-cache/save` has no post step to rely on, so the caller picks when it runs.
Use `if: ${{ !cancelled() }}` to keep the cache after a failed test. It trims
the cache and saves only when the cache grew by more than 1 MB.

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

## Stays in the consumer

- `concurrency`, `fail-fast`, the matrix and the `schedule` trigger.
- Other caches, such as npm.
- If cram tests run dune inside, give them their own cache. Add
  `(env (_ (env-vars (DUNE_CACHE disabled))))` to the root `dune` file so inner
  builds do not share the cache with the outer build.
