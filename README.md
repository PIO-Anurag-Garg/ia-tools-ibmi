# iA Tools for IBM i — Impact Analysis MCP Tool Definitions

SQL-based [MCP](https://modelcontextprotocol.io/) tool definitions for **Impact Analysis (iA)** on IBM i. These tools query pre-parsed IBM i source metadata in the iA repository (built by [programmers.io](https://www.programmers.io/)) to provide cross-reference lookups, call hierarchy analysis, field impact analysis, dead code detection, and more.

## What's in this repo

| File | Description |
|------|-------------|
| `impact-analysis.yaml` | 24 MCP tool definitions for iA queries |
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

## Prerequisites

- **IBM i** with Db2 for i (accessible via Mapepire on port 8076)
- **iA repository** — the iA product from [programmers.io](https://www.programmers.io/) must have parsed your IBM i source code into the iA tables
- **MCP-compatible AI agent** — any agent that supports YAML MCP tool definitions (e.g., Claude Code with `ibmi-mcp-server`, Agno, etc.)

## Setup

1. Copy `.env.example` to `.env` and fill in your Db2i connection details:
   ```
   DB2i_HOST=your-ibmi-host
   DB2i_USER=your-user
   DB2i_PASS=your-password
   DB2i_PORT=8076
   IA_LIBRARY=SDK01
   ```

2. Set `IA_LIBRARY` to the library where your iA repository tables live (default: `SDK01`).

3. Load `impact-analysis.yaml` in your MCP server or agent framework.

## iA Tables

These tools query the following iA repository tables:

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

## YAML Tool Format

Each tool follows this structure:

```yaml
ia_tool_name:
  name: ia-tool-name
  source: ibmi
  description: >
    What this tool does, when to use it, and which iA table it queries.
  parameters:
    - name: param_name
      type: string          # string, integer
      description: "What this parameter controls"
      required: true        # or false
      default: "*ALL"       # optional default value
  statement: |
    SELECT columns
    FROM ${IA_LIBRARY}.TABLE_NAME
    WHERE COLUMN = :param_name
    FETCH FIRST :limit ROWS ONLY
```

**Key conventions:**
- `${IA_LIBRARY}` — replaced at runtime with the iA repository library name
- `:param_name` — parameter placeholders (bound at execution time)
- `source: ibmi` — uses the Db2i connection defined in the `sources` block

## Contributing

We welcome contributions! Here's how to add a new iA tool:

1. **Fork this repo** and create a branch
2. **Add your tool** to `impact-analysis.yaml` following the YAML format above
3. **Test it** against your iA repository to verify the SQL returns correct results
4. **Submit a PR** with:
   - The tool definition
   - A brief description of what iA table(s) it queries
   - Example output (optional but helpful)

### Contribution guidelines

- Tool names must start with `ia_` (underscore-separated)
- MCP names must start with `ia-` (hyphen-separated)
- Include a descriptive `description` that mentions which iA table(s) are queried
- Use `${IA_LIBRARY}` for the library reference (never hardcode a library name)
- Use `:param_name` syntax for all parameters
- Include sensible defaults and `FETCH FIRST :limit ROWS ONLY` for bounded results
- Keep SQL read-only (SELECT only, no INSERT/UPDATE/DELETE)

## License

Apache-2.0 — see [LICENSE](LICENSE).
