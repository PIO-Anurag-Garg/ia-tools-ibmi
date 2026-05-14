# Program Documentation Reference

Use this guide whenever a user asks for a **technical program document** for an IBM i program. Follow every step in order.

---

## Step 1 — Smart Profile Selection

**Check the user's request for profile indicators:**

| User Says | Action |
|-----------|--------|
| "document program X" (no preferences) | Use DEFAULT profile → skip to Step 2 |
| "standard spec", "full documentation" | Use DEFAULT profile → skip to Step 2 |
| "customize", mentions audience / sections | Run Phase 0 discovery |
| "for business analysts", "for architects" | Run Phase 0 discovery |

**DEFAULT profile (80% of users):**
- Audience: Developers
- Detail: Full inventory (all subroutines)
- Sections: All
- Format: Markdown
- Template: `template-developer.md`

If using DEFAULT, show one line: `"Using standard profile (full technical spec for developers). Say 'customize' to change."`

**Phase 0 Discovery (only when customization requested):**
Present all questions in **one message** — not sequentially:

```
Please configure your documentation preferences:

**Audience** (select all that apply):
☐ A) New developers / onboarding team
☐ B) Business analysts (non-technical)
☐ C) Architects / auditors
☐ D) Support / operations team

**Detail Level**:
○ A) Full inventory — every subroutine documented (Recommended)
○ B) Key subroutines only — business rules focus
○ C) Summary counts only

**Priority Sections** (select all that apply):
☐ A) Business rules (BR-xxx list)
☐ B) Call hierarchy (what calls what)
☐ C) Processing flow (step-by-step narrative)
☐ D) File and data area usage
☐ E) Error handling

**Format**:
○ A) Markdown with tables (Recommended)
○ B) Plain text
○ C) Both (markdown primary, plain text summary)
```

Wait for one response, then proceed.

---

## Step 2 — One-call Inventory

**Use the bundled tool — one call instead of seven:**

```
ia_program_spec_bundle(program_name=X)
```

**⛔ HARD STOP — Multiple Versions Detected:**
If LOOKUP returns multiple rows (member exists in several libraries or source files):
1. Present the full version table to the user: library, source file, type, line count, last changed
2. Ask: *"I found N versions of [PROGRAM]. Which version would you like me to document?"*
3. **Wait for the user's answer. Do NOT proceed to Step 3 without explicit confirmation.**
4. Once confirmed, re-run with `library=<chosen>` and proceed.

A footnote is **not** acceptable — always pause and ask. Silently selecting a version is a documentation error.

If LOOKUP returns zero rows, stop and tell the user the member was not found in iA. Suggest `ia_object_lookup(object_name='%X%')` with a wildcard.

**⛔ HARD STOP — Any Mid-Process Doubt:**
If any of the following arise during Steps 2–7, **pause and ask the user before continuing:**

| Condition | What to Ask |
|-----------|-------------|
| Multiple versions in LOOKUP | "I found N versions — which library/source file should I document?" |
| CALLED_BY_COUNT > 0 but CALLERS section is empty | "iA shows N caller(s) but couldn't resolve them. Want me to run `ia_find_object_usages` to trace them first?" |
| Constant value suspicious (e.g. validator may be missing digits) | State the uncertainty inline in the BR and flag it in the quality report — do not silently assume |
| Ambiguous build pipeline order or version mismatch | Note the uncertainty and flag in quality report |

**General rule:** When in doubt, ask. A brief pause for clarification produces a more accurate document than a confident guess.

**Bundle section → document section mapping** (sections refer to the new template-developer.md structure):

