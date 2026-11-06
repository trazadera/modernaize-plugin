# ModernAIze plugin for Claude Code

Connects Claude Code to your ModernAIze environment and teaches the agent how to use it: create
projects, upload legacy source code, generate and enrich TIL, understand the system, and produce
generated outputs.

## Install

```bash
claude plugin marketplace add trazadera/modernaize-plugin
claude plugin install modernaize@trazadera
```

## Configure

Start Claude Code. It asks for the MCP URL and your token. If it does not, run
`/plugin configure modernaize@trazadera`.

Both values are on the ModernAIze page **Settings → Access tokens**: copy the MCP URL from the
"Connect your agent" block and create a token with the "Create token" button (a token is shown
only once). The token is marked sensitive: Claude Code keeps it in the operating system credential
store, not in a settings file.

Then run `/modernaize:start` to check the connection. It lists your projects and changes nothing.

## Update

```bash
claude plugin marketplace update trazadera
claude plugin update modernaize@trazadera
```

Third-party marketplaces do not update automatically unless you turn that on.

## Uninstall

```bash
claude plugin uninstall modernaize@trazadera
claude plugin marketplace remove trazadera
```

## What it contains

- An MCP server entry for your ModernAIze environment, with a 30 minute timeout so long uploads
  and parsing are not cut off by the client.
- The `modernaize` skill, which teaches the agent the tools and the safe way to use them.
- The `/modernaize:start` command.

It contains no hooks and no executable code.

## Privacy

The plugin sends nothing anywhere by itself. When the agent calls a tool, the request goes to the
MCP URL you configured, with your token, and the answer comes back to your Claude Code session.
Source files you ask the agent to upload are sent to that ModernAIze environment.

## Licence

Apache License 2.0, see `LICENSE`.
