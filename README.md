# Knowledge Base

Personal knowledge repository for smart contract security auditing.

---

## Structure

```
knowledge-base/
├── checklist/
│   └── checklist.md          # Audit checklist built from real missed findings
├── vulnerability-cards/
│   ├── Index.md              # Card index
│   ├── code-level/           # Code-level patterns (reentrancy, access control, signatures, etc.)
│   ├── defi-level/           # DeFi-specific patterns (ERC4626, liquidation, flash loans, etc.)
│   ├── Oracle/               # Oracle manipulation patterns
│   ├── real-cases/           # Real protocol case studies
│   └── templates/            # Card templates
└── audit-records/
    └── 盲审记录.md            # Blind audit records with findings and post-mortems
```

---

## Contents

### Checklist
An audit checklist built entirely from real missed or nearly-missed vulnerabilities across past audits — not copied from generic sources. Updated after every blind audit.

Covers: state variable errors, hook/callback risks, ERC20/ERC721 interactions, access control, reentrancy, oracle usage, lending/vault patterns, Uniswap V3 LP collateral, and more.

### Vulnerability Cards
Structured deep-dives into specific vulnerability patterns. Each card covers the mechanism, a concrete example, detection method, and fix.

- **code-level**: ABI encoding issues, access control, front-running, reentrancy, signatures, token quirks
- **defi-level**: ERC4626, flash loans, liquidation logic, share inflation, bad debt cascades
- **Oracle**: RedStone, TWAP manipulation

### Audit Records
Blind audit records for each protocol reviewed, including:
- What I found vs. what was reported
- Root cause analysis for missed findings (categorized by failure mode)
- Lessons converted into checklist items or vulnerability cards
