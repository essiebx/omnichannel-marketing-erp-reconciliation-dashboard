# Contributing

## Working conventions

- Keep generated raw data out of version control.
- Put identity normalization and hashing in staging SQL.
- Keep Python limited to generation and pipeline orchestration.
- Use the ERP as the source of truth; platform tables contain claims only.
- Document metric formulas in `docs/METRICS_DEFINITIONS.md` before implementation.
- Add a blocking test for every reconciliation rule.

## Development workflow

1. Create a focused branch.
2. Update the relevant specification and SQL/Python implementation together.
3. Run the target pipeline and validation report.
4. Review generated outputs for revenue tie-out.
