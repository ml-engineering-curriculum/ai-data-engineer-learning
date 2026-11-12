# Chapter 03 — lakeFS: Branch, Merge, Rollback on Object Storage

## Why this chapter exists

lakeFS (Treeverse, Apache 2.0) is the substrate you reach for when the dataset lives in object storage at a size where copying it to a laptop is not an option, when multiple teams need to read and write concurrently, and when the natural mental model is **git-for-your-data-lake**: branches you develop on, merges that promote to `main`, atomic rollbacks when something goes wrong. It differs from DVC (Chapter 02) in two load-bearing ways:

1. **Branches operate directly on the object store.** Branching is O(1) and copy-on-write — creating a dev branch of a 100 TB corpus does not duplicate the 100 TB.
2. **The substrate is the object store itself.** lakeFS exposes the bucket through an S3-compatible API with the `lakefs://repo/branch/path` scheme; anything that speaks S3 (Spark, Trino, Dask, Ray, Hugging Face Datasets with an S3 endpoint override, `aws s3 cp`) can read and write through it without code changes.

Primary sources: <https://docs.lakefs.io/>, <https://github.com/treeverse/lakeFS>.

## The lakeFS model in one page

The core objects:

- **Repository** — the top-level namespace. One per data domain. Backed by exactly one *storage namespace* (an S3 / GCS / Azure Blob prefix).
- **Branch** — a mutable reference to a commit. `main` is created at repository creation. Branches are O(1); the branch's bytes are derived from the underlying commit's manifest and any writes staged on the branch.
- **Commit** — an immutable content-addressed snapshot of the repository's tree. Commits have a SHA-256 ID, a parent (or two parents for a merge commit), a message, and metadata (key-value pairs).
- **Tag** — an immutable reference to a commit. Used for releases (`v1.2.0`) and policy pins.
- **Object** — a file in the repository at a `path`. Stored in the underlying object store under a content-addressed key.
- **Hook** — an event handler (pre-commit, pre-merge, post-commit, post-merge) that can block the operation. The policy-enforcement surface.

The *copy-on-write* semantics are the key difference from DVC. When you create a branch, lakeFS writes a new branch pointer referencing the current commit's manifest. No bytes are copied. When you write an object on the branch, lakeFS writes the new object's bytes once (content-addressed) and records the branch's uncommitted state. Committing produces a new manifest referencing the new object plus every unchanged object inherited from the parent commit.

The underlying object store sees content-addressed keys; the `lakefs://` URL is a *logical path* that lakeFS resolves at read time.

## When lakeFS fits

lakeFS fits when:

- The dataset lives in **object storage** at volumes of **O(TB)–O(PB)**. DVC's push / pull cycle becomes impractical here; lakeFS keeps the data in-place and overlays branch semantics.
- **Multiple teams or pipelines** write to overlapping prefixes. lakeFS's branch / merge / commit model is the primitive that keeps them from stomping on each other.
- You need **atomic rollback** at the data-lake level. A bad pipeline run that wrote 20 million garbage objects can be reverted with a single commit revert, not by scripting an object-list-and-delete.
- The workflow is **S3-compatible tooling** — Spark, Trino / Presto, Dremio, Dask, Ray, Hugging Face Datasets, `duckdb`, `pyarrow`, `aws s3 sync`. lakeFS slots in as an S3 endpoint; the consumer code does not change.
- You want **CI/CD-style gates on data writes** — pre-commit and pre-merge hooks that run data-quality, schema-conformance, or licence-policy checks before anything lands on `main`.

lakeFS does **not** fit (or fits less well) when:

- The dataset is **small enough that DVC's Git-shaped UX wins on simplicity** (sub-100 GB, single-researcher iteration).
- The team is **lakehouse-native** — Delta / Iceberg (Chapter 05) provides branch-like semantics (via Nessie or Iceberg table branches) with better integration into the query engines.
- The primary consumer is a **row-oriented analytical query** — Delta / Iceberg will serve this workload better than lakeFS over Parquet.
- The pipeline needs **datum-level incremental semantics** — Pachyderm (Chapter 04) is purpose-built for this.

## Setup and a worked example

A minimal lakeFS repository workflow, with the `lakectl` CLI and the S3 endpoint override:

