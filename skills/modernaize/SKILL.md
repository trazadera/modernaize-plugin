---
name: modernaize
description: Use when the user works with ModernAIze, TIL (Trazadera Intermediate Language), legacy program migration or modernization, ModernAIze projects, the Understand views, enrichment, proposals or generated outputs. Teaches how to drive the ModernAIze MCP tools.
---

# Working with ModernAIze

ModernAIze converts legacy programs (SAS, Qlik, Informatica, DataStage, COBOL and others) into
TIL, a semantic JSON description of what the program does, and then helps produce a modern
implementation from it. The stages are:

1. **Parse**: source files become a TIL snapshot (phase "source").
2. **Enrichment**: the TIL is enriched with derived knowledge (phase "enriched").
3. **Proposal**: the Proposal Worker designs the new system from the enriched TIL with an
   architecture you choose. A proposal is a series of revisions (phase "system-definition").
4. **Output**: the Output Worker generates code from one exact proposal revision into a
   destination (dbt or Java), has it reviewed and repairs it.

You reach all of it through the ModernAIze MCP tools. Every tool runs with the permissions of the
token the user configured, so a refusal is the product's answer, not a bug.

## Recommended flow

1. `list_projects` to see what exists and where each project stands (tilStatus). Reuse a project
   before creating a new one.
2. `create_project` (needs a name and a sourceLanguage) when the user has new source code.
3. Upload everything first: `upload_source_zip` for many files, `upload_source_files` for up to 20
   small text files. Uploads do not start enrichment by default.
4. `generate_til` once, after all uploads, to parse the whole project.
5. `trigger_enrichment`, then poll `get_til_status` every 10 to 30 seconds until the verdict is
   ENRICHED (or ENRICHED_PARTIAL, PARSE_FAILED or ENRICHMENT_FAILED).
6. Understand the system with the `understand_*` tools and the `til_*` tools below.
7. Proposal: `list_proposal_options` to pick an architecture and a model alias, `create_proposal`,
   then `get_proposal_status` every 30 to 60 seconds until it is SUCCEEDED, FAILED or CANCELLED.
   `list_proposals` shows the proposal and its revisions.
8. Output: `list_output_options` to pick a destination, `create_output` with the proposal revision
   version, then `get_output_status` every 30 to 60 seconds until it is final. Read the results
   with `list_output_files`, `get_output_file_content` and `get_quality_report`.

Worked example: "migrate this folder of DataStage jobs to dbt on Snowflake". Call `list_projects`
to check for an existing project, `create_project` with sourceLanguage DATASTAGE, zip the folder
and call `upload_source_zip`, then `generate_til`, `trigger_enrichment` and `get_til_status` until
the verdict is ENRICHED. Call `understand_briefing` for the overview. Ask the user which
architecture they want; `list_proposal_options` lists them (for example a medallion one). Once the
user agrees to start it, call `create_proposal` with that architectureId and follow it with
`get_proposal_status`. When it has SUCCEEDED, `list_proposals` gives its revisions: take the last
revision version. Call `list_output_options`, pick the dbt destination whose platform is
snowflake, and, once the user agrees, call `create_output` with proposalVersion set to that
version and systemBlueprintId set to that destination. Follow it with `get_output_status`.

## Projects and uploads

- `list_projects`: the project catalog, paginated (page, size), with each project's tilStatus.
- `create_project`: new source project. Needs EDITOR.
- `delete_project`: permanently deletes a project and its files. Needs ADMIN and the argument
  confirm set to true, only after the user explicitly agrees.
- `upload_source_files`: up to 20 inline text files, 1 MB each. Needs EDITOR.
- `upload_source_zip`: a base64 ZIP archive, the better choice for many files. Needs EDITOR.
- `generate_til`: parses every source file in the project. It blocks until parsing is done. Needs
  EDITOR.
- `trigger_enrichment`: starts enrichment asynchronously. A reply saying it is already running is
  a normal answer, not an error. Needs EDITOR.
- `get_til_status`: where the TIL stands, as one verdict: NONE, PARSE_FAILED, PARSED, STALE,
  ENRICHING, ENRICHED, ENRICHED_PARTIAL or ENRICHMENT_FAILED. ENRICHED_PARTIAL is usable with
  warnings: report it as is. PARSED right after starting enrichment can be transient, so check once
  more after about 10 seconds.

## Proposals

A proposal needs an ENRICHED (or ENRICHED_PARTIAL) project.

- `list_proposal_options`: the architectures (id, name, description) and the model aliases with
  their level (high or medium) and the default. launchable false means no proposal can start now.
- `create_proposal`: starts the Proposal Worker with an architectureId, and optionally a
  modelAlias, a name and instructions. Needs EDITOR. This is a paid run: only start it when the
  user asked for it.
- `get_proposal_status`: the runs of a project, or one run by runId. QUEUED, LAUNCHING and RUNNING
  are in progress; SUCCEEDED, FAILED and CANCELLED are final. PAUSED means an operator stopped it
  at a save point and may resume it. error says why it failed.
- `list_proposals`: the proposals and their revisions. Pass a revision version to `create_output`.
  The NAKED entry is the enriched TIL, not a proposal.
- `cancel_proposal`: stops a running proposal. Needs EDITOR and confirm set to true, only after the
  user explicitly agrees.

## Outputs

- `list_output_options`: the destinations the Output Worker supports (dbt ones with a
  systemBlueprintId, Java ones with a frameworkRealizationId), the target platforms for Java, the
  user's own blueprints as optional style overrides, and the model aliases with their level.
