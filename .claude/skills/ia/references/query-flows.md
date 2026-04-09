# iA Query Flows — Tool Chains for Common Analysis Tasks

This reference documents optimal tool sequences for common queries. Follow these flows to minimize tool calls while maximizing insight.

---

## Tier 1: High-Impact Analysis (Most Valuable)

### QF-1: "What happens if I change this file/field?" (Full Impact Analysis)

**Single-query approach (preferred):**
```
ia_field_impact(field_name="CUSTNO", file_name="CUSTMAST", limit=500)
```
Returns: All programs affected, their usage type (READ/WRITE/UPDATE), and object types.

**Deep-dive chain (if needed):**
```
1. ia_field_impact → Get affected programs
2. ia_where_used(object_name="<srvpgm>", object_type="*SRVPGM") → Only for *SRVPGM in results (cascade check)
3. ia_call_hierarchy(program_name="<critical_pgm>", direction="callers") → Only for critical programs
```

**Optimization:** Stop after step 1 if results are < 10 objects. Only chain for *SRVPGM amplifiers or > 20 affected objects.

---

### QF-2: "Map complete dependency chain for a program"

**Combined approach (2 calls max):**
```
1. ia_program_detail(program_name="ORDENTRY", section="*ALL") → Gets calls, files, subroutines, variables, overrides in ONE query
2. ia_call_hierarchy(program_name="ORDENTRY", direction="both") → Full call tree
```

**Why this works:** `ia_program_detail` with `section="*ALL"` returns 6 types of information in a single query, eliminating the need for separate calls to ia_file_overrides, ia_subroutines, ia_program_variables.

---

### QF-3: "Find dead code / unused objects for cleanup"

**Efficient approach:**
```
1. ia_unused_objects(object_type="*PGM", limit=100) → Zero-reference compiled objects
2. ia_uncompiled_sources(member_type="RPGLE", limit=100) → Orphaned sources never compiled
3. ia_code_complexity(member_name="*ALL", limit=50) → Also shows CALLED_BY_COUNT (0 = dead)
```

**No need to chain** `ia_object_lifecycle` for every object — the unused_objects query already confirms zero references. Only check lifecycle for specific objects the user wants to investigate.

---

### QF-4: "Assess risk before modifying a shared file"

**Two-query approach:**
```
1. ia_where_used_detail(ref_object_name="CUSTMAST", ref_object_type="*FILE", ref_object_lib="AIDEMOLIB")
   → Returns: using objects + source_exist flag
2. ia_file_fields(file_name="CUSTMAST") → Field definitions
```

**Skip** `ia_reference_count` — it's redundant when you already have `ia_where_used_detail` results (just count them).

---

### QF-5: "Find circular dependencies"

**Single query:**
```
ia_circular_deps()
```

**Only chain** `ia_call_hierarchy` if circular deps are found AND user wants to understand the specific call chain.

---

## Tier 2: Program Understanding

### QF-6: "Complete program anatomy"

**Single comprehensive query:**
```
ia_program_detail(program_name="IASRV01SV", section="*ALL", limit=500)
```

Returns CALLS, FILES, SUBROUTINES, VARIABLES, OVERRIDES, CALL_PARAMS in ONE query.

**Only add** `ia_code_complexity` if user specifically asks about complexity metrics.

---

### QF-7: "Understand how data flows through calls"

**Two-query approach:**
```
1. ia_call_hierarchy(program_name="ORDENTRY", direction="called") → Call targets
2. ia_call_parameters(caller_name="ORDENTRY") → Parameters at each call site
```

**Skip** `ia_data_structures` unless user specifically asks about DS layouts.

---

### QF-8: "Identify most complex/risky programs"

**Single query:**
```
ia_code_complexity(member_name="*ALL", limit=20)
```

Returns: IF_COUNT, DO_COUNT, SQL_COUNT, GOTO_COUNT, CALLED_BY_COUNT, TOTAL_OPERATIONS — all complexity metrics in one call.

**Only drill deeper** on specific programs the user identifies as concerning.

---

### QF-9: "Find all file overrides and their chains"

**Two-query approach:**
```
1. ia_override_chain() → All chained overrides (A→B→C)
2. ia_file_overrides(member_name="<specific>") → Only if user wants details for a specific member
```

---

## Tier 3: Modernization Planning

