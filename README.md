# gh-ci-action

Composite actions for OCaml CI: a cached opam switch and a cached dune build
cache. Works on Linux, macOS and Windows.

## Performance

With a warm cache, the setup step takes under a minute on every OS, 7 to 12
times faster than `ocaml/setup-ocaml` with its own cache. Times with OCaml 5.4.0
in html_of_jsx:

- `ocaml/setup-ocaml`: the `setup-ocaml`, pin and install steps added together, in
  [run 37486977904](https://github.com/davesnx/html_of_jsx/actions/runs/37486977904)
  (2026-10-06). `setup-ocaml` restored its own cache.
- Cold and warm: the setup step of this action in
  [run 39250813287](https://github.com/davesnx/html_of_jsx/actions/runs/39250813287)
  (2026-10-19), attempts 1 and 2.

| OS | `ocaml/setup-ocaml` | Cold | Warm |
| --- | --- | --- | --- |
| `ubuntu-latest` | 298 s | 336 s | 40 s |
| `macos-latest` | 478 s | 620 s | 40 s |
| `windows-latest` | 534 s | 539 s | 59 s |

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
  opam packages need.
- Otherwise: restores an older switch and runs `opam update`, or runs
  `ocaml/setup-ocaml` when nothing matches. Then it pins, runs
  `opam install . --deps-only`, runs `opam clean` and saves the switch at once.

| Input | Default | Meaning |
| --- | --- | --- |
| `ocaml-compiler` | required | Compiler version. |
| `pins` | empty | One `<name> <url>` per line. |
| `with-test` | `true` | Adds `--with-test`. |
| `with-doc` | `false` | Adds `--with-doc`. |
| `with-dev-setup` | `false` | Adds `--with-dev-setup`. |
| `extra-packages` | empty | Space-separated packages to install too. |

Output: `cache-hit` (`true` when this call restored the exact key, else `false`).

## dune-cache

`dune-cache/load` restores the dune cache.

`dune-cache/save` saves the cache. Composite actions cannot run a step at job
end, so add this step yourself. Use `if: ${{ !cancelled() }}` to save the cache
when tests fail. It trims the cache to `max-size` (default `2GB`) and saves
only when the cache grew by more than 1 MB.

## Cram tests

If cram tests run dune, turn off the dune cache inside them. Add
`(env (_ (env-vars (DUNE_CACHE disabled))))` to the root `dune` file.