- `create_output`: starts the Output Worker on one proposal revision (proposalVersion) and exactly
  one destination: systemBlueprintId (dbt, optional orchestration) or frameworkRealizationId (Java,
  optional targetPlatformId). Optional userBlueprintId, modelAlias and name. Needs EDITOR. This is a
  paid run: only start it when the user asked for it.
- `list_outputs`: every output of a project with its status.
- `get_output_status`: the authoritative status of an output. PENDING and GENERATING are in
  progress; COMPLETED, COMPILE_FAILED, ERROR and ABANDONED are final. worker.error says why a
  Worker failed; read it when errorMessage is empty. proposalVersion is the proposal revision the
  output was generated from. worker.phase PAUSED means an operator stopped
  the Worker at a save point: the output stays GENERATING until it is resumed.
- `cancel_output`: stops an output that is still generating. Needs EDITOR and confirm set to true,
  only after the user explicitly agrees.
- `retry_output`: generates a failed or abandoned output again, as a new output whose id it
  returns. Needs EDITOR. This is a paid run.
- `list_output_files`: names, paths and per-file status, without contents.
- `get_output_file_content`: the content of one file, by its exact path from the file list.
- `get_quality_report`: the quality report of an output. Needs EDITOR.
- `delete_output`: permanently deletes an output. Needs EDITOR and confirm set to true, only after
  the user explicitly agrees.

## Models

Proposals and outputs are made by Workers. You choose a Worker only by its model alias (for
example ultraC or ultraT) and see only its level (high or medium). Never ask which model, vendor
or agent is behind an alias: ModernAIze does not disclose it. Omit the alias to use the default.

## Understand views

Read-only, available to any user with read access. Each reply carries status and enrichmentState:
show them as they are. "pending_enrichment" means the answer does not exist yet. Only wait for it
if enrichmentState is not FAILED; if it is FAILED, enrichment must be run again.

- `understand_briefing`: "what does this system do?", the executive overview and effort index.
- `understand_landscape`: regions of the system and their effort.
- `understand_search`: find constructs by text and type. Start here to obtain a ref.
- `understand_construct`: details of one construct, by ref.
- `understand_source`: the source lines behind a construct, by ref, with optional from and to
  (give both or neither).

## TIL tools

TIL snapshots have a phase (source, enriched or system-definition) and a version (default
latest). A proposal revision is a system-definition snapshot: read it with phase
system-definition and the revision version from `list_proposals`. Any other phase answers
PHASE_NOT_SUPPORTED.

Reading:

- `til_spec`: the TIL specification itself. Same for every project.
- `til_summary`: precomputed summaries of a project or a scope.
- `til_search`: substring search over the entity index.
- `til_find`: search by text, column, dataset read or written, kind or SQL presence.
- `til_get`: one construct or a path inside the document, with depth and paging controls.
- `til_slice`: a scoped slice or raw JSON path of a snapshot.
- `til_navigate`: one lineage step (callers, dependents, flowsContaining).
- `til_lineage`: lineage around a node, at construct or field level, upstream or downstream.
- `til_explain`: one construct with its evidence and neighbours.
- `til_provenance`: where a construct comes from, and through which proposal revisions it reached
  which outputs.
- `til_diff`: compare two exact revisions.
- `til_assist`: focused explanation or suggested corrections on selected evidence.

Text that the enrichment authored (descriptions and intentions) is marked with its provenance.
When the provenance is "llm", say "the enrichment describes this as ..." and never present it as
a verified fact about the source system.

Editing a native system definition (needs EDITOR; checking needs USER):

- `til_create`: create a new system definition version from a complete TIL document.
- `til_check`: validate an exact revision, optionally with candidate changes, without writing.
- `til_apply_changes`: apply an ordered batch of changes to an exact revision, creating a new
  revision.

Always run `til_check` before `til_apply_changes`.

## Context tools

Questions about generated code and its origin:

- `listProjects`: the generated outputs, each with the source project it came from.
- `whoIsThisFile`: the constructs and source references behind a generated file or model.
- `explainDecision`: where a construct sits in the proposal the output was generated from
  (architecture, layer, rationale). By model name it answers directly when a generated file
  implements one of the matching constructs; otherwise it returns candidates: call it again with
  the constructId of the one you want.
- `querySourceSemantics`: source semantics by construct id or name query.

## Rules for agents

- **Roles.** Reading needs a USER token. Changing things needs EDITOR. Deleting a project needs
  ADMIN. FORBIDDEN on a write tool is expected for a read-only user: tell the user which role is
  missing and stop, without trying workarounds.
- **Never retry blindly after a timeout.** `upload_source_files`, `upload_source_zip`,
  `generate_til`, `create_proposal` and `create_output` may have succeeded on the server even if
  the call timed out. Read the status first (the file listing, `get_proposal_status` or
  `list_outputs`) and compare with what you sent, or you may start a second paid run.
- **Paid runs need the user's go-ahead.** `create_proposal`, `create_output` and `retry_output`
  start Workers that cost money and take minutes. Start one only when the user asked for it.
- **Confirm before deleting or cancelling.** Ask the user, then pass confirm set to true.
- **Large replies.** If a reply is too large, the client saves it to a file. Read that file in
  parts, or ask for less with a narrower path, id, depth, offset or limit.
- **The token is a secret.** Never print it, echo it, write it to a file or commit it.
- **Report what the tools say.** Quote statuses and error codes as they are, and say so when
  something is not recorded rather than guessing.
