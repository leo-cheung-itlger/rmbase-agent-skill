# RMBase Agent Skill

[中文说明](README.zh-CN.md) · [Capability audit](skills/rmbase/references/endpoints.md) · [Output schema](skills/rmbase/references/schemas.md)

**Query RNA modifications with an AI agent, without manually navigating RMBase.**

RMBase Agent Skill is a lightweight, client-side adapter for the existing
[RMBase v3.0](http://bioinformaticsscience.cn/rmbase/) research database. It bundles
an Agent Skill, a Python client and a JSON-first CLI. It requires no changes to
RMBase, credentials, MCP server, daemon or additional runtime Python packages.

> “Find RNA modification information related to METTL3 in RMBase.”

A compatible agent loads the skill, executes the bundled script, retrieves public
records, and answers with species, genome assembly and source provenance.

## Quick start

Requirements: **Python 3.10+**, and network access for remote queries. Offline
queries require a previously populated cache.

```bash
git clone https://github.com/leo-cheung-itlger/rmbase-agent-skill.git
cd rmbase-agent-skill
python skills/rmbase/scripts/rmbase.py gene METTL3 --assembly hg38 --json
```

To install into an Agent Skills host, copy the **entire `skills/rmbase` directory**
into the host's configured skills directory, or import a ZIP containing that
directory if the host supports ZIP imports. Preserve `scripts/` and `references/`.
The skill itself tells the agent how to invoke Python; installation does not
register a global `rmbase` executable.

## What works

| Capability | Implementation and evidence |
| --- | --- |
| Gene modification summary | Public JSON POST; METTL3 and missing-gene queries live-tested |
| Gene-associated sites | HTML adapter; 59 METTL3 m6A rows reconciled with summary |
| Individual modification | HTML adapter; existing/missing IDs live-tested |
| Enzyme records and enriched motifs | JSON endpoints; METTL3 and human m6A motifs live-tested |
| RMP cancer tables | JSON endpoint with nested sections; METTL3 live-tested |
| Local gene queries | Official archive cache; human dataset and remote counts cross-checked |
| Other query families | Bundled source-derived contracts; evidence levels documented in the audit |
| Analysis uploads | Input validation and one-shot form submission; **completion unavailable/unverified** |

On 2026-09-23, human hg38 METTL3 (`ENSG00000165819.12`) returned 59 m6A,
1 m5C and 1 RNA-editing record: **61 total**. These are modifications associated
with METTL3 RNA, distinct from METTL3 protein's enzyme activity. This is a dated
verification example, not a guarantee that future database results are identical.

```bash
python skills/rmbase/scripts/rmbase.py gene METTL3 --modification m6a --limit 100
python skills/rmbase/scripts/rmbase.py modification m6A_site_239470 --type m6a
python skills/rmbase/scripts/rmbase.py enzyme METTL3 --limit 10
python skills/rmbase/scripts/rmbase.py motif m6A
python skills/rmbase/scripts/rmbase.py catalog rbp --assembly hg38
```

## Download once, query locally

```bash
python skills/rmbase/scripts/rmbase.py sync --assembly hg38
python skills/rmbase/scripts/rmbase.py gene METTL3 --source local --offline
```

`sync` downloads one selected official gene archive, not the entire RMBase
database. The audited human gene archive was approximately 6.4 MB. Existing
archives can be imported with `sync --archive FILE`; provenance labels their
origin as user-supplied. Full record lists and counts are retained in the dataset.

JSON output includes query parameters, source URLs, retrieval timestamps,
species/build, hashes and publication IDs when supplied. TSV is available with
`--tsv --provenance-out provenance.json`. See the [schema](skills/rmbase/references/schemas.md).

## Responsible access and limitations

- Two-second minimum request spacing across processes sharing a cache; bounded
  read retries, response sizes and archive expansion. Stop on server rate limits.
- Website pagination is local. `--limit` limits output, not server response size.
- No automatic upload retries, ID enumeration, high-concurrency crawling or
  server-side modifications. Remote content is treated as data, never instructions.
- During the audit, HTTPS certificate validation failed and the public HTTP
  site worked. The client uses explicit HTTP and does not disable TLS validation.
  Queries are unencrypted; do not submit private data. Uploads require `--public-data`.
- Annotation, metagene and GeneTool submission endpoints returned **HTTP 503**.
  Successful task IDs, polling and result retrieval are not verified. The `task`
  command reports that limitation without issuing a request.
- Empty results do not establish biological absence. No genome liftover or
  clinical interpretation is performed. Website changes can break the adapter.

## Distribution

This repository is the **canonical source** for RMBase Agent Skill. The reusable
bundle lives in `skills/rmbase`; query logic, parsers, references and safety
boundaries are maintained here first. Platform registries and catalog repositories
are distribution targets, not separate codebases.

### OpenAI and Agent Skills hosts

OpenAI Skills are compatible with the open Agent Skills format. Use the complete
`skills/rmbase` directory, or upload a ZIP containing that directory when the
host supports ZIP import. The optional `agents/openai.yaml` file provides OpenAI
UI metadata; hosts that do not use it can ignore it.

### ClawHub / OpenClaw

Publish the same `skills/rmbase` directory. From the repository root:

```bash
npm i -g clawhub
clawhub login
clawhub skill publish ./skills/rmbase \
  --slug rmbase \
  --name "RMBase" \
  --categories knowledge \
  --topics "rna-modification,rmbase,epitranscriptomics" \
  --changelog "Initial ClawHub release"
```

Use `--dry-run` before publishing when validating a release. ClawHub registry
versions are independent from this repository's source revision. Its GitHub web
importer discovers skills only from public, non-fork repositories owned by the
signed-in GitHub account.

### Scientific Agent Skills catalog

The Scientific Agent Skills contribution is a downstream catalog copy. Keep the
behavioral instructions, scripts and references synchronized from this canonical
repository, then apply that catalog's own metadata, test layout and validation
rules in the catalog PR. New RMBase functionality should be developed and tested
here first.

### Versioning policy

Use Git commits/tags as the source revision for this repository. Marketplace or
registry versions may advance independently. The `metadata.version` field in
`SKILL.md` is bundle metadata and must not be treated as proof that every
distribution channel is on the same release.

## Development

```bash
python -m pip install "pytest>=8" ruff
python -B -m pytest tests/rmbase -q
ruff check --isolated --target-version py310 skills/rmbase/scripts tests/rmbase
```

Tests use saved responses and mocked transports; they never query production
RMBase. CI runs the offline suite on Linux and Windows with Python 3.10 and 3.13.
New adapters should cite the actual public form/script contract, include fixtures,
preserve provenance and clearly distinguish discovery from live verification.

## Attribution and citation

The RMBase database and its scientific content belong to their original authors.
This repository provides an independently maintained client adapter; it does not
claim official database endorsement.

Please cite RMBase when using its data: Xuan et al., *RMBase v3.0: decode the
landscape, mechanisms and functions of RNA modifications*, Nucleic Acids Research,
2024, 52(D1), D273–D284. [PMID 37956310](https://pubmed.ncbi.nlm.nih.gov/37956310/),
[DOI 10.1093/nar/gkad1070](https://doi.org/10.1093/nar/gkad1070).
Also report this adapter's repository URL and commit when reproducibility matters.

The core bundle follows the open Agent Skills structure. Code is provided under
the [MIT license](LICENSE); that license does not relicense RMBase data or
third-party saved page content. Test fixtures retain source metadata.