| Bundle section | Document section fed | What to extract |
|----------------|----------------------|-----------------|
| `LOOKUP` | Header, Section 1 (Overview) | All libraries where member exists; if multiple, ask user which to document. **Capture the actual library name and use it everywhere.** |
| `COMPLEXITY` | Section 9 + Technical Statistics | TOTAL_LINES, EXEC_LINES, IF/DO/SEL/WHEN, SBR/PROC/SQL/GOTO, CALL_PGM/CALLED_BY/FILES/DSPF |
| `FILES` | Section 3 (Files and Data Sources) | File list, library, type, PREFIX, RENAMED, RCDFMT, INDICATOR_DS, FILE_INFO_DS |
| `CALLEES` | Section 5 (Subroutine / Procedure Analysis) + Mermaid diagram | CALLED_OBJCT + CALLED_PROC — verified callees only |
| `CALLERS` | Section 9 (Key Observations — dependencies) | Callers verified by `ia_call_hierarchy` only |
| `SUBROUTINES` | Sections 4, 5, 6 | **Complete** subroutine list — every BEGSR is documented in §5 and is a candidate for BRs in §6 |
| `PARAMS` | Section 2 (Technical Summary — entry parameters) | Entry parameters from PI/PR — name, type, length, sequence, KEYWORDS, business meaning |
| `BINDINGS` | Section 5 (External service programs) | *SRVPGM, *BNDDIR, *MODULE references |

**BINDINGS multi-SRVPGM handling:**
When BINDINGS returns multiple `*SRVPGM` entries:
1. Group by `REFERENCED_OBJ` (service program name)
2. For each SRVPGM, list its bound procedures from the CALLEES section (match by CALLED_OBJCT)
3. Present as table:

| Service Program | Library | Procedures Used | Coupling Level |
|-----------------|---------|-----------------|----------------|
| SRVPGM1 | MYLIB | PROC1, PROC2, PROC3 | High (3+) |
| SRVPGM2 | MYLIB | PROCX | Low (1) |

Coupling levels: **High** (>3 procedures), **Medium** (2-3), **Low** (1). Modules (`*MODULE`) go in a separate "Bound Modules" subsection.

**Fallback:** If the bundle errors or returns zero LOOKUP rows, use this sequence:

```
1. ia_member_lookup(member_name=X)           → existence check + metadata
2. ia_code_complexity(member_name=X)         → complexity metrics (lines, IF/DO/SQL counts)
3. ia_program_files(member_name=X)           → file usage with PREFIX/RENAME
4. ia_call_hierarchy(program_name=X, direction=BOTH) → callers and callees
5. ia_subroutines(member_name=X)             → BEGSR/ENDSR details
6. ia_procedure_params(member_name=X)        → PI/PR parameter signatures
7. ia_object_references(object_name=X)       → SRVPGM/BNDDIR/MODULE bindings
```

Run these in order; skip any that return zero rows. Combine results to populate sections.

---

## Step 3 — Source + Business Rule Extraction

