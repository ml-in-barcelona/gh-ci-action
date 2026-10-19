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
      install: make pin install

  - uses: ml-in-barcelona/gh-ci-action/dune-cache@v1
    with:
      ocaml-compiler: ${{ matrix.ocaml-compiler }}

  - run: make test

  - uses: ml-in-barcelona/gh-ci-action/dune-cache/save@v1
    if: ${{ !cancelled() }}
```

The root action restores the opam switch. When no cache matches, it runs
`ocaml/setup-ocaml`. When the exact key misses (nothing cached, or only an
older key matched), it runs `install` and saves the switch at once, before the
build can fail.

`dune-cache/save` has no post step to rely on, so the caller picks when it runs.
Use `if: ${{ !cancelled() }}` to keep the cache after a failed test. It trims
the cache and saves only when the cache grew by more than 1 MB.

## Inputs

Root action:

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Compiler version for `setup-ocaml`. Part of the key. |
| `install` | `make install` | Command that installs dependencies. Runs on an exact-key miss. Retried 3 times on Windows. |
| `cache-key-files` | `*.opam`, `dune-project`, `Makefile` (one per line) | `hashFiles` patterns, one per line. |
| `opam-disable-sandboxing` | `true` | Passed to `setup-ocaml`. |

Outputs: `cache-hit`, `cache-matched-key`.

`dune-cache`:

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Part of the key. |

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
