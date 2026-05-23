# SMS z/OS ACS Routines

SMS Automatic Class Selection (ACS) routines for **z/OS 3.2 ADCD z32a**.

> Last updated: 2026-04-13 — Aligned to DB2 V13 only. DB2 V12 (DSNC130) removed. ZCX DATACLAS consolidated to single rule.

---

## Overview

This repository contains three ACS routines that work together to manage dataset storage allocation on z/OS:

```
┌─────────────────────────────────────────────────────────────┐
│                    Dataset Allocation Request                │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │   DATACLAS    │ ◄── Assigns Data Class
                  │   (Optional)  │     (DB2, ZCX only)
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │   STORCLAS    │ ◄── Assigns Storage Class
                  │   (Required)  │     (Evaluates FIRST)
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │   STORGRP     │ ◄── Assigns Storage Group
                  │   (Required)  │     (Uses STORCLAS value)
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │   Allocated   │
                  │    Dataset    │
                  └───────────────┘
```

**Critical Flow:** STORCLAS evaluates **before** STORGRP. STORGRP relies entirely on the `&STORCLAS` value set by STORCLAS—it does NOT re-evaluate HLQ patterns.

---

## Dataset Routing Summary

| Dataset Pattern | DATACLAS | STORCLAS | STORGRP |
|-----------------|----------|----------|---------|
| `DSNCD10.DSNDBC.**` | `DBDGDC` | `DBCLASSD` | `DBCLASSD` |
| `DSNCD10.DSNDBD.**` | `DBDGDC` | `DBCLASSD` | `DBCLASSD` |
| `DSND10.**` | *(none)* | `DBCLASSD` | `DBCLASSD` |
| `DSNC130.**` | `DBDGDC` | `DBCLASSD` | `DBCLASSD` |
| `ZCX.VS.**` | `CXDC` | `CXROOTSC` | `CXROOTSG` |
| `ZCX.FS.**` | `CXDC` | `CXROOTSC` | `CXROOTSG` |
| Temporary (`DSTYPE=TEMP`) | *(none)* | `SCWORK` | `TEMPVIO`, `SGWORK` |
| All others | *(none)* | `SCEXTEAV` | `SGEXTEAV` |

---

## DATACLAS

Assigns a Data Class based on dataset name qualifiers. Uses FILTLISTs for both DB2 and ZCX matching.

### Assignment Rules

| Data Class | Match Criteria | Purpose |
|------------|----------------|---------|
| `DBDGDC` | `HLQ` in DB2V13_PREFIX (`DSNCD10`, `DSNC130`, `DSND10`) | DB2 V13 datasets |
| `CXDC` | `DSN(1)=ZCX` AND `DSN(2)=VS` or `FS` | z/OS Container Extensions |
| *(none)* | All others | No Data Class assigned |

### Important Notes

- **All DB2 V13 datasets are SMS-managed** — FILTLIST DB2V13_PREFIX includes `DSNCD10`, `DSNC130`, and `DSND10`
- DATACLAS assigns `DBDGDC` to ALL datasets with HLQ in DB2V13_PREFIX (no qualifier filtering)
- Result: All three prefixes get DATACLAS=DBDGDC

---

## STORCLAS

Assigns Storage Class in evaluation order. **This routine evaluates BEFORE STORGRP.**

### Assignment Rules (Evaluation Order)

| Priority | Storage Class | Match Criteria | Action |
|----------|---------------|----------------|--------|
| 1 | `SCNOSMS` | Already set to `SCNOSMS` | Clears SMS management, exits immediately |
| 2 | `CXROOTSC` | `HLQ=ZCX` | All ZCX container datasets |
| 3 | `DBCLASSD` | `HLQ` in DB2V13_PREFIX (`DSNCD10`, `DSNC130`, `DSND10`) | All DB2 V13 datasets |
| 4 | `SCWORK` | `DSTYPE=TEMP` | Temporary datasets |
| 5 | `SCEXTEAV` | All others | Default extended format |

### Critical Behavior

- **Early EXIT CODE(0)** terminates evaluation immediately—order matters
- All datasets with HLQ in DB2V13_PREFIX get STORCLAS=DBCLASSD (no qualifier filtering)
- SCNOSMS clears `&STORCLAS` to empty string and exits (non-SMS path)

