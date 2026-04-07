# Contributing to iA Tools for IBM i

Thanks for your interest in contributing! This guide covers how to report issues, suggest new tools, and submit pull requests.

## Quick Start

```bash
# 1. Fork and clone
git clone https://github.com/<your-username>/ia-tools-ibmi.git
cd ia-tools-ibmi

# 2. Set up your IBM i connection
cp .env.example .env
# Edit .env with your DB2i credentials and IA_LIBRARY

# 3. Start the MCP server
npm install -g @ibm/ibmi-mcp-server
npx @ibm/ibmi-mcp-server --transport http --tools ./impact-analysis.yaml

# 4. Test in VS Code Copilot (Agent mode) or Claude Code
```

## Ways to Contribute

### File an Issue

Use issue templates for structured feedback:

| Template | When to use |
|----------|-------------|
| **Bug Report** | Tool returns wrong data, SQL error, unexpected behavior |
| **New Tool Request** | Suggest a new `ia_*` tool |
| **Improvement** | Enhance an existing tool (add parameter, better output) |
| **Question** | Setup help, usage clarification |

### Submit a Pull Request

1. **Fork** the repo and create a branch (`git checkout -b add-my-tool`)
2. **Edit** `impact-analysis.yaml`
3. **Test** your changes against an iA repository
4. **Commit** with a descriptive message (`feat: add ia_my_tool`)
5. **Push** and open a PR

## Tool Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Tool key | `ia_snake_case` | `ia_where_used` |
| MCP name | `ia-kebab-case` | `ia-where-used` |
| Parameters | `snake_case` | `object_name` |
| Library | `${IA_LIBRARY}` | Never hardcode |
| Bindings | `:param_name` | Prevents SQL injection |
| Limit | `FETCH FIRST :limit ROWS ONLY` | Always include |

## Testing Your Tool

1. **Validate YAML syntax**:
   ```bash
   npx js-yaml impact-analysis.yaml
   ```

2. **Restart the MCP server** after editing (Ctrl+C, then re-run)

3. **Test in Copilot/Claude Code** — ask a question that triggers your tool

4. **Verify** the SQL returns correct results

## PR Checklist

Before submitting, verify:

- [ ] Tool name follows `ia_`/`ia-` convention
- [ ] SQL is read-only (`SELECT` only)
- [ ] Uses `${IA_LIBRARY}` (no hardcoded library)
- [ ] Uses `:param_name` bindings
- [ ] Includes `FETCH FIRST :limit ROWS ONLY`
- [ ] Description mentions queried iA table(s)
- [ ] Tested against an iA repository

## Keeping Your Fork Updated

```bash
# One-time: add upstream remote
git remote add upstream https://github.com/PIO-Anurag-Garg/ia-tools-ibmi.git

# Sync with upstream
git fetch upstream
git merge upstream/main
git push origin main
```

Or use GitHub's **"Sync fork"** button on your fork's page.

## Guidelines

- **One tool per PR** preferred (easier to review)
- Keep SQL readable with clear column aliases
- Include example output in the PR description