```bash
# bootstrap a repo
lakectl repo create \
  lakefs://sft-chat-owned-ugc \
  s3://example-lakefs-storage/sft-chat-owned-ugc \
  --default-branch main

# create a development branch
lakectl branch create \
  lakefs://sft-chat-owned-ugc/dev-q3-2026 \
  --source lakefs://sft-chat-owned-ugc/main

# upload objects using the S3-compatible gateway
export AWS_ENDPOINT_URL=https://lakefs.example.com
aws s3 cp \
  prepared/ \
  s3://sft-chat-owned-ugc/dev-q3-2026/prepared/ \
  --recursive

# commit
lakectl commit \
  lakefs://sft-chat-owned-ugc/dev-q3-2026 \
  --message "feat(prepared): add Q3-2026 prepared slice after CCNet + trafilatura" \
  --metadata pipeline_git_sha=f0e9d8c7b6a5948372615049382716054938271 \
  --metadata firewall_version=firewall-v2026-07-15 \
  --metadata pii_recognisers=presidio-recognisers-v2026-06-30

# see what changed
lakectl diff lakefs://sft-chat-owned-ugc/main lakefs://sft-chat-owned-ugc/dev-q3-2026

# merge to main (gated by hooks)
lakectl merge \
  lakefs://sft-chat-owned-ugc/dev-q3-2026 \
  lakefs://sft-chat-owned-ugc/main \
  --message "release(sft-chat-owned-ugc): v1.2.0"

# tag the release
lakectl tag create \
  lakefs://sft-chat-owned-ugc/v1.2.0 \
  lakefs://sft-chat-owned-ugc/main
```

The equivalent in Python, using `lakefs-client`:

```python
from lakefs_client import ApiClient, Configuration
from lakefs_client.api import branches_api, commits_api
from lakefs_client.models import BranchCreation, CommitCreation

cfg = Configuration(host="https://lakefs.example.com/api/v1")
cfg.username = os.environ["LAKEFS_ACCESS_KEY_ID"]
cfg.password = os.environ["LAKEFS_SECRET_ACCESS_KEY"]

with ApiClient(cfg) as api:
    branches = branches_api.BranchesApi(api)
    commits = commits_api.CommitsApi(api)

    branches.create_branch(
        repository="sft-chat-owned-ugc",
        branch_creation=BranchCreation(name="dev-q3-2026", source="main"),
    )

    # ... run the pipeline, writing to lakefs://sft-chat-owned-ugc/dev-q3-2026/ ...

    commits.commit(
        repository="sft-chat-owned-ugc",
        branch="dev-q3-2026",
        commit_creation=CommitCreation(
            message="feat(prepared): add Q3-2026 prepared slice",
            metadata={
                "pipeline_git_sha": "f0e9d8c7b6a5948372615049382716054938271",
                "firewall_version": "firewall-v2026-07-15",
                "pii_recognisers": "presidio-recognisers-v2026-06-30",
            },
        ),
    )
```

The commit metadata map is deliberately open-ended. Use it to pin snapshot-identity fields (Chapter 01): pipeline Git SHA, policy versions, upstream snapshot IDs, Rekor signature references. Treat the metadata as part of the commit's identity — hooks can and should validate it.

## Branches — the branching patterns to know

Four patterns cover ~95% of production use:

### 1. Dev-branch + merge-to-main

The default. Pipelines write to a dev branch (`dev-q3-2026`, `experiment-dedup-threshold-0.9`); human review + hook gates merge to `main`. Analogous to GitHub pull-request flow. Dev branches are cheap; create one per logical change.

### 2. Date-stamped release branches

A branch per data refresh (`release-2026-10-15`, `release-2026-11-01`). Useful when downstream consumers want to pin to "the Q4 release" and move forward on their own cadence. Tags on commits are the companion pattern — `v1.2.0`, `v1.3.0-rc1`.

### 3. Experiment branches with disposable lifecycle

Short-lived branches (`exp-dedup-threshold-0.75`, `exp-boilerplate-strip-ablation`) for A/B data experiments. The downstream model-eval workflow trains on the experiment branch's snapshot and reports eval deltas. Clean up expired experiment branches on a cadence; `lakectl branch delete` reclaims no storage (bytes are content-addressed) but removes the pointer and the hook evaluation surface.