---

## STORGRP

Assigns Storage Group based **solely on the Storage Class** set by STORCLAS. Does NOT perform additional HLQ checks.

### Assignment Rules

| Storage Group | Mapped from STORCLAS | Purpose |
|---------------|----------------------|---------|
| `CXROOTSG` | `CXROOTSC` | Container storage for ZCX |
| `DBCLASSD` | `DBCLASSD` | DB2 V13 catalog/directory |
| `TEMPVIO`, `SGWORK` | `SCWORK` | Temporary dataset storage |
| `SGEXTEAV` | `SCEXTEAV` | Default extended format storage |
| *(none)* | `SCNOSMS` | Exits without setting group |

### Diagnostic Output

STORGRP writes diagnostic information to the system log before processing:
- Dataset name (`&DSN`)
- Storage Class (`&STORCLAS`)
- Data Class (`&DATACLAS`)
- Dataset Type (`&DSTYPE`)
- Storage Group assignment (after selection)

This output is invaluable for troubleshooting allocation issues.

---

## DB2 Dataset Routing Logic

All DB2 V13 datasets are SMS-managed:

```
DB2 V13 Datasets (all SMS-managed):
├── DSNCD10.**  ──► DATACLAS=DBDGDC, STORCLAS=DBCLASSD, STORGRP=DBCLASSD
├── DSNC130.**  ──► DATACLAS=DBDGDC, STORCLAS=DBCLASSD, STORGRP=DBCLASSD
└── DSND10.**   ──► DATACLAS=DBDGDC, STORCLAS=DBCLASSD, STORGRP=DBCLASSD
```

**Key Points:**
- FILTLIST DB2V13_PREFIX includes all three prefixes: `DSNCD10`, `DSNC130`, `DSND10`
- STORCLAS assigns `DBCLASSD` to ALL datasets with HLQ in DB2V13_PREFIX
- DATACLAS assigns `DBDGDC` to ALL datasets with HLQ in DB2V13_PREFIX
- STORGRP maps STORCLAS=DBCLASSD to STORGRP=DBCLASSD
- No special qualifier filtering—all three prefixes are treated identically

---

## ZCX Container Extensions

ZCX datasets require **both** conditions to be true:
1. HLQ = `ZCX`
2. Second qualifier in (`VS`, `FS`)

This prevents accidentally capturing datasets like `ZCX.BACKUP.**` or `ZCX.CONFIG.**`.

All ZCX datasets use a single consolidated DATACLAS (`CXDC`) as of 2026-04-13.

---

## File Structure

```
SMS/
├── README.md              # This file
├── AGENTS.md              # AI assistant guidance
├── zGIT-DS-Attributes     # PDS/PDSE attributes for Git (FB 80 32720)
└── CNTL/                  # ACS routines (not standard src/ structure)
    ├── DATACLAS           # Data Class assignment
    ├── STORCLAS           # Storage Class assignment
    └── STORGRP            # Storage Group assignment
```

**Note:** No build system—ACS routines are deployed directly to z/OS SMS.

---

## ACS Language Quick Reference

For those unfamiliar with ACS syntax:

```
PROC routine_name                    /* Procedure declaration */
  FILTLIST name INCLUDE('val1')      /* Define reusable pattern list */
  
  SELECT                             /* Begin selection logic */
    WHEN (&HLQ = 'PREFIX')           /* Condition (= for equality) */
      DO                             /* Begin action block */
        SET &STORCLAS = 'VALUE'      /* Assign variable */
        EXIT CODE(0)                 /* Exit immediately */
      END                            /* End action block */
  END                                /* End SELECT */
END                                  /* End PROC */
```

Key syntax notes:
- Variables prefixed with `&` (e.g., `&HLQ`, `&STORCLAS`)
- Array-like access: `&DSN(1)` = first qualifier, `&DSN(2)` = second
- `&&` for AND logic, `=` for equality
- `EXIT CODE(0)` terminates routine immediately (not a return statement)

---

## References

- [IBM z/OS DFSMS documentation](https://www.ibm.com/docs/en/zos)
- [ACS Routine Programming Guide](https://www.ibm.com/docs/en/zos/3.2.0?topic=sms-automatic-class-selection-acs-routines)
