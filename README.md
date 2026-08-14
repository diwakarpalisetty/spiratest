# Spira MCP server: installation and usage

The [Inflectra Spira MCP server](https://github.com/Inflectra/mcp-server-spira)
connects an MCP client to SpiraTest, SpiraTeam, or SpiraPlan. The safest setup is
to build the server from its source and let the MCP client launch it over
standard input/output (stdio).

> This project is a community/sample integration. Test it against a non-production
> project first and grant the Spira account only the permissions it needs.

## 1. Prerequisites

- Git.
- The runtime named in the upstream repository (check `package.json`,
  `pyproject.toml`, or a solution/project file after cloning).
- A Spira base URL, such as `https://company.spiraservice.net`.
- A dedicated Spira user name and that user's API key.
- The ID of the Spira project that the client should access.

Create or retrieve the API key from the user's Spira profile. Do not use the
account password and do not commit the key to Git.

## 2. Clone and build

```bash
git clone https://github.com/Inflectra/mcp-server-spira.git
cd mcp-server-spira
```

Read the checked-out version's instructions before continuing:

```bash
sed -n '1,240p' README.md
find . -maxdepth 2 -type f \
  \( -name package.json -o -name pyproject.toml -o -name '*.csproj' -o -name '*.sln' \) \
  -print
```

Then use the build instructions in that README. Keeping the clone in a stable
absolute location is important: the MCP client configuration will refer to its
entry point by absolute path.

## 3. Configure credentials

Use the **exact setting names documented by the checked-out upstream version**.
The values will represent these four concepts:

| Value | Example | Notes |
| --- | --- | --- |
| Spira URL | `https://company.spiraservice.net` | Use the tenant root, without a REST resource suffix. |
| User name | `mcp.integration` | Prefer a dedicated least-privilege account. |
| API key | `{...}` | Treat as a secret. |
| Project ID | `42` | The numeric ID visible in Spira project URLs. |

If the upstream project supplies `.env.example`, copy it to `.env`, replace its
placeholders, and ensure `.env` remains ignored:

```bash
cp .env.example .env
git check-ignore .env
```

If `git check-ignore` prints nothing, add `.env` to the clone's `.gitignore`
before putting a secret in it.

## 4. Register it with an MCP client

Each client uses slightly different configuration, but a local stdio server has
the same shape:

```json
{
  "mcpServers": {
    "spira": {
      "command": "<runtime executable>",
      "args": ["<absolute path to the built server entry point>"],
      "env": {
        "<UPSTREAM_SPIRA_URL_SETTING>": "https://company.spiraservice.net",
        "<UPSTREAM_USER_SETTING>": "mcp.integration",
        "<UPSTREAM_API_KEY_SETTING>": "your-api-key",
        "<UPSTREAM_PROJECT_ID_SETTING>": "42"
      }
    }
  }
}
```

Replace **both** the angle-bracketed setting names and their values. Do not paste
this template unchanged. Prefer the client's secret store or inherited process
environment over plaintext JSON when the client supports it.

Common configuration locations include:

- **Claude Desktop:** Settings → Developer → Edit Config, then add the `spira`
  member under the existing `mcpServers` object.
- **VS Code:** add the server through the MCP server command/UI and translate
  `command`, `args`, and environment variables into the generated configuration.
- **Other MCP clients:** choose a local **stdio** server and supply the same
  executable, arguments, and environment.

Restart the client after saving the configuration. A JSON file can contain only
one top-level `mcpServers` object, so merge this entry with existing servers
rather than adding a second object.

## 5. Verify and use it

Open the client's MCP/server panel and check that `spira` is connected and its
tools are listed. Begin with read-only requests, for example:

- “List the projects I can access in Spira.”
- “Show the open incidents assigned to me in project 42.”
- “Summarize requirements for release 7 without changing anything.”

Before a write operation, name the project and ask the model to show the proposed
change first:

> In Spira project 42, show me the fields you would use for a new incident titled
> “Login timeout”; do not create it yet.

Review every write request. An MCP server acts with the permissions of its Spira
account.

## Troubleshooting

### Server is disconnected or no tools appear

Run the configured command manually in a terminal. Use absolute paths, verify the
runtime is installed in the desktop application's environment, rebuild after an
upstream update, and inspect the client's MCP logs. A stdio server may appear to
wait silently when run manually; that can be normal because it is waiting for an
MCP client request.

### Authentication fails

Confirm the base URL opens in a browser, the integration user is active, the API
key belongs to that same user, and no extra REST endpoint was appended to the
base URL. Rotate any key accidentally exposed in terminal output, config files,
screenshots, or chat.

### A project cannot be found

Check that the project ID is numeric and that the integration user is a member of
that project with an appropriate role. SpiraTest, SpiraTeam, and SpiraPlan can
also expose different artifact types according to the installed edition and the
user's permissions.

### The server starts in a terminal but not in the client

Desktop applications often have a smaller `PATH` than an interactive shell.
Use absolute paths for both the runtime executable and server entry point, then
fully quit and reopen the client.