### 4. Isolated-ingest branches

An ingest pipeline that reads from an upstream system writes to a *private* branch, runs end-to-end, and merges to `main` only after all hooks pass. The branch is the staging area; `main` is the audited source of truth. This is the pattern most organisations settle on as the production shape.

## Merge, revert, rollback — the three operations that matter

**Merge.** `lakectl merge <source> <destination>` creates a merge commit on the destination. Conflicts on identical paths with different content are flagged; lakeFS's conflict-resolution model is three-way merge on the object level (not byte level), so two branches that edit the *same* object produce a conflict, but two branches that edit *different* objects do not.

**Revert.** `lakectl revert <branch> <commit>` creates a new commit on the branch that *undoes* the changes of a prior commit. Analogous to `git revert`; the history is preserved and the operation is atomic.

**Rollback (branch reset).** `lakectl branch reset <branch> --commit <ref>` moves the branch pointer back to an earlier commit, orphaning the commits in between. Use with care; the orphaned commits are not reachable from any branch but remain in the commit log.

For AI training-data operations, the operational discipline is:

- Prefer *revert* to *rollback*. A revert preserves auditability; a rollback hides it.
- Pair *merge* with a *tag* for anything a downstream consumer will pin to. The tag is the stable name; the commit SHA is the resolve-to identity.
- Keep `main` permanently monotonic — reverts advance, never rewind. Downstream snapshots pin to commit SHAs or tags; a `main` that rewinds breaks them.

## Hooks — pre-commit and pre-merge policy gates

lakeFS hooks are server-side event handlers that run on commits, merges, and branch operations. Supported implementations include `webhook` (POST to an HTTP endpoint) and `lua` (an in-process Lua script). Common AI-pipeline patterns:

- **Pre-commit: schema check.** Validate that every Parquet file in the write set matches an expected schema. Block the commit if schemas drift without an explicit `schema_bump` metadata key.
- **Pre-commit: data-quality check.** Run a Great Expectations suite or a Soda scan against the write set. Block on P0 violations.
- **Pre-commit: licence posture check.** Validate that every object carries a licence tag in its object metadata or that its prefix maps to an allowlisted licence (mod-102).
- **Pre-merge: contamination firewall.** Re-run the mod-106 firewall across the diff between source and destination branches; block if any contaminated examples would land on `main`.
- **Pre-merge: PII scrub proof.** Require that the source branch's commit metadata references a passed PII-scrub report (mod-110).
- **Post-commit: OpenLineage emission.** Emit an OpenLineage event describing the commit as a *run* with the new snapshot as its *output*. Chapter 06.

Hook configuration is YAML in a `_lakefs_actions/` prefix on the repository itself:

```yaml
# _lakefs_actions/pre-merge-firewall.yaml
name: contamination-firewall-check
on:
  pre-merge:
    branches:
      - main
hooks:
  - id: firewall-check
    type: webhook
    description: "Re-run the mod-106 firewall diff on the merge payload"
    properties:
      url: https://ci.example.com/hooks/firewall-check
      timeout: 600s
      query_params:
        strict: "true"
```

The webhook handler is a service your team owns; the hook contract is "block the merge if the response is non-200." This is the lakeFS-native way to *enforce* pipeline policies — not run them as advisory steps a human might skip.

## Snapshot identity under lakeFS

Mapping Chapter 01's five snapshot-identity fields to lakeFS:

| Identity field | lakeFS primitive |
|---|---|
| Content-addressed hash | The commit ID is a SHA-256 over the commit's tree manifest. A snapshot-level SHA-256 Merkle root is derived from the manifest at snapshot time. |
| Substrate identifier | `lakefs://<repo>/<commit-id>` or `lakefs://<repo>/<tag>`. |
| Pipeline version | Commit metadata key `pipeline_git_sha`. |
| Policy versions | Commit metadata keys `licence_policy`, `firewall_version`, `pii_recognisers`, etc. |
| Signatures + attestations | External — `cosign sign-blob` over the commit's Merkle root; the resulting Rekor entry referenced in the commit metadata. |

