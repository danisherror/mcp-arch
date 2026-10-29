# MCP Architecture

Security-first architecture references for MCP servers that let an AI agent
act on real systems while treating the agent as untrusted. The agent never
gets raw commands or queries: it picks typed operations from a fixed
registry, every write needs an out-of-band human approval bound to an exact
operation hash, and the backend re-checks everything independently.

A working implementation of the dev-VM design lives at
[danisherror/mcp-devvm](https://github.com/danisherror/mcp-devvm).

## The docs

**Master references** (`[0]`), one per target system:

| doc | what it covers |
|---|---|
| [`MCP_SERVER.md`](./docs/%5B0%5D%20MCP_SERVER.md) | Secure MCP CRUD server over a database |
| [`MCP_SERVER_PYTHON_API.md`](./docs/%5B0%5D%20MCP_SERVER_PYTHON_API.md) | Same design in Python, fronting an upstream API instead of a database |
| [`MCP_SERVER_PYTHON_SSH.md`](./docs/%5B0%5D%20MCP_SERVER_PYTHON_SSH.md) | Command server in Python reaching a dev VM over SSH |
| [`MCP_SERVER_DEVVM.md`](./docs/%5B0%5D%20MCP_SERVER_DEVVM.md) | Dev-VM architecture: typed operations, sandboxing, approvals (implemented in mcp-devvm) |
| [`WRITING_OPERATIONS.md`](./docs/%5B1%5D%20WRITING_OPERATIONS.md) | How to turn shell commands into safe, typed MCP operations |

**Source material** the references were built from: the design prompts
(`[1]`, `[2]`), answers from different models (`[3]` Claude, `[4]` Gemini,
`[5]` Grok, `[6]` ChatGPT) and architecture diagrams (`[7]`, `[8]`).

## License

MIT — see [LICENSE](./LICENSE).
