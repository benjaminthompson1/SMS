# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Type
z/OS SMS (Storage Management Subsystem) ACS (Automatic Class Selection) routines written in ACS language for z/OS 3.2 ADCD z32a.

## Non-Obvious Patterns

### ACS Language Syntax
- ACS routines use PROC/END blocks, not standard programming constructs
- Variables prefixed with `&` (e.g., `&HLQ`, `&STORCLAS`, `&DSN(1)`)
- FILTLIST defines reusable pattern lists: `FILTLIST name INCLUDE('val1','val2')`
- Comparison uses `=` for equality, `&&` for AND logic
- Array-like access: `&DSN(1)` for first qualifier, `&DSN(2)` for second
- EXIT CODE(0) terminates routine immediately (not return)

### Critical Evaluation Order
- STORCLAS evaluates BEFORE STORGRP - STORGRP relies on STORCLAS being set
- STORGRP uses ONLY &STORCLAS for routing decisions, NOT HLQ checks
- Early EXIT CODE(0) prevents further rule evaluation (order matters)
- SCNOSMS in STORCLAS clears SMS management and exits immediately

### DB2 Dataset Routing
- All DB2 V13 datasets (`DSND10.**`, `DSNC130.**`, `DSNCD10.**`) are SMS-managed
- FILTLIST DB2V13_PREFIX includes all three prefixes
- DATACLAS uses two-qualifier matching for catalog/directory (`DSNCD10.DSNDBC/D.**`)
- STORCLAS assigns `DBCLASSD` to ALL HLQs in DB2V13_PREFIX (no qualifier filtering)
- Result: `DSND10.**` gets STORCLAS=DBCLASSD, STORGRP=DBCLASSD, but no DATACLAS

### ZCX Container Extensions
- ZCX datasets require HLQ=ZCX AND second qualifier in ('VS','FS')
- Both conditions must be true - not just HLQ matching
- Single DATACLAS (CXDC) for all ZCX types (consolidated from multiple rules)

### Diagnostic Output
- STORGRP writes diagnostic info (DSN, STORCLAS, DATACLAS, DSTYPE) before processing
- WRITE statements output to system log for troubleshooting
- Storage group assignment is logged after selection

### File Structure
- `zGIT-DS-Attributes` defines PDS/PDSE attributes for Git integration (FB 80 32720)
- CNTL directory contains the three ACS routines (not standard src/ structure)
- No build system - ACS routines deployed directly to z/OS SMS