A pinned snapshot URL for downstream consumers is a tag or commit SHA. Treat the tag as the stable *name* and the commit SHA as the resolve-to identity; both resolve to the same manifest.

## Lineage integration — lakeFS and OpenLineage

lakeFS does not ship a first-party OpenLineage adapter (it is not listed in the OpenLineage integrations index at [openlineage.io/docs/integrations](https://openlineage.io/docs/integrations/)); the integration is a self-authored webhook pattern on top of lakeFS's Lua/HTTP action hooks ([lakeFS hooks docs](https://lakefs.io/blog/lakefs-hooks/)). A post-commit or post-merge webhook can translate the commit event into an OpenLineage `COMPLETE` event, with:

- `job.name` = `lakefs.commit.<repo>.<branch>` or `lakefs.merge.<repo>.<source>.<destination>`
- `job.namespace` = the repo name
- `run.runId` = a stable UUID derived from the commit ID
- `outputs[]` = a dataset facet for the new commit's `lakefs://<repo>/<commit-id>` path
- `inputs[]` = dataset facets for every parent commit referenced in the merge

The practical shape: the pipeline that *writes* to lakeFS emits OpenLineage events for its own steps (via Chapter 06's integration); the *post-merge* lakeFS webhook emits a *release* event describing the version promotion. Both end up in Marquez or DataHub; the graph becomes queryable.

## Operating notes

- **Garbage collection.** lakeFS's objects are content-addressed on the underlying bucket. A branch deletion does not free bytes; `lakectl gc` scans for objects not reachable from any commit and schedules them for deletion after a retention window. Run it on a cadence.
- **Underlying bucket versioning.** Enable S3 / GCS versioning on the storage namespace. lakeFS's model assumes objects are immutable once written; versioning makes that assumption enforced rather than conventional.
- **Access control.** lakeFS has its own RBAC layer with ACLs on repositories, branches, and paths. Do not rely on the underlying bucket's IAM for human-level access; lakeFS's ACLs operate at commit / branch / path granularity.
- **Protected branches.** Mark `main` and any release branch as *protected* to require pull-request-style merges and to disallow direct writes.
- **Scale.** lakeFS scales to petabyte-scale stores; the commit log scales to hundreds of millions of objects per commit before manifest compaction matters. For truly huge trees (billions of objects per commit), the lakehouse substrate (Chapter 05) becomes the better fit — objects there are table rows, not S3 keys.

## Peer uses in the track — where this substrate lands

- A multi-TB pretraining corpus from mod-103 (a CCNet-style slice of Common Crawl) is a canonical lakeFS fit.
- A shared crawl + curation staging area where ingestion, quality-check, dedup, and firewall stages run on their own branches is a canonical lakeFS fit.
- A labelling-vendor feed where multiple vendors write to vendor-specific branches (`vendor-scale`, `vendor-surge`) and QA merges to a consolidated `main` is a canonical lakeFS fit.
- A benchmark fixture small enough to carry on DVC is **not** a lakeFS fit; the operational overhead is not justified.

## Summary

- lakeFS gives you **git-for-your-data-lake** semantics directly on S3 / GCS / Azure Blob, with O(1) copy-on-write branching.
- The core model: **repositories → branches → commits → objects**, with **tags** for stable release names and **hooks** for pre-commit / pre-merge gates.
- Branches are **free** (storage-wise); use them liberally. Tag releases; prefer **revert** over **rollback** for auditability; keep `main` monotonic.
- **Hooks are the policy-enforcement surface.** Pre-merge contamination firewall, pre-commit PII-scrub proof, pre-commit licence posture — these turn mod-102 / mod-106 / mod-110 policies from advisory into enforced.
- Snapshot identity maps to `(repo, commit-id)` or `(repo, tag)`; commit metadata carries policy and pipeline pins.
- lakeFS fits **object-store-native, multi-team, petabyte-scale** workloads. It does not displace DVC for small-team workflows or the lakehouse substrates for table-shaped data.

Chapter 04 picks up the third filesystem substrate — Pachyderm — where the substrate is organised around *versioned pipelines* rather than versioned data alone, and datum-level incremental semantics turn re-processing into a targeted operation.
