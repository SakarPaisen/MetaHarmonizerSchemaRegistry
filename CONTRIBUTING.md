# Contributing to MetaHarmonizerSchemaRegistry

This repo holds the schema artifacts (LinkML schemas, SchemaMapper target
schemas, value dicts, alias dicts) and the builders that produce them. What the
pipeline *does* is documented in [`scripts/README.md`](scripts/README.md) —
read that first. This document is about making the builders **runnable by
someone who didn't write them**: toolchains, dependencies, and which steps are
offline.

## Two toolchains, one behavior

`scripts/build_schema_artifacts.R` and `scripts/build_schema_artifacts.py` are
equivalent implementations kept in sync **by hand**; likewise
`schema_validation.R` / `schema_validation.py`. Either language is a valid way
to contribute. If you change one, change the other — or state in the PR why the
change is toolchain-specific. There is no CI here, so "do both produce the same
thing?" is on you; see [Checking parity](#checking-parity).

## Dependencies

This repo is not a package: the builders are scripts with imports. The Python
side is declared in [`requirements.txt`](requirements.txt); for R, install the
snippet below. Three dependencies are **not on PyPI/CRAN at all**:

| dependency | used by | where it comes from |
|---|---|---|
| `metaharmonizer.knowledge_db.combined_corpus_builder` | `build_ontology_artifacts.py` | GitHub `shbrief/MetaHarmonizer` |
| `OmicsMLRepoCuration::tableToLinkmlSchema` | `build_schema_artifacts.R`, `schema_validation.R` | GitHub `shbrief/OmicsMLRepoCuration` |
| `MetaHarmonizerR::generate_alias_dict` | `build_schema_artifacts.R`, artifact 4 only | GitHub `shbrief/MetaHarmonizerR` |

### Python

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt        # pandas, PyYAML — artifacts 1-3 + validation

# GitHub-only; needed for artifact 4 (alias dict) and for
# build_ontology_artifacts.py (combined_corpus_builder):
pip install "git+https://github.com/shbrief/MetaHarmonizer"
```

### R

```r
install.packages("jsonlite")
install.packages("remotes")
remotes::install_github("shbrief/OmicsMLRepoCuration")   # tableToLinkmlSchema
# artifact 4 (alias dict) only:
remotes::install_github("shbrief/MetaHarmonizerR")        # generate_alias_dict
```

No `renv.lock` on purpose: two of the three R dependencies are moving GitHub
targets and the scripts need nothing else, so a lock file would freeze a whole
R universe to pin two packages. If that trade-off stops being right, adopting
`renv` later is cheap.

## What needs what

| step | offline & deterministic | network | LLM key |
|---|---|---|---|
| 1 · `<prefix>.linkml.yaml` | ✅ | — | — |
| 2 · `<prefix>_target_attrs.csv` | ✅ | — | — |
| 3 · `<prefix>_target_attrs_allowed_values.json` | ✅ | — | — |
| 4 · `<prefix>_target_attrs_alias_haiku.csv` (+ `.meta.json`) | — | ✅ | `ANTHROPIC_API_KEY` |
| ontology artifacts (`build_ontology_artifacts.py`) | — | ✅ (NCI EVSREST + EBI OLS4; both public, keyless) | — |
| `schema_validation.*` | ✅ | — | — |

Without a key (or with `--no-alias`) artifacts 1–3 are still produced; the R
builder additionally skips artifact 4 when `MetaHarmonizerR` is not installed.

## Running the builders

From a clean clone, both toolchains must produce artifacts 1–3 from
`schema/cbioportal/cbioportal_schema.csv`:

```bash
python scripts/build_schema_artifacts.py --no-alias
Rscript  scripts/build_schema_artifacts.R --no-alias
```

Flags are identical in both builders — `--input`, `--outdir`, `--prefix`,
`--model`, `--schema-domain`, `--no-alias` — see `scripts/README.md` for
semantics.

## The `<prefix>_schema.csv` format

Every builder reads a schema table written by R's `write.table()` defaults:
**space-separated and quoted**, one row per field, quoted header:

```
"col.name" "col.class" "unique" "required" "multiplevalues" "description" "allowedvalues" "static.enum" "dynamic.enum" "dynamic.enum.property" "delimiter" "separator" "corpus.type"
"AGE" "integer" "non-unique" "optional" FALSE "Age at which the condition or disease was first diagnosed, in years." "[0-9]+" NA NA NA NA NA "integer"
```

Rules that every downstream script depends on (the full set is in
`scripts/README.md` §Notes):

- missing values are the literal `NA`;
- `|` is the **single** list delimiter — enum `allowedvalues`, positional
  `static.enum` ids, `dynamic.enum` roots, and compound `corpus.type` tokens
  all split on it;
- `;` / `<;>` in the `delimiter`/`separator` columns are data-cell separators,
  unrelated to list splitting;
- `is_numeric_field` is `yes` for `col.class` ∈ {`integer`, `double`,
  `numeric`}, else `no`; `field_name` is preserved verbatim from `col.name`.

Don't "fix" the quoting, switch to commas, or normalize `NA` to empty strings —
parsing this exact shape is assumed by both builder toolchains and by
SchemaMapper.

## Checking parity

Because the R and Python builders are hand-synced and there is no CI: after
touching a builder, run **both** and diff (a) against each other and (b)
against the artifacts committed under `schema/<prefix>/`. A difference between
the two builders is a bug. A difference between fresh output and the committed
artifacts is a **finding** — report it in the PR or an issue rather than
committing regenerated artifacts; that difference is information.

## Scope of PRs

- Documentation and dependency declaration: always welcome.
- Builder-logic changes and regenerated artifacts: separate PRs, with the
  parity check above attached.
- No opportunistic reformatting of committed schema artifacts.