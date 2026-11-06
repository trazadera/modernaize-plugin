---
name: modernaize
description: Use when the user works with ModernAIze, TIL (Trazadera Intermediate Language), legacy program migration or modernization, ModernAIze projects, the Understand views, enrichment, blueprints or generated outputs. Teaches how to drive the ModernAIze MCP tools.
---

# Working with ModernAIze

ModernAIze converts legacy programs (SAS, Qlik, Informatica, DataStage, COBOL and others) into
TIL, a semantic JSON description of what the program does, and then helps produce a modern
implementation from it. The stages are:

1. **Parse**: source files become a TIL snapshot (phase "source").
2. **Enrichment**: the TIL is enriched with derived knowledge (phase "enriched").
3. **Proposal**: a design for the new system is proposed from the enriched TIL.
4. **Output**: code is generated from the proposal with a blueprint.

You reach all of it through the ModernAIze MCP tools. Every tool runs with the permissions of the
token the user configured, so a refusal is the product's answer, not a bug.

## Recommended flow

1. `list_projects` to see what exists. Reuse a project before creating a new one.
2. `create_project` (needs a name and a sourceLanguage) when the user has new source code.
3. Upload everything first: `upload_source_zip` for many files, `upload_source_files` for up to 20
   small text files. Uploads do not start enrichment by default.
4. `generate_til` once, after all uploads, to parse the whole project.
5. `trigger_enrichment`, then poll `get_til_status` every 10 to 30 seconds until the verdict is
   terminal.
6. Understand the system with the `understand_*` tools and the `til_*` tools below.
7. `list_blueprints`, then `create_output` with an eligible blueprint, then poll
   `get_output_status`. Read the results with `list_output_files`, `get_output_file_content` and
   `get_quality_report`.

Worked example: "migrate this folder of DataStage jobs". Call `list_projects` to check for an
existing project, `create_project` with sourceLanguage DATASTAGE, zip the folder and call
`upload_source_zip`, then `generate_til`, `trigger_enrichment` and `get_til_status` until
enrichment finishes. Call `understand_briefing` for the overview, `understand_search` to find the
biggest jobs, and `understand_construct` on each. When the user is happy, call `list_blueprints`,
pick an entry with eligibleForOutput true, pass it to `create_output` with an explicit proposalId,
and poll `get_output_status`.

## Projects and uploads

- `list_projects`: the project catalog, newest first, paginated (page, size).
- `create_project`: new source project. Needs EDITOR.
- `delete_project`: permanently deletes a project and its files. Needs ADMIN and the argument
  confirm set to true, only after the user explicitly agrees.
- `upload_source_files`: up to 20 inline text files, 1 MB each. Needs EDITOR.
- `upload_source_zip`: a base64 ZIP archive, the better choice for many files. Needs EDITOR.
- `generate_til`: parses every source file in the project. It blocks until parsing is done. Needs
  EDITOR.
- `trigger_enrichment`: starts enrichment asynchronously. A reply saying it is already running is
  a normal answer, not an error. Needs EDITOR.
- `get_til_status`: parse and enrichment status with a single verdict. PARTIAL is terminal with
  warnings: report it as is. A "not started" verdict can be transient, so check once more after
  about 10 seconds.

## Outputs

- `list_blueprints`: the system and personal blueprints. Only entries with eligibleForOutput true
  can be used with `create_output`.
- `create_output`: starts code generation. Needs EDITOR. Always pass proposalId explicitly, or the
  tool may pick a different proposal than the user expects.
- `get_output_status`: the authoritative status and any error message of an output.
- `list_output_files`: names, paths and per-file status, without contents.
- `get_output_file_content`: the content of one file, by its exact path from the file list.
- `get_quality_report`: the quality report of an output. Needs EDITOR.
- `delete_output`: permanently deletes an output. Needs EDITOR and confirm set to true, only after
  the user explicitly agrees.

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

TIL snapshots have a phase (source, enriched, system-definition, proposal-target,
proposal-overlay) and a version (default latest). Not every tool supports every phase; an
unsupported phase answers PHASE_NOT_SUPPORTED.

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
- `til_provenance`: where a construct comes from and what became of it in the outputs.
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

- `listProjects`: source projects together with their output projects and identifiers.
- `whoIsThisFile`: the constructs and source references behind a generated file or model.
- `explainDecision`: the proposal decision behind a target construct or model.
- `querySourceSemantics`: source semantics by construct id or name query.

## Rules for agents

- **Roles.** Reading needs a USER token. Changing things needs EDITOR. Deleting a project needs
  ADMIN. FORBIDDEN on a write tool is expected for a read-only user: tell the user which role is
  missing and stop, without trying workarounds.
- **Never retry blindly after a timeout.** `upload_source_files`, `upload_source_zip`,
  `generate_til` and `create_output` may have succeeded on the server even if the call timed out.
  Read the status and the file listing first, and compare with what you sent.
- **Confirm before deleting.** Ask the user, then pass confirm set to true.
- **Large replies.** If a reply is too large, the client saves it to a file. Read that file in
  parts, or ask for less with a narrower path, id, depth, offset or limit.
- **The token is a secret.** Never print it, echo it, write it to a file or commit it.
- **Report what the tools say.** Quote statuses and error codes as they are, and say so when
  something is not recorded rather than guessing.
