# Agent integration plan

Status: proposal. This document records decisions and their reasons. It is not an implementation plan.

## Goal

Coding agents such as Claude Code run in a terminal beside the editor. They read files from disk, so they do not see unsaved changes, selections or language server diagnostics. The first goal is to give a running agent this editor state. The second goal is to let Mitos host an agent itself.

## Decision 1: One editor API, many adapters

All agent features use one API in a new crate. Adapters only translate between a client and this API, so one set of queries and tests serves every client. The first clients are `mitos ctl` and `mitos mcp`, through a socket. The ACP client comes later and calls the API inside the editor.

The API has queries (snapshots of editor state), edits and events (changes, merged over a short delay). Requests run one at a time on the editor thread through the existing callback channel, so each request sees a consistent state. Tools and resources live in one registry.

## Decision 2: One socket per workspace, with a stable ID

The socket lives in a private runtime directory and its ID comes from the workspace path. The first editor to acquire the workspace lock owns the socket. If the lock is held, subsequent editors log a warning and continue startup without a socket. They skip socket startup for the rest of that run. Several agents can connect to the owner and query its editor state. At process exit, including after a crash, the operating system releases the lock. A newly launched editor that acquires the lock can replace a stale socket and publishes a new generation so clients discard old handles.

The socket speaks a versioned Mitos JSON-RPC, not MCP. MCP stays in `mitos mcp`, so changes to the MCP specification do not reach the editor. A version mismatch between a new client and an old editor fails with a clear error.

## Decision 3: `mitos mcp` is a stdio server for each agent

Each agent starts its own `mitos mcp` over stdio, which every agent supports. A shared server needs supervision, a network transport and shared state. `mitos mcp` holds the state of one agent: its chosen editor and the last generation that it saw. It starts with no editor running and connects on the first call, because agents mark a server that fails at startup as broken. Several agents thus run side by side without shared state.

## Decision 4: The editor choice is explicit or unique

A tool argument, `--instance` or `MITOS_INSTANCE` selects a workspace socket and reaches its owning editor. Without an explicit selection, the client uses the socket if exactly one is available and keeps that choice for the session. Otherwise, the call fails and lists the available sockets. When Mitos starts an agent itself, it sets `MITOS_INSTANCE` only if it owns the socket. A hosted agent in a secondary editor uses the in-process ACP API for that editor. The hosted agent receives the MCP adapter only when its editor owns the socket.

## Decision 5: `@` resource references first, not `/ide`

MCP is pull-only. Mitos offers editor state, such as the selection and diagnostics, as MCP resources that you mention with `@`. You thus choose the context of each prompt.

A `UserPromptSubmit` hook that runs `mitos ctl context` is a later, optional addition. It sends context on every prompt, costs tokens, and you do not choose the content.

Mitos does not implement `/ide`, the private protocol of Claude Code. It has no public specification, works with one agent only, and needs a diff view that Mitos does not have.

## Decision 6: An ACP client comes second

With ACP (Agent Client Protocol), Mitos starts an agent and hosts it. Mitos serves file reads from buffers, applies writes as undoable edits, attaches the selection to prompts and asks you for permissions. If the editor owns the workspace socket, it gives the agent `mitos mcp --instance <id>`. ACP comes second because it needs a chat interface in the editor. It also cannot attach to an agent that already runs in a terminal.

## Decision 7: The agent crates follow the upstream plugin system

Upstream has no published plugin plan, so this plan assumes no plugin design. Until plugins exist, the agent crates use `Editor` and `view` through a thin layer. When upstream ships plugins, the agent crates move onto the plugin API as a built-in plugin. That move also removes most merge conflicts.

To prepare, registry entries have schemas that exist at runtime, and handlers can wait without blocking the editor thread. Only tools that their authors mark as exposed reach agents.

## Decision 8: Approve agent sessions at the editor socket

Socket access is opt-in through user-level configuration. The editor requires user approval for each connection before accepting requests. The first version grants read access. Later, write access requires a separate grant. Revoking approval closes the session, and disabling the integration closes the listener and all active connections. Workspace trust governs repository-controlled configuration and execution. It applies when repository-local settings can influence how Mitos launches a hosted agent. Socket access uses its own approval regardless of workspace trust.

