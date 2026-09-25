# github-mcp-agent

MCP Server built with Node.js + TypeScript that exposes a catalog of *tools* to automate GitHub operations (creating repositories, opening issues, listing resources, and making real commits), designed to be consumed by an AI agent (LLM) from **Antigravity** through the MCP (Model Context Protocol).

The server has no graphical interface or database of its own: the "frontend" is the LLM that interprets requests in natural language and decides which tool to invoke, while GitHub itself acts as the "storage".

## AI Usage During Development

Complete and auditable documentation of AI usage throughout this project (log, decisions, and defense preparation): [Google Drive folder](https://drive.google.com/drive/folders/1hySGV83Z9Fpu3wPlvTVSdCaUnRcKvEVt?usp=sharing).

---

## Table of Contents

* [AI Usage During Development](#ai-usage-during-development)
* [What Does It Do and Why Is It Useful?](#what-does-it-do-and-why-is-it-useful)
* [Architecture](#architecture)
* [Requirements](#requirements)
* [Installation](#installation)
* [Getting a GitHub Personal Access Token](#getting-a-github-personal-access-token)
* [Configuring Environment Variables](#configuring-environment-variables)
* [Configuring the Server in Antigravity](#configuring-the-server-in-antigravity)
* [Available Tools](#available-tools)
* [Usage Examples](#usage-examples)
* [Tests](#tests)
* [Troubleshooting](#troubleshooting)
* [License](#license)

---

## What Does It Do and Why Is It Useful?

It gives an AI agent the ability to operate a GitHub account using natural language, instead of requiring the user to run `git`/`gh` commands or manually navigate the web interface. Typical use cases include:

* **Creating a new repository** to start a project without leaving the agent's chat.
* **Reporting or triaging issues** quickly by describing the problem in natural language.
* **Checking existing repositories** or open issues in a specific repository without opening a browser.
* **Uploading or updating a file** (such as a `README`, `CHANGELOG`, or script) through a real commit by describing the change in plain text.

Each action returns **verifiable evidence** (a URL, issue number, or commit SHA) so the user can confirm on GitHub that the operation actually took place — the agent performs the action and leaves an auditable trail rather than simply making suggestions.

---

## Architecture

```text
┌──────────────┐      ┌────────────────┐      ┌───────────────────┐      ┌─────────────────┐
│  Antigravity │─────▶│  LLM            │─────▶│  MCP Server        │─────▶│  GitHub API      │
│  (Host)      │◀─────│  (Client)       │◀─────│  (this project)    │◀─────│  (via Octokit)   │
└──────────────┘      └────────────────┘      └───────────────────┘      └─────────────────┘
   manages the          interprets the          exposes the tools,          executes the
   session and          request and decides    validates inputs with       actual operation
   connects the         which tool to use      Zod, executes the            and returns the
   components           and with which params  operation, and               result
                                               translates errors
```

Communication between Antigravity and this server takes place exclusively through **stdio** (standard input/output of the process), not HTTP: Antigravity launches the server as a child process. Therefore, `stdout` is entirely reserved for the MCP JSON-RPC 2.0 protocol — all diagnostic logging goes through `stderr`.

---

## Requirements

* **Node.js 18+** (developed and tested with Node `24.16.0`; see `.nvmrc`). If using `nvm`, `nvm use` automatically selects the required version.
* A **GitHub** account with the ability to generate a Personal Access Token (classic).
* **Antigravity** installed if you want to connect the server to a real LLM (not required for development or running the tests).

---

## Installation

```bash
git clone https://github.com/ciro-castellaro/ProyectoM5-Ciro_Castellaro.git
cd ProyectoM5-Ciro_Castellaro
npm install
npm run build
```

Available scripts (`package.json`):

| Script          | Description                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------- |
| `npm run build` | Compiles TypeScript (`src/`) to JavaScript (`dist/`)                                              |
| `npm run dev`   | Runs the server directly from TypeScript with automatic reload (`tsx watch`)                      |
| `npm start`     | Runs the compiled version (`node dist/server.js`) — the version used by Antigravity in production |
| `npm run test`  | Runs the test suite with Vitest                                                                   |

---

## Getting a GitHub Personal Access Token

1. In GitHub, go to **Settings → Developer settings → Personal access tokens → Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Enter a descriptive name, for example `mcp-github-agent`.
4. Required scopes:

   * **`repo`** — required. Used by all 5 tools to read/write repositories, issues, and commits.
   * **`user`** — recommended for basic information about the authenticated user.
   * **`admin:org`** — **do not enable** unless you need to operate on organizations; none of the tools in this project use it.
5. Generate the token and **copy it immediately** — GitHub will not display it again.

> The token must never be pasted into the code, a commit, or the Antigravity configuration as plain text. See the following two sections.

---

## Configuring Environment Variables

Copy the template and add your token:

```bash
cp .env.example .env
```

`.env` (never committed; it is included in `.gitignore`):

```text
GITHUB_TOKEN=ghp_your_token_here
LOG_LEVEL=info
```

* `GITHUB_TOKEN`: the Personal Access Token generated in the previous step.
* `LOG_LEVEL`: `debug` | `info` | `warn` | `error` (optional, default `info`). All logs are sent through `stderr`, never `stdout`.

The server automatically loads `.env` using Node's native environment file loading (`process.loadEnvFile()`, with no `dotenv` dependency). If `.env` does not exist, it assumes that the variables are already defined in the system environment (for example, when Antigravity injects them directly).

---

## Configuring the Server in Antigravity

In Antigravity, click `...` in the agent panel → **MCP Servers → Manage MCP Servers → View raw config** to open the actual configuration file used by your installation (the exact name and location may vary depending on the version and operating system — do not assume it; open it and verify it there). On Windows, it is usually located at `%userprofile%\.gemini\config\mcp_config.json`.

Add the server entry:

```json
{
  "mcpServers": {
    "github-mcp-agent": {
      "command": "node",
      "args": ["/absolute/path/to/where/you/cloned/the/project/dist/server.js"]
    }
  }
}
```

### Key Points

* **`args` is an absolute path that must be adapted to each machine** — `mcp_config.json` is not part of this repository; it lives in each user's local Antigravity installation. If you (or someone else) cloned the project to, for example, `C:\Users\you\projects\github-mcp-agent`, `args` must point there, not to another person's path. This is the same for any MCP server (Claude Desktop, Cursor, etc.): the host needs the actual path to the executable on that disk.
* **Use the compiled build** (`node dist/server.js`), not `npx tsx src/server.ts` — the latter is only intended for active development.
* **No `env` block is required in this configuration.** The server loads its own `GITHUB_TOKEN` directly from the project's `.env` file (`process.loadEnvFile()`, resolved based on the location of the compiled file itself rather than the directory from which Antigravity launches the process). Having a complete `.env` file in the project folder is enough, as explained above.
* **Passing the token through `${GITHUB_TOKEN}` interpolation in an `env` block was explicitly tested and was not reliable** when relying on an operating-system environment variable: in real-world testing, Antigravity did not consistently pass changes to that variable to the server process, even after closing/reopening the application or performing a full Windows restart. Therefore, the final configuration avoids relying on the system environment entirely and uses only the project's `.env`, which was tested to be reliable regardless of how the process is launched.
* Completely restart Antigravity (close and reopen it) after saving `mcp_config.json` or modifying `.env`, so the server starts from scratch and picks up the changes.

---

## Available Tools

All tools return text containing verifiable evidence (URL, number, SHA) upon success, and a natural-language message (without stack traces or sensitive data) in case of an error.

### `create_repository`

Creates a new repository in the authenticated account, including an initial README so it is ready to use `create_commit` immediately.

| Parameter     | Type    | Required | Notes                                                           |
| ------------- | ------- | -------- | --------------------------------------------------------------- |
| `name`        | string  | Yes      | 3–100 characters, letters/numbers/dots/hyphens/underscores only |
| `description` | string  | No       | Up to 350 characters                                            |
| `private`     | boolean | No       | Default `false`                                                 |

**Example prompt:** *"Create a public repository called `demo-api` with the description 'Test API for the course'."*

### `create_issue`

Opens an issue in an existing repository.

| Parameter   | Type     | Required | Notes                                                       |
| ----------- | -------- | -------- | ----------------------------------------------------------- |
| `owner`     | string   | Yes      | Repository owner                                            |
| `repo`      | string   | Yes      | Repository name                                             |
| `title`     | string   | Yes      | Up to 256 characters                                        |
| `body`      | string   | No       | Issue description                                           |
| `labels`    | string[] | No       | No duplicates, up to 100                                    |
| `assignees` | string[] | No       | Up to 10                                                    |
| `milestone` | number   | No       | Positive integer of an existing milestone in the repository |

**Example prompt:** *"Open an issue in ciro-castellaro/demo-api titled 'Add authentication', explaining that OAuth login is missing, and assign it to milestone 3."*

### `list_repositories`

Lists the repositories of the authenticated user.

| Parameter   | Type   | Required | Notes                                                             |
| ----------- | ------ | -------- | ----------------------------------------------------------------- |
| `page`      | number | No       | Default `1`                                                       |
| `perPage`   | number | No       | Default `30`, max. `100`                                          |
| `sort`      | enum   | No       | `created` | `updated` | `pushed` | `full_name`, default `updated` |
| `direction` | enum   | No       | `asc` | `desc`, default `desc`                                    |
| `type`      | enum   | No       | `all` | `owner` | `member`, default `owner`                       |

**Example prompt:** *"Show me my 5 most recent repositories sorted by update date."*

### `create_commit`

Adds or modifies a file on an existing branch using the Git internals workflow (blob → tree → commit → ref).

| Parameter | Type   | Required | Notes                           |
| --------- | ------ | -------- | ------------------------------- |
| `owner`   | string | Yes      |                                 |
| `repo`    | string | Yes      |                                 |
| `branch`  | string | Yes      | Must already exist              |
| `path`    | string | Yes      | File path inside the repository |
| `content` | string | Yes      | Full file content               |
| `message` | string | Yes      | Commit message                  |

**Example prompt:** *"Add a CHANGELOG.md file to the ciro-castellaro/demo-api repository on the main branch with the content '# Changelog\n\n## v1.0.0 - Initial release' and the commit message 'Add initial changelog'."*

### `list_issues`

Lists issues (never pull requests, which are automatically excluded) from a specific repository.

| Parameter   | Type     | Required | Notes                                                 |
| ----------- | -------- | -------- | ----------------------------------------------------- |
| `owner`     | string   | Yes      |                                                       |
| `repo`      | string   | Yes      |                                                       |
| `state`     | enum     | No       | `open` | `closed` | `all`, default `open`             |
| `labels`    | string[] | No       |                                                       |
| `sort`      | enum     | No       | `created` | `updated` | `comments`, default `created` |
| `direction` | enum     | No       | `asc` | `desc`, default `desc`                        |
| `page`      | number   | No       | Default `1`                                           |
| `perPage`   | number   | No       | Default `30`, max. `100`                              |

**Example prompt:** *"Show me the open issues in the ciro-castellaro/demo-api repository, sorted by creation date."*

### `ping`

Diagnostic tool with no parameters that responds with `"pong"`. It is used to confirm that the server is connected before using the actual tools — it does not perform any operation against GitHub.

---

## Usage Examples

Real evidence generated during development (verified with MCP Inspector against a real GitHub account):

```text
> create_repository { name: "mcp-agent-test", description: "..." }
Repository created: ciro-castellaro/mcp-agent-test (https://github.com/ciro-castellaro/mcp-agent-test). Visibility: public.

> create_issue { owner: "ciro-castellaro", repo: "mcp-agent-test", title: "Test issue from MCP agent", body: "..." }
Issue #1 created: "Test issue from MCP agent" (https://github.com/ciro-castellaro/mcp-agent-test/issues/1).

> create_commit { owner: "ciro-castellaro", repo: "mcp-agent-test", branch: "main", path: "test-commit.md", content: "...", message: "Add test-commit.md via create_commit" }
Commit created: bb4bdf0bbb9c0bdf19ff58bed23483d1527e18ba — "Add test-commit.md via create_commit" (https://github.com/ciro-castellaro/mcp-agent-test/commit/bb4bdf0bbb9c0bdf19ff58bed23483d1527e18ba).

> list_issues { owner: "ciro-castellaro", repo: "mcp-agent-test" }
- #1 [open] Test issue from MCP agent — https://github.com/ciro-castellaro/mcp-agent-test/issues/1

> list_repositories { perPage: 3 }
- ciro-castellaro/ProyectoM5-Ciro_Castellaro (public) — https://github.com/ciro-castellaro/ProyectoM5-Ciro_Castellaro
- ciro-castellaro/mcp-agent-test (public) — https://github.com/ciro-castellaro/mcp-agent-test
- ciro-castellaro/ProyectoM2-Ciro_Castellaro (public) — https://github.com/ciro-castellaro/ProyectoM2-Ciro_Castellaro
```

An example of invalid input rejected **without** calling the GitHub API:

```text
> create_repository { name: "ab" }
isError: true
"The repository name must contain at least 3 characters"
```

And an example of an actual GitHub error translated into natural language:

```text
> list_issues { owner: "ciro-castellaro", repo: "this-repo-does-not-exist" }
isError: true
"The requested resource was not found on GitHub. Verify the owner and repository name."
```

Any of these can be tested directly with **MCP Inspector**, without requiring Antigravity:

```bash
npx @modelcontextprotocol/inspector --cli node dist/server.js --method tools/list
npx @modelcontextprotocol/inspector --cli node dist/server.js --method tools/call --tool-name list_repositories
```

---

## Tests

```bash
npm run test
```

47 Vitest tests, deterministic, **with no real calls to the GitHub API** (the Octokit client is mocked by injecting it into the `GitHubClient` constructor):

* `tests/tools.test.ts` (24 tests) — validation of the 5 Zod schemas: valid and invalid inputs, plus hardening cases (path traversal, size limits, formats, duplicates).
* `tests/github.test.ts` (9 tests) — the 5 `GitHubClient` operations with Octokit mocked, including error cases (non-existent repository, invalid credentials) and retry behavior for recoverable errors.
* `tests/errors.test.ts` (14 tests) — classification and translation of errors (400/401/403/404/422/429/5xx) into natural-language messages, and sanitization of external messages.

---

## Troubleshooting

| Error / Symptom                                                                                                                                             | Likely Cause                                                                                                                                                                                                                                         | What to Do                                                                                                                                                                                                                                                                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Server does not start, log `GITHUB_TOKEN is not configured`                                                                                                 | `.env` is missing from the project folder, or the `GITHUB_TOKEN=` line is empty                                                                                                                                                                      | Verify that `.env` exists next to `package.json` and contains `GITHUB_TOKEN=...` with a valid value                                                                                                                                                                                                                 |
| `The GitHub token is invalid or has expired` (401)                                                                                                          | Token expired, revoked, or incorrectly copied                                                                                                                                                                                                        | Generate a new Personal Access Token and update `.env`                                                                                                                                                                                                                                                              |
| You rotated the token and updated `.env`, but Antigravity still returns 401 with the old token (even after closing/reopening the app or restarting Windows) | Antigravity did not actually relaunch the server process, or a system environment variable named `GITHUB_TOKEN` from a previous configuration is overriding the `.env` value (`.env` never overwrites a variable already defined in the environment) | Check whether a residual system environment variable named `GITHUB_TOKEN` exists (`[Environment]::GetEnvironmentVariable("GITHUB_TOKEN","User")` in PowerShell) and remove it if present; confirm the `.env` by running `node dist/server.js` directly in a new terminal (without Antigravity) before testing again |
| `The token does not have sufficient permissions` (403, not rate limit)                                                                                      | The token is missing the `repo` scope                                                                                                                                                                                                                | Regenerate the token with the `repo` scope enabled                                                                                                                                                                                                                                                                  |
| `The GitHub API request limit has been reached` (403, rate limit)                                                                                           | The API request limit has been exhausted                                                                                                                                                                                                             | Wait until the time indicated by GitHub (`x-ratelimit-reset`); the server automatically retries up to 3 times                                                                                                                                                                                                       |
| `The requested resource was not found on GitHub` (404)                                                                                                      | `owner`/`repo` is misspelled, or the repository does not exist/is not accessible with this token                                                                                                                                                     | Verify the exact owner and repository name                                                                                                                                                                                                                                                                          |
| `create_commit` fails even though the repository exists                                                                                                     | The specified branch does not exist yet                                                                                                                                                                                                              | Use an existing branch (repositories created with `create_repository` already have `main` ready, thanks to the initial README)                                                                                                                                                                                      |
| Antigravity does not list the tools                                                                                                                         | The server is not registered correctly, or Antigravity needs to be restarted                                                                                                                                                                         | Check the configuration in "View raw config", verify that `command`/`args` point to `dist/server.js`, and reload Antigravity                                                                                                                                                                                        |
| Antigravity disconnects or reports protocol errors                                                                                                          | Something is writing to `stdout` instead of `stderr`                                                                                                                                                                                                 | Make sure no `console.log` has been added (all logging must go through `utils/logging.ts`, which uses `console.error`)                                                                                                                                                                                              |
| `npm run build` fails with `Cannot find name 'process'`                                                                                                     | Node types are missing from `tsconfig.json`                                                                                                                                                                                                          | This is already configured in the project (`"types": ["node"]`); if the error appears again, confirm that `@types/node` is installed                                                                                                                                                                                |

---

## License

MIT — see [`LICENSE`](./LICENSE).
