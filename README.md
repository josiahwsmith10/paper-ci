# paper-ci

Shared build pipeline for the paper repos. One reusable GitHub Actions workflow plus the
two scripts it calls.

## Using it

In a paper repo, `.github/workflows/build.yml`:

```yaml
name: Build
on: [push, pull_request]

jobs:
  paper:
    permissions:
      contents: write   # required: lets tag builds create the Release
    uses: josiahwsmith10/paper-ci/.github/workflows/build-paper.yml@v1
    with:
      name: spectral-sr-swin
      main: main.tex
      arxiv-main: main_arxiv.tex
```

**`permissions: contents: write` must be on the caller.** A called workflow can only
narrow the caller's permissions, never widen them, so this cannot be set here on your
behalf — declaring it in `build-paper.yml` makes every caller with default (read-only)
token permissions fail to start. Omitting it is easy to miss, because branch builds pass
either way; only the tag build fails, at the very last step, after a full LaTeX run.

Every push produces a `*-build` artifact containing:

| File | What it is |
|---|---|
| `<name>.pdf` | journal/conference build |
| `<name>-arxiv.pdf` | arXiv build |
| `<name>-arxiv.tar.gz` | upload this straight to arXiv |
| `<name>-refs.bib` | only the entries this paper cites |

Pushing a `v*` tag additionally attaches all of them to a GitHub Release.

## Versioning

Paper repos pin `@v1`. Move the `v1` tag for backward-compatible fixes; cut `v2` for
anything that changes required inputs.

```sh
git tag -f v1 && git push -f origin v1
```

## Scripts

### `make_arxiv.py`

Builds the submission tarball from the `.fls` file that `latexmk -recorder` writes. The
`.fls` lists every file TeX actually opened, which means unused figures and stale drafts
are excluded because they were never read — no manual file list to maintain and no risk
of shipping an old draft.

It also:

- keeps `<main>.bbl` and drops the `.bib` (arXiv does not run BibTeX)
- excludes TeX Live system files, since arXiv supplies its own
- strips comment bodies from `.tex`/`.sty`/`.cls` while keeping the `%` character, so
  draft notes do not become public but line-ending spacing is unchanged
- fails if the tarball exceeds arXiv's 50 MB limit

### `prune_bib.py`

Reads `\citation{...}` from the `.aux` and emits a `.bib` with only those entries,
following `crossref` chains. Warns on cited keys missing from the bibliography.

## The TeX Live container

Builds run in `texlive/texlive:latest` (full scheme). The papers use packages well
outside a minimal install — `algorithm2e`, `mdwtab`, `boondox`, `overpic`, `subfig` —
so a slimmer image would need per-paper package lists. Pin a dated tag
(`texlive/texlive:TL2025-...`) via the `texlive-image` input if a paper needs to be
frozen against upstream changes.
