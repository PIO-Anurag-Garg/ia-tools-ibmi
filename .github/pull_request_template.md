## Summary
<!-- What does this PR do? -->

## Type
- [ ] New tool
- [ ] Bug fix
- [ ] Improvement to existing tool
- [ ] Documentation

## Checklist
- [ ] Tool name follows `ia_`/`ia-` convention
- [ ] SQL is read-only (`SELECT` only)
- [ ] Uses `${IA_LIBRARY}` (no hardcoded library)
- [ ] Uses `:param_name` bindings (no string concatenation)
- [ ] Includes `FETCH FIRST :limit ROWS ONLY`
- [ ] Description mentions queried iA table(s)
- [ ] Tested against an iA repository

## Test Results
<!-- Paste sample output or describe what you tested -->
