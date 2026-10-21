# gh-ci-action

Composite actions for OCaml CI: a cached opam switch and a cached dune build
cache. Works on Linux, macOS and Windows.

## Performance

With a warm cache, the setup step takes under a minute on every OS, 8 to 15
times faster than a cold run. Times of the setup step with OCaml 5.4.0 in
[html_of_jsx run 39250813287](https://github.com/davesnx/html_of_jsx/actions/runs/39250813287)
(2026-10-19), cold (attempt 1) and warm (attempt 2):

| OS | Cold | Warm |
| --- | --- | --- |
| `ubuntu-latest` | 5m 36s | 40s |
| `macos-latest` | 10m 20s | 40s |
| `windows-latest` | 8m 59s | 59s |

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
| `extra-packages` | empty | Space-separated packages, such as tools, installed after the dependencies. The `with-*` inputs do not apply to them. |

Output: `cache-hit` (`true` when this call restored the exact key, else `false`).

A pin URL with a branch or tag, or with no `#ref`, can get new commits. The
cache key includes the commit that ref points to now (`git ls-remote`), so a
new commit rebuilds the switch. On Windows, the install runs up to 3 times.

## dune-cache

`dune-cache/load` restores the dune cache.

`dune-cache/save` saves the cache. Composite actions cannot run a step at job
end, so add this step yourself. Keep the default `if` (success only): after a
failed test, the cache holds its wrong output, and dune replays it in the next
run without running the test again. It trims the cache to `max-size` (default
`2GB`) and saves only when the build added more than 1 MB to the cache,
measured before the trim. It does nothing when `dune-cache/load` did not run.

## Cram tests

If cram tests run dune, turn off the dune cache inside them. Add
`(env (_ (env-vars (DUNE_CACHE disabled))))` to the root `dune` file.
