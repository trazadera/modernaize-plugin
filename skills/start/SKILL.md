---
name: start
description: Check that the ModernAIze connection works after installing the plugin, and list the user's projects. Read-only.
disable-model-invocation: true
---

# Check the ModernAIze connection

This is a read-only check. Follow these steps in order.

1. If the ModernAIze tools are not available in this session, tell the user the connection failed
   and how to fix it: open `/mcp`. A 401 means the token is wrong, expired or revoked: create a
   new one in ModernAIze → Settings → Access tokens and set it with `/plugin configure modernaize@trazadera`.
   A connection error means the URL is wrong: fix it with the same command.
2. Otherwise call `list_projects` once. If it fails with an authentication error, give the same
   token message as in step 1.
3. On success, greet the user and show their projects as a short table with name, source system
   and tilStatus. Then suggest three prompts based on where the projects stand:
   - no projects, or tilStatus NONE: "upload this folder as a new project";
   - PARSED or STALE: "enrich <project>" (leads to `trigger_enrichment`);
   - ENRICHED or ENRICHED_PARTIAL: "explain what <project> does" (leads to `understand_briefing`)
     and "propose a new architecture for <project>" (leads to `list_proposal_options`);
   - "generate code from the latest proposal of <project>" (leads to `list_proposals` and
     `list_output_options`) for a project the user says already has a proposal;
   - PARSE_FAILED or ENRICHMENT_FAILED: "why did <project> fail?" (leads to `get_til_status`).
4. Never call a tool that changes anything.