### QF-10: "DDS to DDL migration assessment"

**Single query:**
```
ia_dds_to_ddl_status(limit=200)
```

**Only chain** `ia_where_used_detail` for specific files the user wants to prioritize.

---

### QF-11: "Repository health dashboard"

**Single query:**
```
ia_dashboard()
```

Returns: Object counts by category, line counts, library mapping — comprehensive overview in one call.

**Only add** these if user asks follow-up questions:
- `ia_code_complexity(member_name="*ALL", limit=10)` for hotspots
- `ia_unused_objects` for cleanup candidates
- `ia_exception_log` for parser issues

---

## Tier 4: Source Code Analysis

### QF-12: "Deep token-level analysis"

**For RPG:**
```
1. ia_rpg_source_tokens(member_name="ORDENTRY", limit=500)
```

**For CL:**
```
1. ia_cl_source_tokens(member_name="RUNJOB", limit=500)
```

**Skip** `ia_source_code` — it only returns location metadata, not the actual tokens.

---

### QF-13: "Find all uses of a specific variable/field name"

**Single query:**
```
ia_field_impact(field_name="ORDAMT", file_name="*ALL", limit=500)
```

**Only add** `ia_program_variables` for specific programs where you need to see all variables, not just the target field.

---

## Tier 5: Discovery & Inventory

### QF-14: "What objects exist matching a pattern?"

**Single query:**
```
ia_object_lookup(object_name="%ORD%", limit=100)
```

Supports `%` wildcards. Returns type, library, attribute for all matches.

---

### QF-15: "List all service programs and their callers"

**Two-query approach:**
```
1. ia_object_lookup(object_name="%SRV") → Find SRVPGMs
2. ia_where_used(object_name="<srvpgm>", object_type="*SRVPGM") → For each SRVPGM of interest
```

---

## Optimization Decision Tree

```
User Question
    │
    ├─► "What uses X?" ──────────► ia_where_used (single call)
    │                              └─► Chain ia_call_hierarchy ONLY if *SRVPGM in results
    │
    ├─► "Impact of changing X?" ─► ia_field_impact (single call)
    │                              └─► Chain ia_where_used ONLY for *SRVPGM amplifiers
    │
    ├─► "What does X call?" ─────► ia_call_hierarchy direction=called (single call)
    │
    ├─► "Tell me about program X" ► ia_program_detail section=*ALL (single call)
    │                               └─► Covers: calls, files, subroutines, vars, overrides
    │
    ├─► "Dead code (compiled)?" ─► ia_unused_objects (single call)
    │
    ├─► "Orphaned sources?" ─────► ia_uncompiled_sources (single call)
    │
    ├─► "Complexity hotspots?" ──► ia_code_complexity member_name=*ALL (single call)
    │
    ├─► "Find object named X" ───► ia_object_lookup with % wildcards (single call)
    │
    └─► "Repository overview" ───► ia_dashboard (single call)
```

---

## Anti-Patterns (Avoid These)

| Bad Pattern | Why It's Bad | Better Approach |
|-------------|--------------|-----------------|
| `ia_where_used` then `ia_reference_count` on same object | Redundant — just count the where_used results | Use only `ia_where_used` |
| `ia_source_code` before `ia_rpg_source_tokens` | source_code only returns location, not content | Go straight to `ia_rpg_source_tokens` |
| `ia_program_summary` + `ia_program_variables` + `ia_subroutines` | Three queries for one program | Use `ia_program_detail section=*ALL` |
| `ia_object_lifecycle` for every unused object | Unnecessary — unused_objects already confirms zero refs | Only check lifecycle for specific objects |
| Chaining to `ia_call_hierarchy` for every *PGM result | Overkill — most programs don't need deep analysis | Only chain for *SRVPGM or critical programs |

---

## Response Efficiency Rules

1. **Answer in one query when possible.** Most questions can be answered with a single well-chosen tool.

2. **Chain only when results demand it.** Seeing `*SRVPGM` in results = chain. Seeing only `*PGM` = usually stop.

3. **Use section filters.** `ia_program_detail section=FILES` is faster than `section=*ALL` if user only asks about files.

4. **Respect result counts.** If < 10 results, summarize and stop. If > 50, offer to narrow before deep-diving.

5. **Don't pre-chain.** Run the first query, interpret results, THEN decide if chaining adds value.
