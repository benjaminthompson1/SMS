# SMS z/OS Parameters Documentation

This repository contains the Storage Management Subsystem (SMS) parameters for z/OS, defining data class, storage class, and storage group assignments.

## Table of Contents
- [Overview](#overview)
- [Components](#components)
  - [Data Class (DATACLAS)](#data-class-dataclas)
  - [Storage Class (STORCLAS)](#storage-class-storclas)
  - [Storage Group (STORGRP)](#storage-group-storgrp)

## Overview

These parameters define the SMS configuration for automated storage management in z/OS, handling dataset allocation and management based on predefined criteria.

## Components

### Data Class (DATACLAS)

The Data Class routine assigns specific data class attributes based on the high-level qualifier (HLQ) of datasets.

#### Assignments:
- `DBDGDC` assigned to:
  - DSND10.**
  - DSNC130.**
  - DSNCD10.**
  - DZ00SYS.**
- `CXDC` assigned to:
  - ZCX.**

### Storage Class (STORCLAS)

The Storage Class routine determines performance and availability requirements for datasets.

#### Key Features:
- Supports DB2 V13 datasets
- Special handling for temporary datasets
- ZCX (z/OS Container Extensions) support

#### Assignments:
- `SCNOSMS`: Bypasses SMS management
- `CXROOTSC`: Assigned to ZCX datasets
- `DBCLASSD`: Assigned to DB2 V13 datasets
- `SCWORK`: Assigned to temporary datasets
- `SCEXTEAV`: Default storage class for unmatched datasets

### Storage Group (STORGRP)

The Storage Group routine maps storage classes to physical storage pools.

#### Assignments:
- `CXROOTSG`: For ZCX datasets (mapped from CXROOTSC)
- `DBCLASSD`: For DB2 V13 datasets
- `SGWORK`: For temporary datasets (mapped from SCWORK)
- `SGEXTEAV`: For standard datasets (mapped from SCEXTEAV)

## Usage Notes

- The routines include diagnostic WRITE statements for troubleshooting
- Storage Group assignments are directly mapped from Storage Classes
- Special handling is implemented for DB2 V13 and ZCX environments
- Temporary datasets are automatically directed to work storage groups

## Implementation Details

All routines include proper error handling and exit codes. The Storage Group routine includes additional logging for troubleshooting, outputting:
- Dataset Name
- Storage Class
- Data Class
- Dataset Type

---
For more information about SMS configuration in z/OS, refer to the IBM documentation.