On Linux, Mitos puts the Unix socket in an editor-user-owned directory with mode `0700` and restricts the socket to mode `0600`. Linux enforces these permissions at connection time. Processes running as the same user can still reach the socket unless the sandbox blocks access. The Unix socket uses filesystem permissions and editor session approval without a token. A TCP transport would require a token because filesystem permissions do not control TCP connections.

For a sandboxed agent, a trusted host launches `mitos mcp` outside the sandbox and gives the agent the MCP stdio channel. The sandbox must deny direct access to the socket path, prevent the agent from launching an unsandboxed helper, and keep the connected socket descriptor out of the agent process. The helper connects to the editor, which enforces the session's approved permissions. The sandbox is what makes the helper the only route to the socket for that agent.

Mitos marks returned content as data because files can contain instructions that target the agent. Limits on message size, connections and request time keep a slow client from stalling the editor.

## Decision 9: Writes carry the version that they read

The first version is read-only. Agents write files on disk, and editor queries return the current in-memory buffer. Auto-reload may be disabled, may still be pending, or may leave a buffer with unsaved changes open for the user to reconcile. Use a disk read to verify a disk write. If the next step depends on the editor reflecting that write, ask the user to reload or reconcile the buffer. The first version leaves reload decisions to the editor and the user.

Later, each write carries the document version that the agent read, and the editor rejects a write to a changed buffer. This rule covers two agents, or you and an agent, on one buffer. Each write is a normal undoable edit, and the agent crate records which agent made it.

## Implementation guidance: stay apart from upstream code

This fork merges upstream often, and each changed upstream line is a possible conflict. New files do not conflict.

1. Put all agent code in new crates, behind one cargo feature. A build without the feature matches upstream.
2. Call `Editor` and `view` only from the thin layer of Decision 7. When upstream ships plugins, only this layer changes.
3. Do not change existing types, signatures, handlers, keymaps or commands. Register hooks from the new crates.
4. Keep agent configuration in its own file, because the shared configuration types reject unknown fields.
5. Follow upstream plugin work, and do not build a second extension mechanism in this fork.
6. Keep each required upstream change, such as startup and subcommand dispatch, to one short block. Offer missing general interfaces upstream.

## Order of work

1. The editor API, the socket, `mitos ctl` and `mitos mcp`.
2. MCP resources for `@` references.
3. Write tools with the version rule, and a `propose` tool with a review view.
4. The ACP client, with a minimal chat interface.
5. The optional prompt hook.
6. After upstream ships plugins, the move onto the plugin API.

## Future work: Send a selection and prompt to an agent

A Mitos command could capture the exact selected text, including unsaved edits, and prompt for a message to send with it. Both delivery options can use the same snapshot of the workspace, document identity, selection range, and buffer version. The user chooses the destination before sending.

- **User-configured command.** Mitos could launch an executable with arguments from user-level configuration and pass the prompt and selection snapshot as versioned JSON on stdin. The command handles routing to an existing agent, for example by pasting a draft into a paired tmux pane or Ghostty terminal. For terminal delivery, the user reviews and submits the draft. Mitos reports the outcome the command confirms, such as a paste into the pane. The agent can use the existing MCP integration to fetch fresh editor context.
- **ACP session.** Mitos could send the prompt and selection as an ACP user turn to a session exposed on its ACP connection. A compatible agent or external adapter handles agent-specific behavior, and Mitos can show structured session updates and permission requests. Loading a saved conversation, when supported, is separate from joining a live session in an independently launched terminal.

## Open questions

1. Can `claude-agent-acp` load a session from the Claude Code terminal interface?
2. Which small interfaces can go upstream?
3. Which transport and which lock does Windows use?
4. Which agents support MCP resources and notifications that the tool list changed?

## References

- Greed agents documentation: https://github.com/johnae/greed/blob/main/runtime/docs/agents.md
- Agent Client Protocol: https://agentclientprotocol.com
- Zed Inline Assistant: https://zed.dev/docs/ai/inline-assistant
