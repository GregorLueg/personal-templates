# personal-templates

Copier template for my R packages. It owns the files that have to exist inside
each repo and cannot be called remotely: the rextendr build scaffolding, the
thin CI callers, and a couple of dotfiles.

The CI logic itself lives in
[personal-actions](https://github.com/GregorLueg/personal-actions). This
template only ships the callers that point at it.

## Why this exists rather than `rextendr::use_extendr()`

`use_extendr()` is a create-time generator. It writes the scaffolding once and
has no merge path, so it cannot propagate a fix.

More to the point, what is in these repos is not stock rextendr output. Two
patches were applied by hand and then drifted:

- `tools/config.R` forces `is_debug <- FALSE`, so tests always run under release
  and the Rust speed is visible from R during development. Applied everywhere.
- `Makevars.in` passes `@PROFILE@` to the `cargo run --bin document` line so it
  matches. Applied in two of four, and only one of those on Windows.

So `use_extendr()` still bootstraps a brand-new package, and this template owns
the files from that moment on. When rextendr bumps its own templates, diff them
against this one and decide in a single place rather than four.

## Usage

Adopting a repo that already exists:

```sh
cd ~/repos/shared/manifoldsR
git switch -c chore/copier
copier copy --trust --overwrite gh:GregorLueg/personal-templates .
git diff                # this is the drift, paid down once
```

Afterwards, `copier update` is a three-way merge against the recorded template
version in `.copier-answers.yml`.

## Answers per package

| package | `has_rust` | `ci_rust` | `windows` | `linux_runner` | other |
|---|---|---|---|---|---|
| `manifoldsR` | true | true | true | ubuntu-latest | |
| `genewalkR` | true | true | true | ubuntu-latest | `rust_toolchain: 1.91.0` |
| `bixverse` | true | true | false | ubuntu-22.04 | `rebuild_from_source: stringfish qs2` |
| `bixverse.plots` | false | true | false | ubuntu-22.04 | `quarto: false` |
| `bixverse.gpu` | true | true | false | ubuntu-22.04 | `has_gpu: true` |

`has_rust` and `ci_rust` differ on purpose. `has_rust` controls the scaffolding,
`ci_rust` controls whether CI installs a toolchain. `bixverse.plots` has no
`src/` of its own but builds `bixverse` from `Remotes`, so it needs one.

## Template mechanics

Two things are easy to get wrong here.

**Delimiters.** GitHub Actions uses `${{ ... }}`, which collides with Jinja's
default `{{ ... }}`, so `_envops` shifts them to `[[ ]]` and `[% %]`. That in
turn would collide with R's own `[[ ]]`, which appears in `tools/config.R` as
`.Platform[["OS.type"]]`.

**The `.jinja` suffix.** Copier only renders files ending in `.jinja` and copies
everything else byte for byte. That is what saves the R files: they ship without
the suffix, so the delimiter collision never arises. Only add `.jinja` to a file
that genuinely needs a variable substituted.

Conditional files use a Jinja expression as the filename, e.g.
`[% if has_rust %]configure[% endif %]`. A name that renders empty is skipped.
Directories need the same treatment or you get an empty directory.

## What is not templated

`DESCRIPTION`, `NAMESPACE`, `_pkgdown.yml`, `Cargo.toml`, `Cargo.lock`, `R/`,
`man/`, `src/rust/src/`, `vignettes/`, `NEWS.md`, `.Rbuildignore`. These diverge
for real reasons, and templating `DESCRIPTION` would mean a conflict on every
update in the one file nobody wants a conflict in.

Version pins stay out too. They move on a release cadence that has nothing to do
with template versions, which is Renovate's job.
