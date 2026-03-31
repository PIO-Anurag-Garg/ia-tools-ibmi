# iA Tools for IBM i — Impact Analysis MCP Tool Definitions

SQL-based [MCP](https://modelcontextprotocol.io/) tool definitions for **Impact Analysis (iA)** on IBM i. These tools query pre-parsed IBM i source metadata in the iA repository (built by [programmers.io](https://www.programmers.io/)) to provide cross-reference lookups, call hierarchy analysis, field impact analysis, dead code detection, and more.

Use them with **VS Code GitHub Copilot**, **Claude Code**, or any MCP-compatible AI agent to ask natural-language questions about your IBM i codebase.

## What's in this repo

| File | Description |
|------|-------------|
| `impact-analysis.yaml` | 24 MCP tool definitions for iA queries |
| `.vscode/mcp.json` | VS Code MCP server config (auto-detected on open) |
| `.env.example` | DB2i connection template |
| `LICENSE` | Apache-2.0 |

## Tools (24)

| # | Tool | Description |
|---|------|-------------|
| 1 | `ia_where_used` | Find all objects referencing a given object |
| 2 | `ia_call_hierarchy` | Program call tree (CALLERS/CALLEES/BOTH) |
| 3 | `ia_field_impact` | Blast radius of changing a field in a file |
| 4 | `ia_program_variables` | All variables declared in a program |
| 5 | `ia_data_structures` | DS definitions with subfields and positions |
| 6 | `ia_call_parameters` | Parameters passed to external calls |
| 7 | `ia_subroutines` | BEGSR/ENDSR details with usage counts |
| 8 | `ia_file_overrides` | OVRDBF statements (hidden file routing) |
| 9 | `ia_file_fields` | File field metadata (richer than DSPFFD) |
| 10 | `ia_object_list` | Repository inventory by type |
| 11 | `ia_program_info` | Program/module metadata |
| 12 | `ia_rpg_source_tokens` | RPG token-level source analysis |
| 13 | `ia_cl_source_tokens` | CL token-level source analysis |
| 14 | `ia_dashboard` | Repository health summary |
| 15 | `ia_repo_config` | Repository configuration settings |
| 16 | `ia_exception_log` | Parser error log |
| 17 | `ia_dds_to_ddl_status` | DDS-to-DDL conversion tracking |
| 18 | `ia_reference_count` | Quick reference count by object type |
| 19 | `ia_unused_objects` | Dead code candidates (unreferenced objects) |
| 20 | `ia_circular_deps` | Circular dependency detection |
| 21 | `ia_where_used_detail` | Enhanced where-used with source existence check |
| 22 | `ia_override_chain` | Chained OVRDBF dependency analysis |
| 23 | `ia_object_lifecycle` | Creation, change, and last-used dates |
| 24 | `ia_code_complexity` | Complexity metrics (IF/DO/SQL/GOTO counts) |

---

## Quick Start — VS Code + GitHub Copilot

This is the recommended way to use and develop iA tools. VS Code will automatically start the MCP server when you open this folder.

### Prerequisites

| Requirement | Details |
|-------------|---------|
| **VS Code** | With [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) extension installed (Copilot Chat required) |
| **Node.js 18+** | Download from [nodejs.org](https://nodejs.org/) — required to run the MCP server |
| **IBM i access** | Db2 for i accessible via [Mapepire](https://mapepire.github.io/) on port 8076 |
| **iA repository** | Your IBM i source must be parsed by [programmers.io iA](https://www.programmers.io/) into a library (e.g., `SDK01`) |

### Step 1: Fork and clone

```bash
# Fork this repo on GitHub (click the "Fork" button), then:
git clone https://github.com/<your-username>/ia-tools-ibmi.git
cd ia-tools-ibmi
```

### Step 2: Configure your IBM i connection

```bash
cp .env.example .env
```

Edit `.env` with your IBM i credentials:

```
DB2i_HOST=your-ibmi-hostname
DB2i_USER=your-user-profile
DB2i_PASS=your-password
DB2i_PORT=8076
IA_LIBRARY=SDK01
```

| Variable | Description |
|----------|-------------|
| `DB2i_HOST` | IBM i hostname or IP address |
| `DB2i_USER` | IBM i user profile with `*READ` authority to iA tables |
| `DB2i_PASS` | Password for the user profile |
| `DB2i_PORT` | Mapepire port (default: `8076`) |
| `IA_LIBRARY` | Library where iA repository tables are stored (default: `SDK01`) |

> **Security**: `.env` is in `.gitignore` — your credentials are never committed.

### Step 3: Open in VS Code

```bash
code .
```

When VS Code opens the folder, it detects `.vscode/mcp.json` and automatically:
1. Runs `npx -y @ibm/ibmi-mcp-server@latest` (downloads the MCP server if needed)
2. Loads `impact-analysis.yaml` (all 24 iA tools)
3. Connects to your IBM i using the `.env` credentials

> **First time?** The MCP server download may take a minute. You'll see a notification in VS Code when it's ready.

### Step 4: Use iA tools in Copilot Chat

1. **Open Copilot Chat**: Press `Ctrl+Alt+I` (Windows/Linux) or `Cmd+Alt+I` (Mac)
2. **Switch to Agent mode**: Click the mode dropdown at the top of the chat panel and select **"Agent"**
3. **Verify tools are loaded**: Click the **tools icon** (wrench/hammer) at the top-left of the chat input — you should see the 24 `ia-*` tools listed under "ibmi-ia-tools"
4. **Ask a question** — the agent will automatically pick the right iA tool:

```
Find all programs that reference CUSTFILE
```
```
Show the call hierarchy for ORDERPGM
```
```
What fields are in CUSTMASTER and which programs use them?
```
```
Are there any unused objects in the repository?
```

> **Note**: Copilot will ask for your confirmation before running each MCP tool. Click **"Allow"** (or "Always allow" for this session).

---

## Alternative Setup

### Claude Code

Create a `.mcp.json` file in your project root:

```json
{
  "mcpServers": {
    "ibmi-ia-tools": {
      "command": "npx",
      "args": ["-y", "@ibm/ibmi-mcp-server@latest", "--tools", "/full/path/to/impact-analysis.yaml"],
      "cwd": "."
    }
  }
}
```

Set `DB2i_HOST`, `DB2i_USER`, `DB2i_PASS`, `DB2i_PORT`, and `IA_LIBRARY` as environment variables (or in `.env` in the same directory — the MCP server loads it automatically).

### Command Line (manual testing)

Run the MCP server directly to test without an AI agent:

```bash
# Install the MCP server globally
npm install -g @ibm/ibmi-mcp-server

# Set environment variables
export DB2i_HOST=your-ibmi-host
export DB2i_USER=your-user
export DB2i_PASS=your-password
export DB2i_PORT=8076
export IA_LIBRARY=SDK01

# Start in HTTP mode for manual testing
npx @ibm/ibmi-mcp-server --transport http --tools ./impact-analysis.yaml
# Server starts at http://localhost:3000
```

---

## Developing Tools

### YAML tool format

Each tool follows this structure:

```yaml
ia_tool_name:
  name: ia-tool-name
  source: ibmi
  description: >
    What this tool does, when to use it, and which iA table it queries.
  parameters:
    - name: param_name
      type: string          # string or integer
      description: "What this parameter controls"
      required: true        # or false
      default: "*ALL"       # optional default value
    - name: limit
      type: integer
      description: "Maximum rows to return"
      required: false
      default: 200
  statement: |
    SELECT columns
    FROM ${IA_LIBRARY}.TABLE_NAME
    WHERE COLUMN = :param_name
    FETCH FIRST :limit ROWS ONLY
```

**Key conventions:**

| Syntax | Purpose |
|--------|---------|
| `${IA_LIBRARY}` | Replaced at runtime with the iA repository library name |
| `:param_name` | Parameter placeholder (bound at execution time, prevents SQL injection) |
| `source: ibmi` | Uses the Db2i connection from the `sources` block at the top of the YAML |
| `FETCH FIRST :limit ROWS ONLY` | Always include this to bound result sets |

**Naming rules:**
- Tool key: `ia_snake_case` (e.g., `ia_where_used`)
- MCP name: `ia-kebab-case` (e.g., `ia-where-used`)
- Parameters: `snake_case`

### Testing your changes

1. **Edit** `impact-analysis.yaml` — add or modify a tool definition
2. **Restart the MCP server** in VS Code:
   - Open Command Palette: `Ctrl+Shift+P`
   - Run: **"MCP: List Servers"**
   - Select **ibmi-ia-tools** → **Restart**
3. **Test in Copilot Chat** — switch to Agent mode and ask a question that should trigger your tool
4. **Verify** the SQL returns correct results and the response makes sense

### Validating SQL separately

Before adding a tool, you can test the SQL directly:

- **IBM ACS Run SQL Scripts**: Paste the SQL (replace `${IA_LIBRARY}` with your library name and `:param_name` with test values)
- **Mapepire client**: Connect via port 8076 and execute the query
- **VS Code Db2 for i extension**: Run SQL directly from VS Code

---

## Contributing via Pull Request

We welcome contributions! Whether it's a new tool, a bug fix, or an improvement to an existing tool.

### Step-by-step PR workflow

```
1. Fork          → Click "Fork" on GitHub to create your copy
2. Clone         → git clone https://github.com/<you>/ia-tools-ibmi.git
3. Branch        → git checkout -b add-my-new-tool
4. Edit          → Add/modify tools in impact-analysis.yaml
5. Test          → Verify with VS Code Copilot (see "Testing your changes")
6. Commit        → git add impact-analysis.yaml && git commit -m "feat: add ia_my_tool"
7. Push          → git push origin add-my-new-tool
8. Open PR       → Go to GitHub, click "Compare & pull request"
9. Review        → Maintainer reviews your PR, may request changes
10. Merge        → Once approved, maintainer merges into main
```

### PR checklist

Before submitting, verify:

- [ ] Tool name starts with `ia_` (key) / `ia-` (MCP name)
- [ ] SQL is read-only (`SELECT` only — no INSERT/UPDATE/DELETE)
- [ ] Uses `${IA_LIBRARY}` for the library reference (never hardcode)
- [ ] Uses `:param_name` syntax for all SQL parameters
- [ ] Includes `FETCH FIRST :limit ROWS ONLY` for bounded results
- [ ] `description` mentions which iA table(s) are queried
- [ ] Tested against an iA repository and returns correct results
- [ ] Includes sensible default values for optional parameters

### Contribution guidelines

- **One tool per PR** is preferred (easier to review), but related tools can be grouped
- Include a brief description in the PR of what the tool does and example output
- If your tool queries a new iA table not listed in the "iA Tables" section below, add it to the table in your PR
- Keep SQL readable — use clear column aliases and indentation

---

## iA Tables

These tools query the following iA repository tables (pre-parsed IBM i source metadata):

| Table | Purpose |
|-------|---------|
| `IAALLREFPF` | Cross-reference: object-to-object relationships |
| `IAPGMCALLS` | Call graph: CALL, CALLP, bound module references |
| `IAFIDTL` | Field-level details for database files |
| `IAPGMVARS` | Program variables (standalone, DS subfields, indicators) |
| `IAPGMDS` | Data structure definitions |
| `IACALLPARM` | Parameters passed to external calls |
| `IASUBRDTL` | Subroutine (BEGSR/ENDSR) details |
| `IAOVRPF` | OVRDBF (Override with Database File) statements |
| `IAPGMINF` | Program/module metadata |
| `IAPGMREF` | RPG token-level source analysis |
| `IACPGMREF` | CL token-level source analysis |
| `OBJECT_DETAILS` | Master object inventory |
| `IA_DASHBOARD_DETAIL` | Dashboard summary (categories, line counts) |
| `REPO_CONFIGURATION` | Repository configuration settings |
| `IAEXCPLOG` | Parser exception log |
| `DDSTODDL_FILE_CONVERSION_DETAILS` | DDS-to-DDL conversion status |
| `IAOBJMAP` | Object-to-source member mapping |
| `IAOBJECT` | Object lifecycle metadata (create/change/last-used dates) |
| `IA_CODE_INFO` | Code complexity metrics per source member |
| `IASRCMBRID` | Source member master |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `npx: Access is denied` (Windows) | Install globally instead: `npm install -g @ibm/ibmi-mcp-server`, then edit `.vscode/mcp.json` — change `"command"` to `"node"` and prepend the full path to `dist/index.js` in `"args"` |
| MCP server not starting | Check Node.js version: `node --version` (must be 18+) |
| Tools not showing in Copilot Chat | Make sure you're in **Agent** mode (not Ask or Edit mode). Click the tools icon to enable/disable specific tools |
| `Connection refused` on port 8076 | Verify Mapepire is running on your IBM i — check with your system administrator |
| Tools return no data | Check `IA_LIBRARY` in `.env` matches your iA library name. Verify the user profile has `*READ` authority to the iA tables |
| YAML syntax error | Validate the file: `npx js-yaml impact-analysis.yaml` |
| MCP server shows stale tools after edit | Restart the server: Command Palette → "MCP: List Servers" → select server → Restart |

---

## License

Apache-2.0 — see [LICENSE](LICENSE).
