# Spira MCP server: installation and usage

The [Inflectra Spira MCP server](https://github.com/Inflectra/mcp-server-spira)
connects an MCP client to SpiraTest, SpiraTeam, or SpiraPlan. The safest setup is
to build the server from its source and let the MCP client launch it over
standard input/output (stdio).

> This project is a community/sample integration. Test it against a non-production
> project first and grant the Spira account only the permissions it needs.

## 1. Information and software you need

There are two separate connections: the MCP client launches the MCP server, and
the MCP server authenticates to Spira. Gather the following before configuring
either connection.

### Information needed to access Spira

| Required information | Where to get it | Safe example |
| --- | --- | --- |
| **Spira base URL** | Copy the tenant root from the browser address bar or ask the Spira administrator. Do not append a REST endpoint. | `https://company.spiraservice.net` |
| **Spira user name** | Use the login name of a dedicated integration user. It must be active and a member of the relevant projects. | `mcp.integration` |
| **API key for that user** | The integration user generates or retrieves it from their Spira user profile. Use an API key, not the account password. | Keep the real value secret |
| **Project ID** | Open the project in Spira and take its numeric ID from the project URL, or ask the administrator. | `42` |

The user account's project membership and role determine which artifacts the MCP
server can read or change. For read-only use, give the account a read-only role.
For create or update operations, the role must grant those specific permissions.
If access is limited by a VPN, proxy, firewall, IP allowlist, or private DNS, the
computer running the MCP server must also meet those network requirements.

Do **not** send an API key in chat, an issue, email, or a screenshot. The person
configuring the MCP client should place it directly in the client's secret store
or local environment. If another person is helping with setup, they need the base
URL, user name, and project ID, but they should not need to see the API key.

### Information needed by the MCP client

To launch the local server, the MCP client also needs:

- the absolute path to the installed runtime executable;
- the absolute path to the built Spira MCP server entry point;
- the setting names expected by the checked-out server version; and
- permission to run local stdio MCP servers and use the server's tools.

The machine needs Git and the runtime named in the upstream repository. Check
`package.json`, `pyproject.toml`, or a solution/project file after cloning to
identify that runtime and its supported version.

### Access checklist

Before troubleshooting the MCP client, confirm all of these are true:

- [ ] The Spira base URL opens from the machine that will run the MCP server.
- [ ] The integration user can sign in and is not locked or disabled.
- [ ] The API key belongs to that same user and has not expired or been revoked.
- [ ] The user is a member of the intended project with the required role.
- [ ] The numeric project ID refers to that project.
- [ ] The runtime and built MCP server paths are absolute and exist locally.

Only the API key is a secret in this checklist, but the URL, user name, and
project membership may still be sensitive organizational information.

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

## 5. Use the server

The server is not an interactive command-line application. Your MCP client starts
it in the background, discovers the tools it publishes, and calls those tools in
response to ordinary chat requests. Once it is registered, use it as follows.

### Start a session and confirm the connection

1. Fully restart the MCP client after changing its configuration.
2. Open the client's MCP, integrations, or tools panel.
3. Confirm that `spira` is connected and expand it to see the tools advertised by
   the installed server version.
4. Enable those tools for the current conversation if the client requires tools
   to be approved per chat.
5. Ask a harmless discovery question: “Using Spira, list the projects I can
   access. Do not make any changes.”

If the client answers without calling a Spira tool, explicitly say **using the
Spira MCP tools**. Check the client's tool-call details to verify that the answer
came from Spira rather than from the model's general knowledge.

### Identify artifacts unambiguously

Include the project ID and, when known, the artifact type and artifact ID in each
request. Names and numeric IDs are safer than vague references such as “that
ticket.” A useful request contains:

- the Spira project, for example `project 42`;
- the artifact type, such as requirement, incident, test case, task, or release;
- the desired filter or artifact ID;
- whether the operation must be read-only; and
- the fields to return or change.

The available artifact types and operations depend on the installed server
version, the Spira edition, and the integration user's project role. The tool list
shown by the MCP client is the source of truth for that session.

### Read and summarize Spira data

Start with small, read-only queries so that identities, permissions, and project
scope can be checked before any updates. For example:

- “Using Spira, list the releases in project 42. Return ID, name, status, start
  date, and end date. Do not modify anything.”
- “Using the Spira MCP tools, show open incidents assigned to me in project 42.
  Return at most 20 items with ID, name, priority, status, and owner.”
- “Read requirement RQ:123 in project 42 and summarize its description and
  acceptance criteria. Include the artifact ID in the answer.”
- “Find test cases associated with requirement RQ:123. Report only data returned
  by Spira and do not create or update artifacts.”

Ask for a result limit when listing artifacts. For auditability, ask the client to
include Spira artifact IDs in its response.

### Create or update an artifact safely

Use a two-step preview-and-apply workflow for writes:

1. Ask the client to retrieve the target artifact and show the exact proposed
   values without making a change.
2. Check the project ID, artifact ID, type, status, owner, dates, and text.
3. Send a second message explicitly approving that proposal.
4. Ask the client to read the artifact back from Spira and show its final ID and
   values.

Example preview:

> Using the Spira MCP tools, prepare an incident for project 42 titled “Login
> timeout.” Set priority to High and put the reproduction steps below in the
> description. Show every field you intend to send, but do not create it yet.

After reviewing the preview:

> Create the incident exactly as previewed in project 42. Do not change any other
> artifact. Then read it back and return its incident ID, name, status, priority,
> and URL if the tool provides one.

The same pattern works for an update:

> Read incident IN:456 in project 42. Propose changing only its priority to
> Critical; do not update it yet.

Never approve a write if the client selected a different project or artifact than
the one requested. Review every write operation because the server acts with the
permissions of its configured Spira account.

### Use tool approval controls

If the MCP client supports per-tool approval, allow read tools automatically only
when that fits the organization's policy, and require confirmation for create,
update, delete, or transition operations. Avoid broad instructions such as “fix
all incidents.” Prefer a bounded selection, preview it, and apply changes in small
batches.

### End or switch a session

Starting a new chat does not change the credentials or default project supplied
to the server process. To use another Spira tenant or identity, change the MCP
configuration (or create a separately named server entry), restart the client,
and verify the identity and accessible projects again. Do not ask the model to
override credentials in chat.

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