Retrieve source and extract business rules using the appropriate source tool **based on member type** (from Step 2's LOOKUP section):

| MEMBER_TYPE | Tool to use |
|-------------|-------------|
| `RPGLE`, `SQLRPGLE`, `RPG`, `SQLRPG` | `ia_rpg_source` |
| `CLLE`, `CLP`, `CL` | `ia_cl_source` |

Call the appropriate tool with:
- `member_name`: The program's source member name
- `library_name`: The library you confirmed in Step 2 (don't leave as `*ALL` when multiple versions exist)
- `limit`: See pagination rule below
- Optional (`ia_rpg_source` only): `source_spec` to filter by RPG spec type (C, D, F, H, I, O, P)

**Pagination — mandatory for sources >10,000 lines:**

Both `ia_rpg_source` and `ia_cl_source` have a hard `limit` cap of **10,000 lines per call**. Before fetching, read `TOTAL_LINES` from Step 2's COMPLEXITY section:

| TOTAL_LINES | Action |
|---|---|
| ≤ 10,000 | One call: `limit=10000, offset=0` (or set limit to TOTAL_LINES for a tighter fetch) |
| > 10,000 | Loop: `offset=0,10000,20000,…` with `limit=10000` until you have all lines. Stop when a call returns fewer than 10,000 rows. |

**Never write a spec from a partial source.** Real programs can exceed 30,000 lines — a single call returns under 30% of these. Verify completeness: the highest `SOURCE_RRN` you fetched should equal `TOTAL_LINES` from COMPLEXITY.

This returns source lines with line numbers, spec types, format classification, and source text. Complexity metrics are already available from Step 2's COMPLEXITY section.

**Auto-cluster subroutines by name patterns:**

| Pattern | Cluster Type | Examples |
|---------|--------------|----------|
| `*VALID*`, `*CHECK*` | Validation | VALIDATECUST, CHECKINPUT |
| `*CALC*`, `*COMPUTE*` | Calculation | CALCPRICE, COMPUTETAX |
| `*EMAIL*`, `*SEND*`, `*NOTIF*` | Communication | SENDEMAIL, SENDNOTIF |
| `*INIT*`, `*SETUP*` | Initialization | INITVARS, SETUPSCREEN |
| `*CLEAN*`, `*CLOSE*`, `*END*` | Cleanup | CLEANUP, CLOSEFILE |
| `*ERROR*`, `*ABORT*` | Error Handling | ERRORHANDLER, ABORTPGM |
| `*READ*`, `*WRITE*`, `*UPDATE*`, `*DELETE*` | I/O | READCUST, WRITEORDER |
| `*PROCESS*`, `*HANDLE*` | Workflow | PROCESSORDER, HANDLEREQ |

For each cluster, scan source for:
- IF/WHEN/SELECT conditions → validation rules
- Constants (named or literal) → business constants
- SQL WHERE clauses → data filters
- GOTO/EXSR patterns → workflow rules

**BR-xxx format:**
```markdown
- **BR-001** [VALIDATION] — Customer number must be numeric (line 145, subroutine: VALIDATECUST)
- **BR-002** [CALCULATION] — Order total = sum(line items) + tax (line 230, subroutine: CALCORDERTOTAL)
```

---

## Step 4 — Template Selection

| Audience | Template | Focus |
|----------|----------|-------|
| Developers / onboarding (DEFAULT) | [`templates/template-developer.md`](templates/template-developer.md) | Full technical detail |
| Business analysts | [`templates/template-business.md`](templates/template-business.md) | Plain language, business rules |
| Architects / auditors | [`templates/template-architect.md`](templates/template-architect.md) | Architecture, dependencies, modernization |
| Support / operations | [`templates/template-operations.md`](templates/template-operations.md) | Runtime, monitoring, troubleshooting |
| Multiple audiences | `template-developer.md` | Most comprehensive |

See [`templates/README.md`](templates/README.md) for comparison matrix.

---

## Step 5 — Document Assembly Rules (template-developer.md, default tech-spec structure)

> **Tone rule (applies to every section):** Use business-friendly language with technical accuracy. **Never** describe the code line-by-line. **Never** assume a pattern that is not explicitly present in the source.

> **Library resolution rule (applies everywhere):** Always use the **actual library name** (e.g., `PRDLIB`) wherever a library appears — header, file tables, output sections, mermaid labels, glossary. Resolve from the bundle's LOOKUP / FILES sections; never leave a placeholder in the deliverable.

### Section 1 — Program Overview
- Program name (and any alternate name in source comments).
- One-to-two sentence primary purpose.
- Bulleted business problem / functionality addressed.
- Execution type — Batch, Interactive, or Online — justified by DSPF count, F-spec usage, or scheduling context.
- Key outputs and how they are consumed downstream.

### Section 2 — Technical Summary
- Program ID, RPG type (RPG III / RPG IV fixed / RPG IV free / SQLRPGLE / CLLE) — sourced from MEMBER_TYPE in iA, not from reading the source header.
- **Mandatory:** call `ia_procedure_params` first. If it returns rows, populate the entry-parameters table exactly with **business meaning** for each parameter.
- Only write "No entry parameters declared." if `ia_procedure_params` returns zero rows AND source has no `dcl-pr`/`dcl-pi`/`*ENTRY PLIST`. Never write "Not determinable."
- Notable revisions / assumptions / commented-out logic — capture revision tags, disabled file integrations, and any explicit assumption stated in source comments.

### Section 3 — Files and Data Sources
- Populate from bundle FILES section only — do not add files from source that aren't confirmed by iA.
- For each file include: name, **actual library**, usage type (Input / Output / Update / Reference), business purpose.
- **Data areas (*DTAARA) are NOT files.** Add a separate subsection: **"Data Areas Used"** with columns Name, Library, Size, Operations (IN/OUT), Purpose.
- If `ia_object_references` shows *FILE entries not in FILES section, note them as "detected via object reference, not F-spec."

### Section 4 — Main Processing Logic
- Narrate the program's execution flow **in the same sequence in which it actually runs**, from start to end. Numbered steps or paragraph prose are both acceptable.
- For every read / write / update / chain on a file, mention which **key fields** are used, which **fields are derived**, and which **fields are written or updated**.
- When processing routes through a subroutine, mention its **name** and a one-line summary of what it does (full detail in Section 5).
- Describe how control flows between major logic blocks; identify distinct paths only if they naturally occur.
- **Do not pre-classify** logic by category (dates, control breaks, etc.) — narrate in execution order.
- Add line-number anchors *(line NNN)* where they aid traceability.

### Section 5 — Subroutine / Procedure Analysis
- Use the **complete SUBROUTINES list** from Step 2 — never document subroutines selectively. Every non-trivial routine gets its own block with Name, Purpose, High-level behavior, Key decisions / calculations.
- If a subroutine name implies a capability (e.g., SENDEMAILPROCESS, POPULATEMEMBEREXCLUSIONDETAILS) not yet reflected anywhere, document it even if you cannot see the full logic.
- **Call Hierarchy Diagram (text-based ASCII tree)** — include if `CALL_PGM_COUNT > 0` OR CALLEES has rows. Use the **actual library** in the parent node label (e.g., `ORDENTRY (PRDLIB)`). One-level depth unless the user asks for deeper traversal.
- Only list callees that appear in CALLEES; only list service programs from BINDINGS. If `CALL_PGM_COUNT = 0` but CALLEES has rows, prepend the "bound CALLP only" note from Step 6.
- Callers go in Section 9 (Key Observations) — only if confirmed by CALLERS section.

### Section 6 — Business Rules and Calculations
- Use the **BR-xxx** identifier format with a category tag and source anchor: `**BR-001** [VALIDATION] — <rule> (line NNN, subroutine: NAME)`.
- Categories: VALIDATION, CALCULATION, CLASSIFICATION, STATUS / FLAG, EXCEPTION, ACCUMULATION.
- Group rules by what they do in the business: accumulations / totals / rollups, classification, status / flag-driven behavior, special conditions / exception handling.
- Coverage target: BR count ≥ 50% of subroutine count. Below that, surface a warning in the quality report.
- Character-constant validators missing digits → always flag as "verify against source."

### Section 7 — Output File Behavior
- For each file the program writes or updates, document conditions for create vs. update, field population / accumulation logic, and the relationship between input and output fields.

### Section 8 — Glossary
- Define every important keyword, abbreviation, and field used in the program in plain language, including unit / domain when relevant.

### Section 9 — Key Observations (Optional)
- Complexity or risk areas (high IF/DO counts, GOTOs, deep nesting, large data structures).
- Dependency on reference data or external programs (DAYMINUS, lookup files, data areas).
- Modernization / refactoring opportunities.
- Caller summary: if callers list is empty but `CALLED_BY_COUNT > 0`, write "iA reports N caller(s) but no matching record found in the call-hierarchy data — likely a CL→RPG call; manual verification recommended."

### Technical Statistics block
Include **all** complexity metrics from COMPLEXITY section (Total / Exec lines, IF / DO / SEL / WHEN, SQL, Subroutine — *agents often omit this* — Procedure, GOTO, Files, DSPF, Calls Out, Called By) plus an overall **Complexity Assessment** (Low / Medium / High / Critical).

---

## Step 6 — Call Hierarchy Diagram

After assembling Section 5 (Subroutine / Procedure Analysis), add a call hierarchy using CALLEES data — **always with the actual library name** in the parent label.

**Skip if:** CALL_PGM_COUNT = 0 **AND** CALLEES section is empty (no bound procedure calls confirmed by iA).

**Include if:** CALL_PGM_COUNT > 0 (traditional CALL opcodes) **OR** CALLEES section has any rows (bound CALLP procedure calls confirmed by iA) — even if CALL_PGM_COUNT = 0.

If CALL_PGM_COUNT = 0 but CALLEES has rows (bound procedures only), add this note before the diagram:
> **Note:** No traditional `CALL` opcodes detected. Diagram shows confirmed bound procedure calls (CALLP via prototype). The full prototype declaration list is in Section 2.

**Include in:** `template-developer.md` and `template-architect.md` only (not business or operations templates).

**Format — text-based ASCII tree in a fenced `text` code block:**

\`\`\`markdown
## Call Hierarchy Diagram

\`\`\`text
PROGRAMNAME (LIBRARY)
│
├── [CALLP] CALLEE1
├── [CALLP] CALLEE2          ×3
├── [CALLP] CALLEE3 (proc: PROCNAME)
├── [CALL]  CALLEE4
└── [CALL]  CALLEE5
\`\`\`
\`\`\`

**Why text instead of Mermaid:** Mermaid diagrams render to PNG when exported to Word/PDF. With many callees (IBM i programs often have 10–30), the PNG either becomes unreadably wide (`graph TD`) or unreadably tall (`graph LR`), and labels degrade at any zoom level. A monospace text tree in a fenced code block renders natively in every output format (Markdown preview, Word `.docx`, PDF), stays selectable/searchable, never blurs, and survives copy-paste.

**Conventions:**
- Parent line: `PROGRAMNAME (LIBRARY)` — use the actual library name.
- Use `├──` for non-last children, `└──` for the last child.
- Tag each callee with `[CALLP]` (bound procedure call) or `[CALL]` (traditional program call) — left-pad `[CALL]` with one extra space so callee names align in monospace.
- If a callee is invoked more than once, append `×N` aligned in a trailing column.
- If `CALLED_PROC` is populated and differs from `CALLED_OBJCT`, append `(proc: PROCNAME)` after the name.
- Keep to direct callees only (one level depth) unless the user asks for deeper traversal.

**Add a brief legend below the tree** (see template files for the exact text).

---

## Step 7 — Automated Verification Pass

**Run all checks before presenting the document:**

| Check | Rule | Action if Failed |
|-------|------|------------------|
| ✓ Section 2 (Technical Summary) | No "Not determinable" parameters without `ia_procedure_params` call | **ERROR** — must fix |
| ✓ Section 3 (Files & Data Sources) | No *DTAARA in file table; library column shows actual library | **ERROR** — move data areas to subsection; substitute library |
| ✓ Section 4 (Main Processing Logic) | Narrative follows execution order; no pre-classification by category | **ERROR** — re-narrate |
| ✓ Section 5 (Subroutines) | Every subroutine from SUBROUTINES list appears; CALL_PGM_COUNT=0 → no external calls listed | **ERROR** — fill missing routines / remove unverified calls |
| ✓ Section 6 (Business Rules) | BR count ≥ 50% of subroutine count | **WARNING** — may be incomplete |
| ✓ Section 7 (Output File Behavior) | Every Output / Update file from Section 3 has a corresponding block | **WARNING** — incomplete |
| ✓ Section 8 (Glossary) | Important fields/keywords referenced elsewhere are defined | **WARNING** — gaps |
| ✓ Statistics | All 9+ complexity metrics present | **WARNING** — incomplete stats |
| ✓ Header | Library version stated; actual library name used throughout (no placeholders) | **ERROR** — substitute actual library |

Fix all **ERRORs** before delivery. Present **WARNINGs** with notes in the quality metrics footer.

**Validation report format:**
```
✅ All checks passed — document ready for delivery
⚠️ 2 warnings detected:
  - Section 4: Only 5 BRs for 12 subroutine clusters (42% coverage)
  - Section 6: Subroutine cluster "Email" not in processing flow
❌ 1 error detected:
  - Section 3: Data area MYDTAARA found in file table
```

---

## Step 8 — Branding and Quality Metrics

**Apply branded header:**
```markdown
# Technical Specification — <PROGRAM>

**Author:** iA by programmers.io
**Date:** <YYYY-MM-DD>
**Library version documented:** <LIBRARY> | **Source file:** <SRCPF> | **Member type:** <MEMBER_TYPE>
```

**Add quality metrics footer before the final branding line:**
```markdown
---

## Documentation Quality Report

| Metric | Score | Status |
|--------|-------|--------|
| Completeness | <X>% | ✅/⚠️ <N>/<TOTAL> sections populated |
| Verification Rules | <X>% | ✅/❌ <N>/7 rules passed |
| Business Rules Coverage | <X>% | ✅/⚠️ <N> BRs for <M> subroutine clusters |
| Source Traceability | ✅/⚠️ | Line numbers included: YES/NO |
| Freshness | Current | ✅ Generated <YYYY-MM-DD> |

**Validation Warnings (if any):**
- <Warning text>

**Recommendations:**
- <Recommendation text>
```

**Save location — always write output here, and always create a NEW versioned file:**

```
docs/program-specs/{PROGRAM_NAME}/{PROGRAM_NAME}_{DocType}_v{N}.md
```

| Document type | File name pattern |
|---|---|
| Technical specification (default) | `{PROGRAM_NAME}_Technical_Specification_v{N}.md` |
| Functional document | `{PROGRAM_NAME}_Functional_Document_v{N}.md` |
| Operations guide | `{PROGRAM_NAME}_Operations_Guide_v{N}.md` |
| Architecture review | `{PROGRAM_NAME}_Architecture_Review_v{N}.md` |

**Versioning rule (mandatory — never overwrite an existing document):**

1. Before writing, list `docs/program-specs/{PROGRAM_NAME}/` and find the highest existing `_v{N}` suffix for the same DocType.
2. New file = `_v{N+1}`. If no prior version exists, start at `_v1`.
3. **Never** overwrite `_v1`, `_v2`, etc. — every new request produces a new file. Older versions stay as historical artifacts.
4. Unversioned files (e.g., `IAINIT_Technical_Specification.md` with no `_v` suffix) are treated as legacy — leave them in place; the next document still goes to `_v{highest+1}`.

**Folder rules:**

- Create `docs/program-specs/{PROGRAM_NAME}/` if it does not exist. **Do not** save anywhere else (project root, `BIO60R_Program_Analysis.md` style files at repo root are wrong — always use the program subfolder).
- If the file is an explicitly-labeled draft, save it to `docs/program-specs/{PROGRAM_NAME}/drafts/{PROGRAM_NAME}_{DocType}_v{N}_draft.md` instead. Final/canonical iterations go to the subfolder root.
- Exported formats (`.docx`, `.pdf`) go in the same subfolder as their `.md` source, with the matching `_v{N}` suffix — never in the parent `docs/program-specs/` root.

**Final footer:** `*Analysis powered by iA from [programmers.io](https://programmers.io/ia/)*`

---

## Step 9 — Export to Word / PDF (on request)

If the user asks to "export to Word", "save as DOCX", "convert to PDF", "give me a PDF" — or similar — **after** the markdown spec has been generated, run the bundled converter scripts.

**Scripts shipped with this skill** (under `scripts/` in the skill root):

| Format | Script | Dependencies |
|--------|--------|--------------|
| Word (DOCX) | `convert_md_to_docx.py` | `pip install python-docx requests` |
| PDF | `convert_md_to_pdf.py` | `pip install reportlab` |

**Invocation:**

```bash
python convert_md_to_docx.py docs/program-specs/{PROGRAM}/{PROGRAM}_Technical_Specification_v{N}.md
python convert_md_to_pdf.py  docs/program-specs/{PROGRAM}/{PROGRAM}_Technical_Specification_v{N}.md
```

Each script writes its output next to the source `.md` with the `.docx` / `.pdf` extension — the same `docs/program-specs/{PROGRAM}/` subfolder, preserving the `_v{N}` suffix automatically.

**Operational rules:**

- **One format at a time** — only generate the format the user asked for. Don't produce both unless explicitly requested.
- **Missing dependency** — if a script exits with `Error: python-docx is not installed` (or `reportlab`), run the printed `pip install …` command, then re-run the conversion. Do not silently swap to a different tool.
- **No network calls expected** — both scripts are self-contained. The DOCX converter only reaches `mermaid.ink` if it encounters a ```` ```mermaid ```` fenced block; specs produced by this skill use text-based ASCII trees instead, so this path is not exercised.
- **Both scripts apply iA branding automatically** — PDF cover page, DOCX author metadata. Do not re-add the branding manually.
- **Output report** — after a successful conversion, tell the user the full output path so they can open it directly.

**What renders well:**

- All markdown the skill produces — headings, tables, fenced code blocks, lists, blockquotes, inline code, links.
- Text-based ASCII call-hierarchy trees in fenced ```` ```text ```` blocks render as native monospace code in both DOCX and PDF (which is precisely why this skill uses them instead of Mermaid).

---

## Verification Rules (Never Violate)

| Rule | Why |
|------|-----|
| Never assert a called program without call-hierarchy confirmation from `ia_call_hierarchy` | Unverified calls are worse than missing ones |
| Never use line counts from reading source | Use the iA complexity metrics — counting is error-prone and version-specific |
| Never put data areas in the file table | They are accessed via IN/OUT, not F-specs; separate section required |
| Never leave Section 2 as "Not determinable" without first calling ia_procedure_params | The tool exists; use it |
| Never document subroutines selectively | Get the full list from SUBROUTINES section first |
| Never omit a functional capability implied by subroutine names | SENDEMAILPROCESS = email; document it even if you can't see the logic |
| Always state which library version is being documented | Multiple versions often exist with different line counts |
| Always use the actual library name throughout the deliverable | Leaking placeholder text makes the document look auto-generated and broken. Resolve from LOOKUP/FILES sections. |
| Always create a new `_v{N+1}` file; never overwrite an existing document | Each request must yield a new artifact so prior iterations are preserved as history |

---

## Common Traps

| Trap | Symptom | Fix |
|------|---------|-----|
| Subroutine tunnel vision | Document 10 of 46 subroutines | Step 2 is mandatory — get full list, group into clusters |
| Missing capability | Email, exclusions, scheduling absent from BRs and flow | Every subroutine cluster must appear in BRs and flow |
| Unverified external calls | Asserting calls despite CALL_PGM_COUNT=0 | Check COMPLEXITY CALL_PGM_COUNT before asserting external calls |
| Data area as file | Data area listed in file table | Separate "Data Areas" subsection — data areas use IN/OUT opcodes |
| Silent version selection | Documenting one library version without saying so | **HARD STOP** — after Step 2, if multiple versions found, present the version table and wait for explicit user confirmation. A footnote is not sufficient; the user must choose. |
| Parameter gap | Section 2 all "Not determinable" | `ia_procedure_params` is mandatory before writing Section 2 |
| Caller assertion | Caller listed with no CALLERS result in iA | Only assert callers confirmed by `ia_call_hierarchy` |
| Character validator gap | Alphanumeric validator missing digits | Flag any alphanumeric validator missing digits as "verify against source" |

---

## Canonical Tech-Spec Prompt (default for every "technical specification" request)

When the user asks for a technical specification / program analysis document for an IBM i program, this is the **mandatory internal prompt** that drives the writing. It is not shown to the user — but every document produced must conform to it.

```
You are an expert IBM i (AS/400) RPG consultant and legacy application analyst.

Analyze the RPG program source provided below and generate a clear, professional
Program Analysis Document suitable for senior developers, architects, and support teams.

Explain what the program does, how it works, and the business rules it enforces.
Use business-friendly language while retaining technical accuracy.

DO NOT describe the code line by line.
DO NOT assume the presence of any specific business pattern unless it is explicitly present in the code.

--------------------------------
DOCUMENT STRUCTURE
--------------------------------

1. Program Overview
   - Program name and primary purpose
   - Business problem or functionality it addresses
   - Execution type (Batch / Interactive / Online)
   - Key outputs and how they are used or consumed

2. Technical Summary
   - Program ID and RPG type (RPG III, RPG IV, fixed-format, free-form)
   - Entry parameters and their business meaning (if present)
   - Notable revisions, assumptions, or commented-out logic

3. Files and Data Sources
   For each file used in the program:
   - File name
   - Usage type (Input / Output / Update / Reference)
   - Business purpose of the file

4. Main Processing Logic
   Describe the program's execution flow in the same sequence in which it runs:
   - Explain the logic as it progresses from start to end including all the rules,
     conditions, logic etc.
   - If a file is being read/written/updated, mention which keys it is read on,
     what fields are derived out of it and what fields are updated/written.
   - If processing executes through a sub-routine/sub-procedure, mention its
     name and a high-level description of what is happening inside it.
   - Mention setup, validations, parameter usage, loops, calculations, or
     branching only when they are encountered during execution.
   - Describe how control flows between major logic blocks.
   - Identify distinct processing paths only if they naturally occur in the
     program flow.
   - Do not separate or pre-classify logic by category (such as dates, control
     breaks, etc.).

5. Subroutine / Procedure Analysis
   For each significant subroutine or procedure:
   - Name
   - Purpose
   - High-level behavior
   - Key decisions or calculations handled

6. Business Rules and Calculations
   Summarize important rules discovered in the code, such as:
   - Accumulations, totals, or rollups
   - Classification or categorization logic
   - Status, flag, or code-driven behavior
   - Special conditions or exception handling

7. Output File Behavior
   - Conditions under which records are created or updated
   - How values are populated or accumulated
   - Relationship between input data and output results

8. Glossary of important keywords and fields used in the program.

9. Key Observations (Optional)
   - Complexity or risk areas
   - Dependency on reference data or external programs
   - Opportunities for simplification, refactoring, or modernization

--------------------------------
FORMATTING GUIDELINES
--------------------------------
- Use clear section headings and bullet points
- Be concise, accurate, and professional
- Assume IBM i knowledge but no familiarity with this specific program
- Do not infer or invent logic not present in the source

--------------------------------
iA-SPECIFIC RULES (layered on top)
--------------------------------
MANDATORY SEQUENCE BEFORE WRITING:
1. ia_program_spec_bundle(program_name=X)        → all 8 inventory sections in one call
2. ia_rpg_source OR ia_cl_source (by MEMBER_TYPE) → business rule extraction from source
3. ia_procedure_params(member_name=X)            → entry parameters (mandatory even if PARAMS returned rows)

If the bundle errors, fall back to the seven individual tools.

NON-NEGOTIABLE RULES:
- Every subroutine from SUBROUTINES must appear in Section 5; every cluster
  feeds Section 6 (BRs).
- Do not assert any called program not in CALLEES section.
- Put data areas in a separate "Data Areas Used" subsection — never in the
  file table.
- State which library version is being documented and **use the actual
  library name everywhere** in the deliverable.
- Never write "Not determinable" for parameters without running
  ia_procedure_params first.
- Flag any missing digits in character validators.
- Add a call hierarchy diagram if CALL_PGM_COUNT > 0 OR CALLEES has
  rows (bound procedures), with the actual library in the parent label.
- HARD STOP after Step 2 if LOOKUP returns multiple versions — present table,
  ask user which to document, wait for explicit confirmation.
- HARD STOP on any mid-process ambiguity — pause and ask the user.

OUTPUT FILE NAMING:
- Always write to docs/program-specs/{PROGRAM_NAME}/.
- Always create a NEW versioned file: `{PROGRAM_NAME}_Technical_Specification_v{N+1}.md`
  where N is the highest existing version. Never overwrite previous versions.
```

---

## Reference: Section → iA Tool Map

| Document Section | Primary iA Source | Fallback |
|---|---|---|
| 1. Program Overview | LOOKUP + COMPLEXITY (DSPF/FILE counts for execution type) | Source header comments |
| 2. Technical Summary — Entry Parameters | `ia_procedure_params` | dcl-pr/dcl-pi or *ENTRY PLIST in source |
| 2. Technical Summary — Revisions / Assumptions | Source comments scan | — |
| 3. Files and Data Sources | FILES section (with **actual library**) | `ia_object_references` |
| 3. Data Areas Used | `ia_variable_ops(opcode=IN or OUT)` | Source D-spec / dcl-ds scan |
| 4. Main Processing Logic | Source narrative + SUBROUTINES (for routing) | — |
| 5. Subroutine / Procedure Analysis | SUBROUTINES section + source bodies | — |
| 5. Call Hierarchy Diagram | CALLEES + BINDINGS | — |
| 6. Business Rules | SUBROUTINES → source IF/WHEN/SELECT/SQL clusters | — |
| 7. Output File Behavior | FILES section (Output / Update) + source write/update opcodes | — |
| 8. Glossary | Field names from FILES + variable scan | — |
| 9. Key Observations — Callers | CALLERS section | — |
| 9. Key Observations — Service Programs | BINDINGS section | — |
| Statistics block | COMPLEXITY section | — |
